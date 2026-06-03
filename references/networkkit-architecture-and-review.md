# NetworkCore 架构、迁移模式与代码审查

## 一、架构总览

```
Sources/NetworkCore/
├── Core/           网络请求调度 (NetworkClient → NetworkClient → Alamofire)
├── Service/        业务 API Target (Account/Device/Glasses/VoiceChat)
├── Plugin/         插件系统 (Auth/Logger/RequestID/Sign)
├── Parser/         解码层 (SmartDecoder/GraphQLDecoder)
├── Environment/    环境配置
├── Error/          错误枚举
├── Models/         响应 Model (JSONResponse)
├── Rx/             RxSwift 扩展
├── Utils/          Configuration + Logger
└── VoiceChat/      VoiceChat 独立子模块
```

**依赖**: Alamofire, RxSwift, SmartCodable, CryptoKit

## 二、AFNetworking → Alamofire 迁移模式

### 迁移策略（方案 B：纯 NetworkCore）

1. **扩展 NetworkCore** — 添加 Sign 签名、Multipart 上传、文件下载能力
2. **创建 BusinessTarget 协议** — 统一签名逻辑，自动计算 Sign header
3. **每个 API 域一个枚举** — AccountAPI/DeviceAPI/GlassesAPI 等
4. **提供兼容层** — `sendRequest()` 保持旧回调签名 `(Bool, [String: Any]?, String?)`
5. **业务层逐文件迁移** — `import ServiceLayer` → `import NetworkCore`

### 关键设计：BusinessTarget 协议链

```swift
NetworkTargetType          ← 请求抽象 (associatedtype Response: SmartDecodable)
  └─ ServiceAPITarget      ← 绑定 ServiceType，自动解析 baseURL/headers
      └─ BusinessTarget      ← 业务 API，自动计算 Sign 签名
          ├─ AccountTarget  ← 17 个账户 API
          ├─ DeviceTarget   ← 9 个设备 API
          ├─ GlassesTarget  ← 11 个眼镜 API
          └─ ...
```

### Sign 签名逻辑

```swift
// BusinessTarget.headers 自动计算
var headers = NetworkCoreConfiguration.shared.mergedHeaders(for: service)
// 参数转大写 key → 合并 headers → 排序 → 拼接 → MD5
let upperParams = parameters.mapKeys { $0.prefix(1).uppercased() + $0.dropFirst() }
let merged = headers.merging(upperParams) { headers, _ in headers }
let signString = merged.keys.sorted().map { "\($0)\(merged[$0]!)" }.joined()
headers["Sign"] = signString.md5
```

### 兼容旧代码

```swift
// 旧代码
ServiceLayerAPI.defaultService().fetchUserInfo(userId: uid, countryCode: "CN") {
    (success: Bool, dict: [String: Any]?, message: String?) in
}

// 新代码（兼容写法，回调签名一样）
NetworkCore.shared.sendRequest(AccountTarget(.fetchUserInfo(uid: uid))) {
    (success, dict, message) in
}

// 新代码（async/await）
let dict = try await NetworkCore.shared.sendRequest(AccountTarget(.fetchUserInfo(uid: uid)))
```

### SmartDecodable conformance

```swift
// NetworkCore typealias（关键！）
public typealias SmartDecodable = SmartCodable.SmartDecodable  // 仅需 Decodable + init()
// 不要用 SmartCodableX（要求 Decodable + Encodable）

// 响应 Model 必须实现
public final class JSONResponse: SmartCodable.SmartDecodable {
    public let json: [String: Any]
    public required init() { self.json = [:] }           // 协议要求
    public required init(from decoder: Decoder) throws { ... }  // 解码
}
```

## 三、线程安全模式

### plugins — init 后不可变

```swift
// ✗ public private(set) var — 并发不安全
// ✓ public let — init 后不可变，天然线程安全
public let plugins: [NetworkPlugin]
```

### authPlugin — 独立锁

