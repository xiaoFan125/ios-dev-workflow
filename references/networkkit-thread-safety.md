# NetworkCore 线程安全模式

## 问题：NSLock + @MainActor 混用死锁

```swift
// ✗ 错误：@MainActor 方法内持有 NSLock
@MainActor
public func configureAuth(tokenRefresher: TokenRefreshable) {
    let plugin = AuthInterceptor(tokenRefresher: tokenRefresher)
    withLock {  // NSLock.lock() 在 @MainActor 上
        self.authPlugin = plugin
        if !plugins.contains(where: { $0 is AuthInterceptor }) {
            plugins.append(plugin)
        }
    }
    // 如果其他线程持有 withLock 的锁并等待主线程 → 死锁
}
```

## 修复模式

### 1. 分离锁

不同关注点用不同的锁：

```swift
private let lock = NSLock()           // 保护 serviceConfigurations
private let authLock = NSLock()       // 保护 authPlugin

@MainActor
public func configureAuth(tokenRefresher: TokenRefreshable) {
    let plugin = AuthInterceptor(tokenRefresher: tokenRefresher)
    authLock.lock()
    self.authPlugin = plugin
    let alreadyHasPlugin = plugins.contains(where: { $0 is AuthInterceptor })
    authLock.unlock()
    if !alreadyHasPlugin {
        withLock { plugins.append(plugin) }  // 用另一个锁
    }
}
```

### 2. 共享可变状态用计算属性+锁

```swift
// ✗ 错误：直接暴露 var
public var voiceChatClientContext: (any VoiceChatClientContext)?

// ✓ 正确：计算属性+锁
private static let contextLock = NSLock()
private static var _voiceChatClientContext: (any VoiceChatClientContext)?

public var voiceChatClientContext: (any VoiceChatClientContext)? {
    get {
        Self.contextLock.lock()
        defer { Self.contextLock.unlock() }
        return Self._voiceChatClientContext
    }
    set {
        Self.contextLock.lock()
        defer { Self.contextLock.unlock() }
        Self._voiceChatClientContext = newValue
    }
}
```

### 3. 不可变属性优先

```swift
// ✗ 错误：可变公开属性，无锁保护
public private(set) var plugins: [NetworkPlugin]

// ✓ 正确：init 后不可变
public let plugins: [NetworkPlugin]
```

### 4. Token 刷新重试不重复触发插件

```swift
// ✗ 错误：重试时传 plugins，willSend 被触发两次
_ = self.networkClient.send(request, plugins: self.plugins) { ... }

// ✓ 正确：重试时不传 plugins
_ = self.networkClient.send(request, plugins: []) { ... }
```

## 检查清单

- [ ] `@MainActor` 方法内不持有 NSLock（或用独立锁）
- [ ] 共享可变状态用计算属性+锁包装
- [ ] 插件列表 `init` 后用 `let` 不可变
- [ ] Token 刷新重试不重复触发插件
- [ ] 所有 completion 回调统一在 main queue
