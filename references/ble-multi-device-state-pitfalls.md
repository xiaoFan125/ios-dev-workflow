# BLE 多设备状态管理 Pitfalls

## 架构概览

BLE 伴侣 App（如 BrandC/SmartGlasses）的多设备状态管理层：

```
WRBleToolManager (BLE singleton)
  ├── connectionStatusBlock → 统一连接态回调
  ├── currentDevice(.Headset) → 当前耳机 live
  ├── currentDevice(.HeadsetCharging) → 当前仓 live
  └── connectionDevices → 已连接设备字典

WRDeviceHomeViewModel (首页 VM)
  ├── dataSource: [WRDeviceInfo]  → Realm 列表快照
  ├── currentModel → 当前选中设备别名 (=== dataSource[selectIndex])
  ├── connectionStatusBlock handler → 状态同步入口
  └── PresentationBus → UI 刷新总线
```

## Pitfall 1: stale currentModel 导致断连设备显示连接态

**场景**: A 已连接 → 搜索页添加 B 成功 → 返回首页 → A 概率显示连接态

**根因**: `applyCurrentBleConnectionState` 中的兜底逻辑保留了 stale runtime 引用。

```swift
// 多设备：未命中 live 的条目恢复为初始未连接快照
for (index, item) in items.enumerated() where !matchedIndices.contains(index) {
    if let runtime = currentModel.value,
       Self.isSameDevice(runtime, item),
       runtime.connectionStatus == .connected || runtime.connectionStatus == .connecting {
        preserveRuntimePresentationFields(from: runtime, onto: item)  // ← BUG
        continue
    }
    resetDeviceToDisconnectedSnapshot(item)  // 正常应该走这里
}
```

**时序**:
1. A 已连接，`currentModel.value` = A（connectionStatus = .connected）
2. 搜索页连接 B → BLE singleton 切换 liveHeadset 为 B
3. pop 回首页 → `handleDeviceAddedSuccess` → `applyCurrentBleConnectionState`
4. B 匹配到 live → `matchedIndices = {B}`
5. A 不在 matchedIndices → 进入兜底分支
6. `currentModel.value` 仍指向 A 且 `.connected` → 错误保留

**修复**: 兜底分支增加 live 设备一致性检查：

```swift
if let runtime = currentModel.value,
   Self.isSameDevice(runtime, item),
   runtime.connectionStatus == .connected || runtime.connectionStatus == .connecting {
    // 如果 BLE live 已指向另一设备，runtime 是过期引用，不保留
    if let live = liveHeadset,
       live.connectionStatus == .connected || live.connectionStatus == .connecting,
       !Self.isSameDevice(live, item) {
        resetDeviceToDisconnectedSnapshot(item)
    } else {
        preserveRuntimePresentationFields(from: runtime, onto: item)
    }
    mergeConnectedBoxStateIfNeeded(on: item)
    continue
}
```

## Pitfall 2: pair_connectionStatus 条件过严导致仓电量不刷新

**场景**: 搜索页连接 B 耳机成功 → 返回首页 → B 仓电量不显示

**根因**: `handleDeviceAddedSuccess` 中 `startBoxObserve()` 被 `pair_connectionStatus == .connected` 条件阻断。

```swift
// handleDeviceAddedSuccess line 774-780
if current.isBoxAddDevice {
    scheduleBoxAddFetchAfterSearchReturn(for: current)
} else {
    if current.connectionStatus == .connected {
        refreshBattery(for: current.deviceType)
    }
    if current.pair_connectionStatus == .connected {  // ← 条件过严
        startBoxObserve()  // 新设备 pair_connectionStatus 通常不是 .connected
    }
}
```

**修复**: 移除前置条件，`startBoxObserve` 内部已有完善保护：

```swift
} else {
    if current.connectionStatus == .connected {
        refreshBattery(for: current.deviceType)
    }
    // startBoxObserve 内部检查 isBoxLiveConnected + fetchedBoxInitialStateKeys
    startBoxObserve(for: current)
}
```

## Pitfall 3: isBoxAddDevice 的 MAC 实为仓 MAC 导致回调归属错误