```swift
// ✗ 用 withLock（NSLock）+ @MainActor 混用 → 死锁风险
// ✓ 独立 authLock，与 serviceConfigurations 的锁分开
private let authLock = NSLock()

@MainActor
public func configureAuth(tokenRefresher: TokenRefreshable) {
    let plugin = AuthInterceptor(tokenRefresher: tokenRefresher)
    authLock.lock()
    self.authPlugin = plugin
    let alreadyHas = plugins.contains(where: { $0 is AuthInterceptor })
    authLock.unlock()
    if !alreadyHas { withLock { plugins.append(plugin) } }
}
```

### voiceChatClientContext — 计算属性 + 锁

```swift
// ✗ public var — 无锁保护
// ✓ 计算属性，读写均走 withLock
public var voiceChatClientContext: (any VoiceChatClientContext)? {
    get { withLock { _voiceChatClientContext } }
    set { withLock { _voiceChatClientContext = newValue } }
}
private var _voiceChatClientContext: (any VoiceChatClientContext)?
```

### willSend — 不依赖 Alamofire 内部 API

```swift
// ✗ dataRequest.onURLRequestCreation { ... } — Alamofire 内部 API，时机不对
// ✓ 手动构造 URLRequest，在请求发出前调用
var urlRequest = URLRequest(url: url)
urlRequest.httpMethod = afMethod.rawValue
urlRequest.timeoutInterval = request.timeoutInterval
for (key, value) in request.headers { urlRequest.setValue(value, forHTTPHeaderField: key) }
allPlugins.forEach { $0.willSend(urlRequest) }
```

### token 刷新重试 — 跳过插件

```swift
// ✗ 重试时传 self.plugins → willSend 被触发两次
// ✓ 重试时传空数组
_ = self.networkClient.send(request, plugins: []) { retryResult in
    self.decodeAndDeliver(retryResult, ...)
}
```

### completion 回调 — 统一主线程

```swift
// 所有 completion 回调统一通过 DispatchQueue.main.async 派发
// token 刷新走 DispatchQueue.main.async { Task { ... } }
// 不要 Task { @MainActor } 混用 Alamofire 回调线程
```

### 协议扩展 fatalError 陷阱

```swift
// ✗ 协议扩展中 fatalError — 运行时崩溃
extension NetworkClientProtocol {
    func upload(...) { fatalError("not implemented") }
}

// ✓ 引入子协议，仅具体类型实现
public protocol NetworkUploadCapable: NetworkClientProtocol {
    func upload(...) -> NetworkCancellable?
}
extension NetworkClient: NetworkUploadCapable { ... }
// 调用方: (client as? NetworkUploadCapable)?.upload(...)
```

## 四、代码审查清单（9 维度）

| 维度 | 检查项 |
|------|--------|
| 架构质量 | 协议设计、模块职责、依赖关系 |
| 类型安全 | 泛型使用、SmartDecodable conformance、associatedtype |
| 并发安全 | Sendable、锁保护、@MainActor 隔离 |
| 错误处理 | HTTPError 枚举覆盖度、token/HTTP/解码/取消 |
| 插件系统 | willSend/didReceive 时机、副作用控制 |
| 上传/下载 | Multipart 实现、进度回调、能力协议 |
| 兼容层 | 旧回调签名兼容、async/RxSwift 支持 |
| 业务 API | Sign 签名统一、Target 枚举设计 |
| 潜在问题 | 编译警告、运行时风险、死代码 |

### 严重性分级

- **P0** — 编译失败/运行时崩溃（fatalError、线程不安全、锁冲突）
- **P1** — 潜在 bug 或架构缺陷（插件重复触发、编码丢失类型、废弃 API）
- **P2** — 代码质量/可维护性（死代码、文件臃肿、逻辑重复）
- **P3** — 风格/改进（类型安全、日志脱敏、性能优化）

## 五、AnyCodable 递归编码

