# Swift 模块拆分模式：通用库 + 业务层

## 适用场景

当一个 SPM 模块同时包含通用基础设施和业务代码时，拆分为两个模块：
- **通用库** — 零业务依赖，任何项目可用
- **业务层** — 依赖通用库，项目专用

## 架构模式

```
通用库 (NetworkCore)          业务层 (ServiceType)
├── Core/                    ├── API/           (业务路由)
├── Plugin/                  ├── Service/       (业务协议)
├── Parser/                  ├── VoiceChat/     (业务子模块)
├── Error/                   ├── Plugin/        (业务插件)
├── Models/                  ├── Legacy/        (兼容层)
└── Utils/                   └── 入口.swift      (@_exported import)
```

## 关键设计决策

### 1. `@_exported import` 模式

业务层入口文件使用 `@_exported import`，消费者只需 `import 业务层`：

```swift
// ServiceType.swift
@_exported import NetworkCore
```

**好处**: 业务层消费者不需要同时 import 两个模块。
**注意**: `@_exported` 会污染命名空间，仅用于紧密耦合的分层模块。

### 2. HTTPMethod 导出

通用库通过 typealias 导出 Alamofire 类型，业务层不需要直接依赖 Alamofire：

```swift
// NetworkCore/Core/NetworkTargetType.swift
public typealias HTTPMethod = Alamofire.HTTPMethod
```

**好处**: 业务层的 Package.swift 只依赖通用库，不重复依赖第三方库。

### 3. 签名逻辑收敛

签名计算放在业务层的独立文件，不在通用库的 Plugin 中：

```swift
// ServiceType/Service/RequestSigner.swift
enum RequestSigner {
    static func sign(headers: [String: String], parameters: [String: Any]?) -> String {
        // headers + 大写参数 → 排序 → MD5
    }
}
```

**好处**: 签名是业务特定逻辑，不应污染通用库。

### 4. 配置隔离

业务特定的配置（如 VoiceChatClientContext）放在业务层，不污染通用库的 Configuration：

```swift
// ServiceType/VoiceChat/VoiceChatConfiguration.swift
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

**好处**: 通用库的 Configuration 保持纯净，业务配置由业务层管理。

### 5. 共享实例

通用库提供带默认插件的共享实例：

```swift
// NetworkCore.swift
public enum NetworkCore {
    public static let shared: NetworkClient = {
        NetworkClient.build()
            .add(plugin: LoggerInterceptor())
            .add(plugin: RequestIDInterceptor())
            .build()
    }()
}
```

## Package.swift 示例

```swift
// 通用库
let package = Package(
    name: "NetworkCore",
    dependencies: [
        .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.12.0"),
    ]
)

// 业务层 — 只依赖通用库，不重复依赖 Alamofire
let package = Package(
    name: "ServiceType",
    dependencies: [
        .package(path: "../NetworkCore"),
    ]
)
```

## 迁移检查清单

- [ ] 通用库中无业务特定的类型（API 路由、服务标识）
- [ ] 通用库中无业务特定的协议扩展（如 configureBizAPI）
- [ ] 业务层通过 `@_exported import` 重导出通用库
- [ ] 业务层 Package.swift 只依赖通用库
- [ ] 业务层通过 typealias 使用第三方库类型（如 HTTPMethod）
- [ ] 签名/加密等业务逻辑在业务层，不在通用库
- [ ] 配置按职责分层，不互相污染