**场景**: C 通过仓添加（`isBoxAddDevice = true`）→ 冷启动 → 点击 C 耳机连接 → 仓回调错误刷新到耳机

**根因**: `isBoxAddDevice` 设备的 `mac_adress` 是仓 MAC，不是耳机 MAC。

```swift
// WoerSearchViewModel.handleConnectionStatus 中仓添加映射
case .HeadsetCharging:
    deviceInfo.pair_device_adress = deviceInfo.mac_adress  // 仓 MAC
    deviceInfo.pairuuidString = deviceInfo.uuidString
    deviceInfo.deviceType = .Headset       // 强制映射为耳机类型
    deviceInfo.connectionStatus = .disconnected
    deviceInfo.isBoxAddDevice = true
```

当用户点击 C 的连接按钮 → `connect(C)` → `reconnectDevice(C)` → BLE 用 C 的 `mac_adress`（仓 MAC）重连 → 仓连接成功 → `connectionStatusBlock` 收到 `.HeadsetCharging` 回调。

`connectionStatusBlock` 未区分「仓主动连接」和「误将仓 MAC 当耳机 MAC 发起的连接」，直接进入 `syncAndNotifyDeviceUpdate`。

**修复方案 A**: `connectionStatusBlock` 中增加仓回调专用路径 + `connect()` 区分仓添加设备：

```swift
// connectionStatusBlock 中
if deviceInfo.deviceType == .HeadsetCharging,
   deviceInfo.connectionStatus == .connected {
    self.applyBoxConnectedCallback(deviceInfo)
} else {
    self.syncAndNotifyDeviceUpdate(incoming: deviceInfo)
}

/// 仓连接回调专用路径：仅写入 pair 态，禁止触发耳机 connectionStatus 变更。
private func applyBoxConnectedCallback(_ boxInfo: WRDeviceInfo) {
    guard let owner = resolveBoxOwnerInDataSource(for: boxInfo) else { return }
    owner.pair_connectionStatus = .connected
    let battery = boxInfo.pair_battery_value > 0 ? boxInfo.pair_battery_value : boxInfo.leftBatteryLevel
    if battery > 0 { owner.pair_battery_value = battery }
    if owner.isBoxAddDevice {
        owner.connectionStatus = .disconnected  // 仓添加设备：耳机保持 disconnected
    }
    clearMisassignedPairState(for: owner)
    publishPresentationPulse(for: owner)
    startBoxObserve(for: owner)
    refreshBoxSettingsIfNeeded(for: owner, reason: .initialConnect)
}

// connect() 中区分仓添加设备
if listItem.isBoxAddDevice {
    // 仓已连时跳过冗余 reconnect（mac_adress 实为仓 MAC）
    if listItem.pair_connectionStatus == .connected || isBoxLiveConnected(for: listItem) {
        listItem.pair_connectionStatus = .connected
        notifyCurrentModelPropertiesChanged()
        startBoxObserve(for: listItem)
        refreshBoxSettingsIfNeeded(for: listItem, reason: .initialConnect)
    } else {
        WRBleToolManager.shared.reconnectDevice(deviceInfo: listItem)
    }
} else if listItem.connectionStatus != .connected && listItem.pair_connectionStatus != .connected {
    WRBleToolManager.shared.reconnectDevice(deviceInfo: listItem)
}
```

## Pitfall 4: debounced 订阅竞态导致顶栏 icon 错位

**场景**: A 连接 → 搜索页添加 B 成功 → 返回首页 → 顶栏第一个设备显示选中态但 icon 是 A 的

**根因**: `viewWillAppear` 的 `refreshOnAppear` 和搜索页 `addSuccess` 闭包（0.2s 延迟）存在时序竞争。

```
T=0ms    viewWillDisappear (搜索页) → isInScanPage = false
T≈50ms   viewWillAppear (首页) → refreshOnAppear → dataSource.accept(旧顺序)
         → 200ms debounce 开始
T≈200ms  addSuccess 闭包 → handleDeviceAddedSuccess → dataSource.accept(新顺序)
         → 200ms debounce 重置
```