```swift
// ✗ compactMap { "\($0)" } — 全部变成字符串，丢失类型
// ✓ 递归编码，保留类型
public func encode(to encoder: Encoder) throws {
    var container = encoder.singleValueContainer()
    if let bool = value as? Bool { try container.encode(bool) }
    else if let int = value as? Int { try container.encode(int) }
    else if let double = value as? Double { try container.encode(double) }
    else if let string = value as? String { try container.encode(string) }
    else if let array = value as? [Any] {
        try container.encode(array.map { AnyCodable(value: $0) })  // 递归
    } else if let dict = value as? [String: Any] {
        try container.encode(dict.mapValues { AnyCodable(value: $0) })  // 递归
    } else { try container.encodeNil() }
}

// 需要添加 init(value:)
public init(value: Any) { self.value = value }
```

## 六、模块拆分模式（NetworkCore + ServiceType）

当通用网络库膨胀了业务代码时，拆分为两个模块：

```
NetworkCore (通用网络库) — 零业务依赖，任何项目可直接使用
├── Core/        NetworkClient, NetworkClient, TargetType, Upload, Multipart
├── Plugin/      Auth, Logger, RequestID（不含 SignPlugin）
├── Parser/      SmartDecoder, GraphQLDecoder, Connection
├── Environment/ NetworkEnvironment
├── Error/       HTTPError
├── Models/      JSONResponse
├── Utils/       NetworkCoreConfiguration, NetworkCoreLogger
├── Rx/          NetworkClient+Rx
└── Service/     ServiceType（空枚举）, ServiceAPITarget, NetworkAuth, TokenRefreshable

ServiceType (业务网络层) — 依赖 NetworkCore
├── API/         Account, Device, Glasses, Feedback, Live, Meditation, Pay, OTA, VIP
├── Service/     BusinessTarget, ServiceType+BizAPI（添加 .bizAPI/.voiceChat 标识）
├── VoiceChat/   API, Models, Targets, ClientContext, RequestFactory, Error
├── Plugin/      SignPlugin（业务特定签名）
├── Legacy/      NetworkCore+Legacy（兼容旧代码）
└── ServiceType.swift（全局入口 + configure()）
```

### 拆分原则

1. **NetworkCore 的 ServiceType 是空枚举** — 业务层通过 extension 添加标识
2. **ServiceAPITarget 留在 NetworkCore** — 通用协议，业务层扩展为 BusinessTarget
3. **SignPlugin 移到 ServiceType** — 签名是业务特定逻辑
4. **voiceChatClientContext 移到 ServiceType** — 业务特定属性
5. **configureServices() 在 NetworkCore** — 通用服务注册
6. **configureAppServices() 在 ServiceType** — 兼容旧代码 + voiceChatContext

### 最终模块状态（用户重构后）

用户进一步优化后的最终状态：

1. **NetworkCore 入口** — `NetworkCore.shared` 含 Logger/RequestID，开箱即用
2. **ServiceType 入口** — 仅 `@_exported import NetworkCore`，无独立 shared/configure
3. **HTTPMethod 导出** — `public typealias HTTPMethod = Alamofire.HTTPMethod`，NSK 不直接依赖 Alamofire
4. **VoiceChatConfiguration** — 独立 enum，不污染 NetworkCoreConfiguration
5. **RequestSigner** — 签名收敛到独立 enum，BusinessTarget 调用之
6. **App 层统一 import ServiceType** — 业务文件只需一个 import

```swift
// ServiceType.swift — 最终版本
@_exported import NetworkCore  // 消费者只需 import ServiceType

// NetworkCore.swift — 通用入口
public enum NetworkCore {
    public static let shared: NetworkClient = {
        NetworkClient.build()
            .add(plugin: LoggerInterceptor())
            .add(plugin: RequestIDInterceptor())
            .build()
    }()
}

// NetworkCore/NetworkTargetType.swift — 导出 HTTPMethod
public typealias HTTPMethod = Alamofire.HTTPMethod
```

### Package.swift 依赖

```swift
// ServiceType/Package.swift
let package = Package(
    name: "ServiceType",
    platforms: [.iOS(.v16)],
    dependencies: [
        .package(path: "../NetworkCore"),  // 本地依赖
    ],
    targets: [
        .target(name: "ServiceType", dependencies: ["NetworkCore"]),
    ]
)
```

### 拆分后 Xcode 配置步骤

