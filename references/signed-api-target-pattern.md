# SignedTarget 基类模式

## 问题

多个 API Target 类型（Biz/Cloud/ThirdParty）都需要相同的 Sign 签名逻辑和 headers（Timestamp/Token/Customer/Language）。如果每个 Target 独立实现，会导致：
1. 代码重复
2. 签名逻辑不一致
3. headers 字段遗漏

## 解决方案

提取 `SignedTarget` 基类协议，所有需要签名的 Target 继承它：

```
SignedTarget (基类，包含签名逻辑)
├── BusinessTarget (.bizAPI)      — 账户相关
├── CloudTarget (.cloudAPI)  — 设备/OTA/发现/直播
└── ThirdPartyTarget (.thirdParty) — chat/upload/translate
```

## 实现要点

### 1. SignedTarget 基类

```swift
public protocol SignedTarget: ServiceAPITarget {
    var apiPath: String { get }
    var apiMethod: HTTPMethod { get }
    var apiParameters: [String: Any]? { get }
    var apiEncoding: NKParameterEncoding { get }
    var apiTimeout: TimeInterval { get }
    var needsSign: Bool { get }  // 默认 true
}

extension SignedTarget {
    // 默认实现：path/method/parameters/encoding/timeout
    public var path: String { apiPath }
    public var method: HTTPMethod { apiMethod }
    public var parameters: [String: Any]? { apiParameters }
    public var encoding: NKParameterEncoding { apiEncoding }
    public var timeoutInterval: TimeInterval { apiTimeout }
    public var needsSign: Bool { true }

    // 签名 headers
    public var headers: [String: String] {
        let token = NetworkCoreConfiguration.shared.mergedHeaders(for: service)["Authorization"]?
            .replacingOccurrences(of: "Bearer ", with: "") ?? ""
        let customer = Bundle.main.object(forInfoDictionaryKey: "CFBundleDisplayName") as? String ?? "BrandC"
        let language = NetworkCoreConfiguration.shared.language ?? Self.currentLanguage()
        let timestamp = "\(Int(Date().timeIntervalSince1970))"
        var headers = ["Timestamp": timestamp, "Token": token, "Customer": customer, "Language": language]
        if needsSign {
            headers["Sign"] = RequestSigner.sign(headers: headers, parameters: parameters)
        }
        return headers
    }
}
```

### 2. 子协议

```swift
public protocol BusinessTarget: SignedTarget { ... }
extension BusinessTarget { public var service: ServiceType { .bizAPI } }

public protocol CloudTarget: SignedTarget { ... }
extension CloudTarget { public var service: ServiceType { .cloudAPI } }

public protocol ThirdPartyTarget: SignedTarget { ... }
extension ThirdPartyTarget { public var service: ServiceType { .thirdParty } }
```

### 3. App 层注入语言

```swift
// NetworkCoreBootstrap.configure() 中
NetworkCoreConfiguration.shared.language = LanguageManager.default.mapping()  // "zh-CN", "en"
```

## 关键 Pitfalls

1. **CloudTarget 也必须继承 SignedTarget** — 旧代码中所有 API 都使用签名
2. **Language 不能用 Locale.current** — 旧代码返回 "zh-CN"，Locale 返回 "zh"
3. **Customer 不能硬编码** — 应使用 Bundle.displayName
4. **无参请求也需要 Sign** — needsSign 默认 true，对 headers-only 计算签名
5. **POST 编码默认 .url** — 旧代码使用 form-encoded，不是 JSON
6. **ServiceType 不能依赖 App 类型** — 通过 NetworkCoreConfiguration 注入