中间窗口（T=50ms ~ T=250ms）：
- dataSource debounced 订阅可能用旧顺序触发 `reloadItems`
- `currentModel` 订阅（100ms debounce）触发 `updateSelection` → `reloadItems` 配置旧数据
- 最终 `reloadData()`（新顺序）的 layout pass 与之前的 `reloadItems` 冲突
- 第一个 cell 的 icon 停留在 A 的图片

**关键代码路径**:
```swift
// WRDeviceHomeCtr dataSource 订阅（200ms debounce）
viewModel.dataSource
    .distinctUntilChanged { ... }
    .debounce(.milliseconds(200), scheduler: MainScheduler.instance)
    .subscribe(onNext: { items in
        self.baseCollectionContainer.reloadData()
        self.lCollectionsView.setItems(items, current: selectedModel)  // ← 可能被竞态覆盖
    })

// WRDeviceHomeCtr currentModel 订阅（100ms debounce）
viewModel.currentModel
    .debounce(.milliseconds(100), scheduler: MainScheduler.instance)
    .subscribe(onNext: { current in
        self.lCollectionsView.updateSelection(at: index)  // ← 用旧 items 配置 cell
    })
```

**修复**: 新增不经 debounce 的直接刷新通道：

```swift
// VM 中
let deviceListDirectRefresh = PublishRelay<Void>()

// handleDeviceAddedSuccess 末尾
deviceListDirectRefresh.accept(())

// Controller 中订阅
viewModel.deviceListDirectRefresh
    .observe(on: MainScheduler.instance)
    .subscribe(onNext: { [weak self] in
        guard let self else { return }
        let items = self.viewModel.dataSource.value
        let sIndex = self.viewModel.selectIndex
        let selectedModel = (sIndex < items.count) ? items[sIndex] : items.first
        self.lCollectionsView.setItems(items, current: selectedModel)
    })
    .disposed(by: disposeBag)
```

**通用模式**: 当 debounced 订阅和非 debounced 事件存在时序竞争时，为关键路径增加不经 debounce 的直接刷新通道（`PublishRelay<Void>`），确保最终状态一致。

## 通用规则

### 多设备状态刷新三原则

1. **BLE live 优先于 runtime 引用** — 刷新列表时以 `WRBleToolManager.shared.currentDevice()` 为准，`currentModel.value` 可能过期
2. **仓回调只写 pair 态** — `HeadsetCharging` connected 回调不应修改耳机主 `connectionStatus`
3. **startBoxObserve 自行保护** — 内部有 `isBoxLiveConnected` + `fetchedBoxInitialStateKeys` + `boxInitialStateInFlightKeys` 三重保护，调用方无需前置条件

### Debounce 竞态防护原则

4. **关键 UI 路径不经 debounce** — 当两个事件源（一个 debounced、一个非 debounced）修改同一 UI 时，关键路径应有直接刷新通道绕过 debounce
5. **pop 动画期间的事件时序不确定** — `viewWillAppear` 和异步回调的先后顺序取决于动画时长和 runloop 调度，不要假设固定顺序

### 关键方法速查

| 方法 | 职责 |
|------|------|
| `applyCurrentBleConnectionState` | 将 BLE live 态合并到 Realm 列表快照 |
| `syncAndNotifyDeviceUpdate` | connectionStatusBlock 主路径：resolve → merge → commit |
| `resolveBoxOwnerInDataSource` | 仓归属哪条耳机列表项（pair MAC 优先） |
| `startBoxObserve` | 仓初始态 fetch（电量/版本/设置） |
| `isBoxForHeadset` | 仓是否属于指定耳机（多维 MAC/UUID 匹配） |
| `handleDeviceAddedSuccess` | 搜索页添加成功后首页刷新入口 |
| `clearMisassignedPairState` | 清理其他设备的错误 pair 态 |
| `applyBoxConnectedCallback` | 仓回调专用路径：仅写 pair 态 + 仓初始态 |
| `deviceListDirectRefresh` | 绕过 debounce 的顶栏直接刷新通道 |
