# Swift 模块拆分模式：NetworkCore → NetworkCore + ServiceType

## 拆分动机

当通用网络库膨胀了业务代码（API 路由、签名逻辑、业务配置），需要拆分为：
- **NetworkCore** — 通用网络库，零业务依赖，任何项目可用
- **ServiceType** — 业务网络层，依赖 NetworkCore

## 模块边界

```
NetworkCore (通用) — 27 个文件
├── Core/        APIClient, Client, TargetType, Upload, Multipart
├── Plugin/      Auth, Logger, RequestID (无 SignPlugin)
├── Parser/      SmartDecoder, GraphQLDecoder, Connection
├── Environment/ NetworkEnvironment
├── Error/       HTTPError
├── Models/      JSONResponse
├── Utils/       Configuration, Logger
├── Rx/          NetworkClient+Rx
└── Service/     ServiceType, ServiceAPITarget, NetworkAuth, TokenRefreshable

ServiceType (业务) — 22 个文件
├── API/         Account, Device, Glasses, Feedback, Live, Meditation, Pay, OTA, VIP
├── Service/     BusinessTarget, CloudTarget, RequestSigner, ServiceType+BizAPI
├── VoiceChat/   API, Models, Targets, Configuration, RequestFactory, Error
├── Plugin/      SignPlugin (签名是业务特定的)
├── Legacy/      NetworkClient+BizLegacy
└── ServiceType.swift (@_exported import NetworkCore)
```

## 关键设计决策

### 1. @_exported import

ServiceType 入口文件：
```swift
@_exported import NetworkCore
```

消费者只需 `import ServiceType`，自动获得 NetworkCore 所有类型。

### 2. HTTPMethod 导出

Alamofire 的 `HTTPMethod` 类型不会通过 @_exported 自动传递。
需要在 NetworkCore 中显式导出：
```swift
// NetworkTargetType.swift
public typealias HTTPMethod = Alamofire.HTTPMethod
```

ServiceType 不再直接依赖 Alamofire（Package.swift 只依赖 NetworkCore）。

### 3. 服务标识分离

```swift
// ServiceType 中定义业务服务标识
extension ServiceType {
    static let bizAPI = ServiceType(rawValue: "bizAPI")      // 账户
    static let cloudAPI = ServiceType(rawValue: "cloudAPI")  // 云
    static let thirdParty = ServiceType(rawValue: "thirdParty")
    static let voiceChat = ServiceType(rawValue: "voiceChat")
}
```

### 4. Target 协议分层

```swift
// NetworkCore 中定义通用协议
public protocol ServiceAPITarget: NetworkTargetType {
    var service: ServiceType { get }
}

// ServiceType 中定义业务协议
public protocol BusinessTarget: ServiceAPITarget {
    var apiPath: String { get }
    var apiMethod: HTTPMethod { get }
    var apiParameters: [String: Any]? { get }
    // ...
}

// BusinessTarget 绑定 .bizAPI 服务
extension BusinessTarget {
    public var service: ServiceType { .bizAPI }
    // 自动签名
    public var headers: [String: String] {
        var headers = NetworkCoreConfiguration.shared.mergedHeaders(for: service)
        if needsSign, let parameters {
            headers["Sign"] = RequestSigner.sign(headers: headers, parameters: parameters)
        }
        return headers
    }
}

// CloudTarget 绑定 .cloudAPI 服务
public protocol CloudTarget: ServiceAPITarget {
    // ...
}
extension CloudTarget {
    public var service: ServiceType { .cloudAPI }
}
```

### 5. Sign 签名收敛

签名逻辑独立为 `RequestSigner` 枚举：
```swift
enum RequestSigner {
    static func sign(headers: [String: String], parameters: [String: Any]?) -> String {
        // headers + 大写参数 → 排序 → MD5
    }
}
```

### 6. VoiceChat 配置隔离

不用 extension 给 NetworkCoreConfiguration 添加业务属性，用独立配置类：
```swift
// ServiceType 中
public enum VoiceChatConfiguration {
    private static let lock = NSLock()
    private static var _clientContext: (any VoiceChatClientContext)?
    
    public static var clientContext: (any VoiceChatClientContext)? {
        lock.lock()
        defer { lock.unlock() }
        return _clientContext
    }
}
```

### 7. 兼容层

提供 `sendRequest()` 方法兼容旧回调签名：
```swift
extension NetworkClient {
    public func sendRequest<T: BusinessTarget>(
        _ target: T,
        completion: @escaping (Bool, [String: Any]?, String?) -> Void
    ) -> NetworkCancellable? where T.Response == JSONResponse {
        send(target) { result in
            switch result {
            case .success(let response):
                let dict = response.json
                let success = (dict["code"] as? Int) == 200
                completion(success, dict["data"] as? [String: Any], dict["msg"] as? String)
            case .failure(let error):
                completion(false, nil, error.localizedDescription)
            }
        }
    }
}
```

## Package.swift

```swift
// NetworkCore/Package.swift
let package = Package(
    name: "NetworkCore",
    platforms: [.iOS(.v16)],
    dependencies: [
        .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.12.0"),
        .package(url: "https://github.com/nicklama/SmartCodable.git", from: "6.0.0"),
        .package(url: "https://github.com/ReactiveX/RxSwift.git", from: "6.8.0"),
    ],
    // ...
)

// ServiceType/Package.swift
let package = Package(
    name: "ServiceType",
    platforms: [.iOS(.v16)],
    dependencies: [
        .package(path: "../NetworkCore"),
        // 不重复依赖 Alamofire！
    ],
    // ...
)
```

## 验证流程

1. 语法检查：`swiftc -parse -sdk $(xcrun --sdk iphonesimulator --show-sdk-path) -target arm64-apple-ios16.0-simulator`
2. Xcode 编译：⌘B
3. 运行测试：⌘R

**不要用 `swift build` 编译 iOS-only 包**（会报 macOS 平台不兼容）。
