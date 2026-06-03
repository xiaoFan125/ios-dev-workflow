# iOS 系统音量同步到 BLE 设备

## 问题

iOS 没有公开 API 直接监听用户按音量键的事件。`MPVolumeView` 只能显示和控制音量滑块，不能监听物理按键。

## 解决方案

监听私有系统通知 `AVSystemController_SystemVolumeDidChangeNotification`：

```swift
// 在 bindPublishers() 或类似初始化位置
NotificationCenter.default.publisher(for: NSNotification.Name("AVSystemController_SystemVolumeDidChangeNotification"))
    .sink { [weak self] notification in
        self?.handleSystemVolumeChange(notification)
    }
    .store(in: cancelBag)
```

## 获取当前音量

```swift
private func handleSystemVolumeChange(_ notification: Notification) {
    // AVAudioSession.sharedInstance().outputVolume 返回 0.0 - 1.0
    let volume = AVAudioSession.sharedInstance().outputVolume
    let volumeInt = Int(round(volume * 100))  // 转换为 0-100

    // 同步到 BLE 设备
    XRDeviceInfoService.syncVolume(volumeInt) { success in
        // 处理结果
    }
}
```

## 注意事项

1. **私有通知风险** — `AVSystemController_SystemVolumeDidChangeNotification` 是私有 API，App Store 审核可能被拒。如果需要上架，考虑使用 `MPVolumeView` 的 KVO 或 `AVAudioSession` 的 `outputVolume` 属性观察
2. **蓝牙 A2DP 场景** — 当音频通过蓝牙 A2DP 输出时，音量控制由蓝牙设备处理，系统音量可能不会变化
3. **V1/V2 设备区分** — V1 设备使用 `XRGlassesInfoService.syncVolume`，V2 设备使用 `XRDeviceInfoService.syncVolume`
4. **guard isDesktopModeModule** — 只在 AI Desktop 模块激活时同步音量，避免在非语音场景下发送无用指令

## 替代方案（App Store 安全）

如果需要上架 App Store，使用 `MPVolumeView` + KVO：

```swift
let volumeView = MPVolumeView()
// 找到 UISlider 子类并添加 KVO
if let slider = volumeView.subviews.first(where: { $0 is UISlider }) as? UISlider {
    slider.addObserver(self, forKeyPath: "value", options: .new, context: nil)
}
```

或者使用 `AVAudioSession` 的 `outputVolume` 属性观察（iOS 15+）：

```swift
// 需要先激活 AVAudioSession
try AVAudioSession.sharedInstance().setActive(true)
// 然后通过 KVO 或 Combine 观察 outputVolume
```