1. 创建新 SPM 模块目录和 Package.swift
2. 迁移文件，添加 `import NetworkCore`
3. 语法检查：`swiftc -parse -sdk $(xcrun --sdk iphonesimulator --show-sdk-path) -target arm64-apple-ios16.0-simulator -I <build_dir> <file>`
4. Xcode → File → Add Package Dependencies → Add Local... → 选择新模块目录
5. 添加到 target
6. 更新业务层 import：`import ServiceType`（需要业务 API 的文件）
7. ⌘B 验证编译

**注意**: 不要用 `swift build` 编译 iOS-only 包，会报 macOS 平台不兼容。用 `swiftc -parse` 做语法检查，用 Xcode ⌘B 验证编译。

### 文件命名规范（Apple API Design Guidelines）

- **API 枚举**: `{Domain}API.swift` — AccountAPI, DeviceAPI, GlassesAPI
- **Target 结构体**: `{Domain}Target.swift` — 嵌入 API 文件中
- **协议**: `{Feature}Capable.swift` — NetworkUploadCapable
- **插件**: `{Function}Plugin.swift` — SignPlugin, AuthInterceptor
- **配置**: `{Module}Configuration.swift` — NetworkCoreConfiguration
- **兼容层**: `{Module}+Legacy.swift` — NetworkCore+Legacy

## 七、批量迁移模式（ObjC 网络栈 → Swift）

### 大规模迁移策略

49+ 文件的迁移不能一次性委托 Claude Code（会超时 600s）。推荐分阶段：

1. **批量 import 替换**（脚本，秒级完成）
2. **批量 API 调用替换**（脚本 + 手动修复）
3. **复杂调用手动适配**（delegate_task 分批）

### Phase 1: 批量 import 替换

```bash
find SmartGlasses -name "*.swift" -exec sed -i '' 's/import ServiceLayer/import ServiceType/' {} \;
find SmartGlasses -name "*.swift" -exec sed -i '' 's/import DomainNetwork//;s/import DomainNetworkLayer//' {} \;
```

### Phase 2: 批量 API 调用替换

```bash
# @_exported import 后，正确调用是 NetworkCore.shared
find SmartGlasses -name "*.swift" -exec sed -i '' 's/ServiceType\.shared/NetworkCore.shared/g' {} \;
```

### Phase 3: 复杂调用手动适配

delegate_task 分批（每批 10-15 个文件），处理：
- 多行 API 调用 → 拆分为变量
- 参数名差异（verifyCode vs captcha）
- 类型转换（Int → String: `"\(userId)"`）
- 枚举值映射

### ObjC 模块移除时的类型迁移

```bash
cp Modules/DomainNetwork/Classes/User.swift SmartGlasses/AppGroup/
cp Modules/DomainNetwork/Classes/UserManager.swift SmartGlasses/AppGroup/
# 在 Xcode 中手动添加到 target
```

### Crash Reporting 独立于网络层

旧代码使用 `DomainNetworkCore.multipartFormRequest` 上传 crash 日志，改用 `URLSession` 直接实现：

```swift
let task = URLSession.shared.uploadTask(with: request, from: body) { data, response, error in
    completionQueue.async { completionBlock(data != nil) }
}
task.resume()
```

## 八、迁移后审查发现（2026-05-30）

迁移完成后的架构审查发现的遗留问题，按优先级分类。

### P1 — 类型安全未落地

**问题**: 所有 API Target 均使用 `JSONResponse`，`associatedtype Response: SmartDecodable` 的编译时类型绑定完全未发挥。

```swift
// 现状 — 所有 Target 都是这个
public typealias Response = JSONResponse  // → [String: Any]

// 期望 — 每个 API 有自己的 Response Model
public typealias Response = UserInfoResponse  // → 编译时类型检查
```

**影响**: 业务层仍需手动解析字典，类型错误只能在运行时发现。

**修复策略**: 逐步迁移，优先高频 API（登录、用户信息、设备列表）。

### P1 — NetworkCoreConfiguration 可写属性无锁保护

**问题**: `environment`/`globalHeaders`/`plugins` 三个公开可写属性用 `@unchecked Sendable` 绕过编译器检查，但无锁保护。

