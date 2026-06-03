# AFNetworking → Alamofire 网络层迁移指南

## 架构设计

```
旧栈: AFNetworking ← DomainNetworkLayer(ObjC) ← ServiceLayer(ObjC) ← DomainNetwork(Swift) ← 业务层
新栈: Alamofire ← NetworkCore(通用) ← ServiceType(业务) ← 业务层
```

## 模块拆分

### NetworkCore（通用网络库，零业务依赖）
```
Core/       NetworkClient, NetworkClient, TargetType, Upload, Multipart
Plugin/     Auth, Logger, RequestID（无 SignPlugin）
Parser/     SmartDecoder, GraphQLDecoder, Connection
Models/     JSONResponse
Service/    ServiceType, ServiceAPITarget, NetworkAuth, TokenRefreshable
Utils/      NetworkCoreConfiguration, NetworkCoreLogger
```

### ServiceType（业务网络层，依赖 NetworkCore）
```
API/        AccountAPI, DeviceAPI, GlassesAPI, FeedbackAPI, LiveAPI, etc.
Service/    SignedTarget, BusinessTarget, CloudTarget, ThirdPartyTarget
            ServiceType+BizAPI, RequestSigner
VoiceChat/  API, Models, Targets, Configuration, RequestFactory
Legacy/     NetworkClient+BizLegacy（兼容旧回调）
```

**入口文件**: `ServiceType.swift` 仅包含 `@_exported import NetworkCore`

## 多服务器配置

```swift
extension ServiceType {
    public static let bizAPI = ServiceType(rawValue: "bizAPI")        // 账户 (identity)
    public static let cloudAPI = ServiceType(rawValue: "cloudAPI")    // 云服务 (api)
    public static let thirdParty = ServiceType(rawValue: "thirdParty") // 第三方 (tparty)
    public static let voiceChat = ServiceType(rawValue: "voiceChat")  // 语音
}
```

## Target 协议设计（SignedTarget 基类模式）

**⚠️ 关键设计**: 所有需要 Sign 签名的 API Target 继承自 `SignedTarget` 基类协议：

```swift
// SignedTarget.swift — 基类，包含签名逻辑
public protocol SignedTarget: ServiceAPITarget {
    var apiPath: String { get }
    var apiMethod: HTTPMethod { get }
    var apiParameters: [String: Any]? { get }
    var apiEncoding: NKParameterEncoding { get }
    var apiTimeout: TimeInterval { get }
    var needsSign: Bool { get }  // 默认 true
}

extension SignedTarget {
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

### BusinessTarget（账户相关，继承 SignedTarget）
```swift
public protocol BusinessTarget: SignedTarget { ... }
extension BusinessTarget {
    public var service: ServiceType { .bizAPI }
    public var apiEncoding: NKParameterEncoding { .url }  // ⚠️ 默认 form-encoded
}
```

### CloudTarget（设备/OTA/发现/直播，也继承 SignedTarget！）
```swift
// ⚠️ CloudTarget 也必须继承 SignedTarget！旧代码中所有 API 都使用 Sign 签名
public protocol CloudTarget: SignedTarget { ... }
extension CloudTarget {
    public var service: ServiceType { .cloudAPI }
    public var apiEncoding: NKParameterEncoding { .json }
}
```

### ThirdPartyTarget（chat/upload/translate，也继承 SignedTarget）
```swift
public protocol ThirdPartyTarget: SignedTarget { ... }
extension ThirdPartyTarget {
    public var service: ServiceType { .thirdParty }
    public var apiEncoding: NKParameterEncoding { .json }
}
```

## Sign 签名（关键 Pitfall）

**⚠️ 旧代码签名格式**: `key1=value1&key2=value2&...&Token=xxx`（key=value 对，& 分隔，Token 放最后）

**✗ 错误格式**（直接拼接）: `key1value1key2value2...`
**✓ 正确格式**: `CountryCode=+86&Customer=BrandC&Language=zh&Timestamp=xxx&Username=xxx&Token=xxx`

```swift
enum RequestSigner {
    static func sign(headers: [String: String], parameters: [String: Any]?) -> String {
        // 1. 参数首字母大写（username → Username）
        // 2. 与 headers 合并
        // 3. 按 key 排序
        // 4. 拼接为 key1=value1&key2=value2&...
        // 5. Token 放最后
        // 6. 整个字符串 MD5
    }
}
```

## 兼容层

```swift
extension NetworkClient {
    // ⚠️ 泛型约束必须是 ServiceAPITarget（不是 BusinessTarget），才能支持所有 Target 类型
    public func sendRequest<T: ServiceAPITarget>(
        _ target: T,
        completion: @escaping (Bool, [String: Any]?, String?) -> Void
    ) -> NetworkCancellable? where T.Response == JSONResponse {
        send(target) { result in
            switch result {
            case .success(let response):
                let dict = response.json
                let success = (dict["code"] as? Int) == 200 || (dict["code"] as? Int) == 0
                completion(success, dict["data"] as? [String: Any], dict["msg"] as? String)
            case .failure(let error):
                completion(false, nil, error.localizedDescription)
            }
        }
    }
}
```

## 迁移步骤

1. 创建 NetworkCore 模块（通用网络库）
2. 创建 ServiceType 模块（业务网络层）
3. 定义 SignedTarget 基类协议
4. 定义 API Target 枚举（AccountAPI, DeviceAPI, etc.）
5. 实现 RequestSigner 签名
6. 实现 sendRequest 兼容层
7. 配置多服务器地址（identity/api/tparty）
8. 注入 Language（NetworkCoreConfiguration.shared.language）
9. 逐文件迁移业务层（import + API 调用）
10. 迁移 UserManager/User 到主项目
11. 清理旧模块

## 常见 Pitfalls

- **UserManager/User 在旧模块**: 移动到主项目 AppGroup/，否则 NSKeyedUnarchiver 类名不匹配
- **Int vs String**: userId 是 Int，API 参数期望 String，需 `"\(userId)"`
- **服务器地址分离**: 账户(identity)、云(api)、第三方(tparty) 需要不同 host
- **BusinessTarget headers**: 必须包含 Timestamp/Token/Customer/Language
- **CloudTarget 也需要签名**: 继承 SignedTarget，否则服务器返回 20103
- **ThirdPartyTarget**: chat/upload/translate 使用 tparty 服务器
- **sendRequest 泛型约束**: 必须用 `ServiceAPITarget` 而非 `BusinessTarget`
- **POST 编码默认 form-encoded**: BusinessTarget 默认 `.url`，不是 `.json`
- **Sign 格式**: 必须是 `key=value&` 格式，不能省略 `=` 和 `&`
- **Language 格式**: 用 `LanguageManager.default.mapping()`（返回 zh-CN），不是 `Locale`（返回 zh）
- **Customer 不能硬编码**: 用 `Bundle.displayName`，不同构建配置不同
- **无参请求也需要 Sign**: heartbeat、logOut 等仍需对 headers-only 计算签名
- **Crash reporting**: 独立于主网络层，用 URLSession 直接实现
- **批量迁移超时**: 用 sed 做 import 替换，分批委托处理 API 调用迁移
- **API path 必须逐个对照旧代码**: 不能只看 API 名称，必须对比 referenceURL
- **AppDelegate 不覆盖 plugins**: 用 builder 的 `add(plugin:)`，不要覆盖已有 plugins
