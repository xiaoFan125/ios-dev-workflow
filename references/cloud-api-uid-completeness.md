# Cloud API uid 参数完整性清单

## 问题

旧栈几乎所有 cloud 请求带 uid 参数，迁移时容易遗漏。缺少 uid 会导致：
1. 服务器返回参数缺失
2. Sign 签名与服务端不一致（参数参与签名计算）

## 修复模式

### 1. NetworkCoreConfiguration 注入

```swift
// NetworkCoreConfiguration.swift
public var currentUid: String?
public var currentUsername: String?

// NetworkCoreBootstrap.configure()
NetworkCoreConfiguration.shared.currentUid = "\(UserManager.shared.user?.userId ?? 0)"
NetworkCoreConfiguration.shared.currentUsername = UserManager.shared.user?.username ?? ""
```

### 2. Target 参数统一添加

```swift
public var apiParameters: [String: Any]? {
    let uid = NetworkCoreConfiguration.shared.currentUid ?? "0"
    switch api {
    case .getDevicesList:
        return ["uid": uid]
    case .discoveryList(let start, let length, let favorite):
        return ["uid": uid, "start": start, "length": length, "favorite": favorite]
    // ...
    }
}
```

## 需要 uid 的 API 清单

| API | 旧参数 | 需补充 |
|-----|--------|--------|
| DeviceAPI.getDevicesList | uid | ✓ |
| DeviceAPI.getDeviceIdentifier | uid, mac | ✓ |
| DeviceAPI.deviceBind | uid + deviceInfo | ✓ |
| DeviceAPI.deviceUnbind | uid, tid | ✓ |
| DeviceAPI.getDeviceDetail | uid, tid | ✓ |
| DeviceAPI.deviceRename | uid, tid, name | ✓ |
| GlassesAPI.discoveryList | uid, tag, start, length, favorite | ✓ |
| GlassesAPI.discoveryDetail | uid, id | ✓ |
| GlassesAPI.discoveryFavorite | uid, discoverId | ✓ |
| GlassesAPI.mapSaveAddress | uid, type, key, value | ✓ |
| LiveAPI.getRoomList | uid, username | ✓ |
| LiveAPI.createLiveRoom | uid, username, title, cover | ✓ |
| OTAAPI.getLatestFirmwareVersion | uid, serial | ✓ |
| OTAAPI.getAppNewestVersion | uid, type, version | ✓ |
| VIPTarget.* | uid | ✓ |
| FeedbackTarget.* | uid | ✓ |
| MeditationTarget.* | uid | ✓ |
| AccountAPI.heartbeat | uid, username | ✓ |
