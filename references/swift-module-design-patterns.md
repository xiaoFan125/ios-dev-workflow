# Swift Module Design Patterns

## 模块拆分原则

将大型 SPM 模块拆分为通用库 + 业务层：

```
NetworkCore (通用)          ServiceType (业务)
├── Core/                  ├── API/          (Account, Device, Glasses...)
├── Plugin/                ├── Service/      (BusinessTarget, Sign)
├── Parser/                ├── VoiceChat/
├── Error/                 └── ServiceType.swift
├── Models/                    (@_exported import NetworkCore)
├── Utils/
└── NetworkCore.swift
```

**边界规则**:
- 通用库：零业务概念，任何项目可直接使用
- 业务层：通过 `@_exported import` 导出通用库，消费者只需 `import 业务层`

## `@_exported import` 模式

```swift
// ServiceType.swift — 入口文件
@_exported import NetworkCore
```

效果：`import ServiceType` 的文件自动获得 NetworkCore 所有 public 类型，无需重复 import。

## 隐藏第三方依赖

```swift
// 在通用库中导出类型别名
public typealias HTTPMethod = Alamofire.HTTPMethod
```

业务层使用 `HTTPMethod` 而非直接 `import Alamofire`，通用库的依赖对外不可见。

## SmartCodable 6.x Conformance

```swift
// ❌ SmartCodableX = SmartDecodable & SmartEncodable
//    要求 Decodable + Encodable + init()
public typealias SmartDecodable = SmartCodableX

// ✅ SmartCodable.SmartDecodable 仅要求 Decodable + init()
public typealias SmartDecodable = SmartCodable.SmartDecodable
```

**关键**: `SmartCodableX` 需要 `Encodable`，对纯解码类型（如 `[String: Any]` 包装）无法满足。使用 `SmartCodable.SmartDecodable` 更安全。

**JSONResponse 模式**:
```swift
public final class JSONResponse: SmartCodable.SmartDecodable {
    public let json: [String: Any]

    public required init() { self.json = [:] }  // SmartDecodable 要求

    public required init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let dict = try? container.decode([String: AnyCodable].self) {
            self.json = dict.mapValues { $0.value }
        } else {
            self.json = [:]
        }
    }
}
```

## 线程安全模式

```swift
// NSLock 保护可变状态
public final class Configuration: @unchecked Sendable {
    private let lock = NSLock()
    private var _data: [String: Any] = [:]

    public var data: [String: Any] {
        get { lock.lock(); defer { lock.unlock() }; return _data }
        set { lock.lock(); defer { lock.unlock() }; _data = newValue }
    }
}

// @MainActor + NSLock 分离
@MainActor
public func configureAuth(tokenRefresher: TokenRefreshable) {
    let plugin = AuthInterceptor(tokenRefresher: tokenRefresher)
    authLock.lock()           // 专用锁保护 authPlugin
    self.authPlugin = plugin
    authLock.unlock()
    withLock { plugins.append(plugin) }  // 主锁保护 plugins
}
```

**规则**: `@MainActor` 方法内不要持有 `NSLock` 时等待 MainActor，会导致死锁。分离锁的职责。

## 插件系统设计

```swift
public protocol NetworkPlugin {
    func willSend(_ request: URLRequest)
    func didReceive(_ response: HTTPURLResponse?, data: Data?, error: Error?)
}
```

**陷阱**:
- `willSend` 不要依赖 Alamofire 内部 API（如 `onURLRequestCreation`），手动构造 `URLRequest` 更稳定
- Token 刷新重试时不要重复触发插件，传 `plugins: []` 跳过
- 敏感字段（password, token）在日志插件中脱敏

## 业务 API Target 模式

```swift
// 协议继承链
NetworkTargetType → ServiceAPITarget → BusinessTarget
  (associatedtype Response)   (service)      (apiPath, apiMethod, needsSign)

// 业务 Target
struct AccountTarget: BusinessTarget {
    typealias Response = JSONResponse
    let api: AccountAPI

    var apiPath: String { /* switch api */ }
    var apiMethod: HTTPMethod { /* switch api */ }
    var apiParameters: [String: Any]? { /* switch api */ }

    // 自动签名
    var headers: [String: String] {
        var headers = NetworkCoreConfiguration.shared.mergedHeaders(for: service)
        if needsSign, let parameters {
            headers["Sign"] = RequestSigner.sign(headers: headers, parameters: parameters)
        }
        return headers
    }
}

// 兼容旧代码的回调签名
extension NetworkClient {
    func sendRequest(_ target: some BusinessTarget,
                    completion: @escaping (Bool, [String: Any]?, String?) -> Void)
    -> NetworkCancellable? where T.Response == JSONResponse { ... }
}
```
