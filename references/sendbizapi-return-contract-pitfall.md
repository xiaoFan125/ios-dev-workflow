# sendRequest 返回契约 Pitfall（根因级）

## 问题

旧栈 `ServiceLayerAPI+HandleResult.m` 成功时返回**完整 JSON body**：

```objc
if (completionBlock) {
    completionBlock(YES, nil, responseObject);  // responseObject 是完整 JSON
}
```

App 层按完整响应解析：
```swift
// UserManager.login
let user = User.decodeFromDictionary(info)  // info 是完整 JSON
// info["data"]["user"]

// checkError
let erroCode = dic["code"] as? Int  // dic 是完整 JSON

// HomeService
let code = list?["code"] as? Int  // list 是完整 JSON
let data = list?["data"] as? [String: Any]
```

## 错误实现

```swift
// ✗ 只返回 data 字段
case .success(let response):
    let dict = response.json
    let success = (dict["code"] as? Int) == 200
    let data = dict["data"] as? [String: Any]
    completion(success, data, message)  // App 收到的是内层 data
```

## 影响范围

| 调用点 | 期望 | 实际收到 | 后果 |
|--------|------|----------|------|
| UserManager.login | info["data"]["user"] | 已是内层 data | 登录无法解析用户 |
| HomeService.fetchDevice | list["code"], list["data"]["terminals"] | 仅内层 data | 设备列表失败 |
| ExploreViewModel | dictionary["data"] | 已是 data | 双重 unwrap，列表空 |
| checkError | dic["code"] | 通常无 code | 业务错误被跳过 |
| heartbeat | dictionary["code"] 互踢 | 无 code | 互踢检测失效 |

## 正确实现

```swift
// ✓ 返回完整 JSON body（与旧栈一致）
case .success(let response):
    let dict = response.json
    let message = dict["msg"] as? String
    completion(true, dict, message)  // App 收到完整 JSON
```

**关键**: success 判断交给 App 层（旧栈语义：HTTP 成功即 true）。

## 修复效果

一次修复：登录、设备列表、发现页、OTA、版本检查、错误处理、互踢检测等大量调用点。