```swift
// 现状 — @unchecked Sendable，实际不安全
public final class NetworkCoreConfiguration: @unchecked Sendable {
    public var environment: NetworkEnvironment = .development  // 无锁
    public var globalHeaders: [String: String] = [:]           // 无锁
    public var plugins: [NetworkPlugin] = []                   // 无锁
}

// 修复 — 改为 private set + 带锁方法
public private(set) var environment: NetworkEnvironment = .development
public func setEnvironment(_ env: NetworkEnvironment) {
    withLock { environment = env }
}
```

### P2 — @MainActor 隔离不一致

**问题**: callback 路径和 async/await 路径的 token 刷新行为不一致。

```swift
// callback 路径 — DispatchQueue.main.async { Task { ... } }
DispatchQueue.main.async {
    Task { @MainActor in
        let refreshed = await self.authPlugin.refreshTokenIfNeeded()
    }
}

// async 路径 — 直接 await
let refreshed = await authPlugin.refreshTokenIfNeeded()
```

**修复**: 统一使用 `await MainActor.run { }` 替代 `DispatchQueue.main.async`。

### P2 — HTTPError 不符合 Sendable

**问题**: `HTTPError` 包含 `case networkError(Error)`，`Error` 不是 `Sendable`。

```swift
// 现状 — 不满足 Swift 6 strict concurrency
public indirect enum HTTPError: Error, LocalizedError {
    case networkError(Error)        // Error 不是 Sendable
    case tokenRefreshFailed(Error)  // Error 不是 Sendable
}

// 修复 — 改为 String 描述
case networkError(String)
case tokenRefreshFailed(String)
```

### P2 — 未使用的依赖

**问题**: SwiftyJSON 和 RxCocoa 在 NetworkCore 源码中无任何引用。

**修复**: 从 Package.swift 移除，减少编译时间和包体积。

### P3 — 重试 Session 从未使用

**问题**: `HTTPSession.makeDefault()` 创建的带 `RetryPolicy` 的 Session 从未被调用，`NetworkCore.shared` 使用的是 `Session.default`。

```swift
// HTTPSession.swift — 工厂方法存在但未使用
public static func makeDefault() -> Session {
    let interceptor = RetryPolicy(retryLimit: 3, retryableHTTPMethods: [.get])
    return Session(interceptor: interceptor)
}

// NetworkCore.swift — 使用的是默认 Session
public static let shared: NetworkClient = {
    NetworkClient.build()
        .add(plugin: LoggerInterceptor())
        .add(plugin: RequestIDInterceptor())
        .build()
}()
```

**修复**: 让 `NetworkCore.shared` 使用 `HTTPSession.makeFromEnvironment()` 启用重试。

### P3 — LoggerInterceptor Release 默认启用

**问题**: 生产环境不应输出请求/响应 body 日志。

```swift
// 现状 — 始终启用
.add(plugin: LoggerInterceptor())

// 修复 — 根据环境禁用
.add(plugin: LoggerInterceptor(isEnabled: isDebug))
```

### P3 — AuthInterceptor.pendingRequests 无容量限制

**问题**: 短时间大量 401 响应会导致 `pendingRequests` 无限增长。

**修复**: 添加上限或使用 `AsyncStream`。

## 八、日志脱敏

```swift
// LoggerInterceptor 中对敏感字段脱敏
private let sensitiveHeaders: Set<String> = ["authorization", "sign", "token", "cookie"]
private let sensitiveBodyKeys: Set<String> = [
    "password", "newPassword", "token", "accessToken", "idToken", "captcha"
]

// headers 脱敏
for key in sanitized.keys {
    if sensitiveHeaders.contains(key.lowercased()) { sanitized[key] = "***REDACTED***" }
}

// body 脱敏 — JSON 序列化后替换
if let json = try? JSONSerialization.jsonObject(with: body) as? [String: Any] {
    var sanitized = json
    for key in sanitized.keys where sensitiveBodyKeys.contains(key) {
        sanitized[key] = "***REDACTED***"
    }
}
```
