---
name: ios-dev-workflow
description: "Hermes 调度 Claude Code（深度实现）+ Codex（验证/备选）的 iOS 开发范式，遵循控制论六原则。"
author: Hermes Agent
license: MIT
platforms: [macos]
metadata:
  hermes:
    tags: [ios, swift, xcode, architecture, code-review, claude-code, codex]
    related_skills: [claude-code, codex, codebase-inspection]
---

# iOS 开发工作流

## 核心原理（控制论六原则）

1. **反馈闭环** — 每个动作必须有验证环节（截图确认、测试通过、用户反馈），形成"执行→检测→修正"的闭环
2. **系统稳定性** — 稳定优先于性能。不折腾、不冒进，宁可慢也不引入不可控风险
3. **信息流** — memory 存底层逻辑（信号），skill 存操作流程（通道）。区分噪声与信号，减少信息冗余
4. **自调节** — 遇到意外结果时停下来分析根因，不机械重试。系统应能自主纠错
5. **层次结构** — 复杂系统分层处理：战略层（架构）→战术层（模块）→执行层（代码）
6. **黑箱方法** — 关注输入输出而非内部实现，用可观测结果验证不可观测过程

## 三工具分工

| 角色 | 定位 | 职责 |
|------|------|------|
| **Hermes** | 总控调度 | 拆分需求、生成任务说明、分发到 Claude Code/Codex、汇总结果、验收 |
| **Claude Code** | 主执行线 | 深度项目阅读、主线实现、跨文件重构、复杂依赖排查 |
| **Codex** | 第二执行线 | 方案验证、备选实现、批量命令、关键逻辑复核 |

**核心原则**: Hermes 是调度员，不是执行者。编码工作交给 Claude Code 和 Codex，Hermes 负责拆分、分发、汇总、验收。

## 工作流分层

### → Phase 0: 需求分析与任务拆分（Hermes）

**工具**: Hermes

**流程**:
1. 接收用户需求，理解"想达到什么效果"
2. 分析任务复杂度，判断是否需要双执行线
3. 生成结构化任务说明：
   - **Claude Code 任务**: 主实现（修改哪些模块、允许修改的文件范围、验收条件）
   - **Codex 任务**: 验证/备选（需要检查的逻辑、备选方案方向、只读或可写）
4. 约定修改边界和验收标准

**判断是否需要 Codex 参与**:
- 单文件小修改 → Claude Code 即可，不需要 Codex
- 跨文件重构/复杂逻辑 → Claude Code 实现 + Codex 验证
- 方案不确定 → Claude Code 方案 A + Codex 方案 B，Hermes 对比
- 批量操作 → Codex 执行，Claude Code 审查

**输出**: 两份结构化任务文件（或一份，如果不需要 Codex）

### → Phase 1: 深度实现（Claude Code — 主执行线）

**工具**: Claude Code

**流程**:
1. 深度阅读项目结构和相关代码
2. 按任务说明实施变更
3. 跨文件重构、依赖梳理、配置调整
4. 生成变更摘要

**调用方式**:
```bash
# 简单任务（<30s）
claude -p "任务说明" --allowedTools "Read,Edit,Bash" --max-turns 5 --model sonnet

# 复杂任务（>30s，必须用 stream-json 防止阻断）
claude -p "任务说明" --allowedTools "Read,Edit,Bash" --max-turns 10 --model sonnet \
  --output-format stream-json --verbose
```

**验证**: 编译通过、单元测试通过、变更摘要完整

### → Phase 2: 验证与备选（Codex — 第二执行线）

**工具**: Codex

**流程**:
1. 对 Claude Code 的实现进行逻辑复核
2. 生成备选实现方案（如果需要）
3. 执行批量检查命令
4. 输出验证报告

**调用方式**:
```bash
# 验证模式（只读）
codex exec "审查以下代码变更，检查逻辑错误和潜在风险：$(git diff)"

# 备选方案模式
codex exec "用不同方式实现 [功能]，给出备选方案"

# 批量检查模式
codex exec "检查所有 API 调用是否缺少 uid 参数"
```

**关键规则**:
- **不要同时修改同一批文件** — 如果 Claude Code 已修改，Codex 只做只读验证
- **Codex 输出验证报告** — 包含：逻辑正确性、边界条件、潜在风险
- **备选方案单独分支** — 如果 Codex 生成备选实现，放在独立 git 分支

### → Phase 3: 汇总与验收（Hermes）

**工具**: Hermes

**流程**:
1. 收集 Claude Code 的变更摘要
2. 收集 Codex 的验证报告
3. 对比两边思路：
   - 共同结论 → 高置信度
   - 分歧点 → 需要用户决策
   - Codex 发现的风险 → 评估是否需要修复
4. 生成最终报告：问题清单（P0-P3）、修复计划、后续操作清单
5. 按预设验收标准判断是否通过

**验收标准模板**:
- ✅ 编译通过
- ✅ 单元测试通过
- ✅ 无 P0/P1 问题
- ✅ Codex 验证无高风险警告
- ✅ 变更范围不超出约定边界

### → Phase 4: 迭代优化（Hermes）

**工具**: Hermes

**流程**:
1. 修复验证 — 按优先级修复，每次修复后验证
2. 文档完善 — 更新 README、API 文档
3. 经验提炼 — 底层逻辑→memory，操作流程→skill
4. 每日整理 — 按控制论框架分类整理 skill

**验证**: 文档完整、经验已提炼

## Hermes 调度模式

### 任务分发模板

当用户提出开发需求时，Hermes 按以下模板生成任务说明：

```markdown
## 任务: [功能名称]

### 背景
- 项目: [项目名]
- 模块: [涉及模块]
- 目标: [要达到什么效果]

### Claude Code 任务（主实现）
- 修改范围: [允许修改的文件/目录]
- 实现要求: [具体实现细节]
- 验收条件: [编译通过、测试通过等]
- 约束: [不能动的文件、不能改的接口]

### Codex 任务（验证/备选）
- 验证范围: [需要检查的逻辑]
- 备选方向: [如果需要备选方案]
- 输出要求: [验证报告格式]
- 权限: [只读 / 可写（独立分支）]

### 共同约束
- 不能同时修改同一批文件
- 变更必须可回滚
- 出现错误立即停止并报告
```

### 简化模式（单执行线）

对于简单任务，只用 Claude Code，不启动 Codex：

```bash
# Hermes 直接委托 Claude Code
claude -p "修复 [具体问题]" --allowedTools "Read,Edit" --max-turns 5
```

### 并行模式（双执行线）

对于复杂任务，Claude Code 和 Codex 同时工作：

```bash
# Claude Code: 后台执行
terminal(command="cd /project && claude -p '主实现任务' --output-format stream-json --verbose > /tmp/cc_result.json 2>&1",
         background=true, notify_on_complete=true, timeout=300)

# Codex: 后台执行
terminal(command="cd /project && codex exec '验证任务' > /tmp/codex_result.txt 2>&1",
         background=true, notify_on_complete=true, timeout=300)

# 等待两者完成后读取结果
process(action="wait", session_id="<claude_session>")
process(action="wait", session_id="<codex_session>")
```

## Claude Code 长任务超时优化（实测验证）

**问题**: `terminal` 工具对长时间无输出的命令会触发安全阻断（"BLOCKED: Command timed out without user response"），即使设置了 `timeout=180` 也无效。

**根因**: Claude Code 在思考/读取文件时不产生 stdout 输出，终端工具认为命令卡死。

**解决方案**: 使用 `--output-format stream-json --verbose` 产生流式输出：

```bash
# ✗ 会被阻断（无中间输出）
claude -p "复杂任务..." --allowedTools "Read" --max-turns 8 --model sonnet

# ✓ 正常完成（stream-json 持续产生事件）
claude -p "复杂任务..." --allowedTools "Read" --max-turns 5 --model sonnet \
  --output-format stream-json --verbose 2>&1 | grep -o '"type":"result".*'
```

**实测数据**:

| 命令模式 | max-turns | 耗时 | 结果 |
|----------|-----------|------|------|
| 简单命令（无 --format） | 1-5 | <30s | ✅ 正常 |
| 复杂命令（无 --format） | 8 | >60s | ❌ 被阻断 |
| 复杂命令 + stream-json | 5 | 103s | ✅ 正常 |

**解析 stream-json 输出**:
```bash
# 提取最终结果
... | grep -o '"type":"result".*' | python3 -c "
import json, sys
line = sys.stdin.readline()
data = json.loads('{' + line[1:])
print(data['result'])
"
```

## 实测案例：sendRequest 审查

以下是一个完整的双执行线模式测试案例。

### Phase 0: 需求拆分（Hermes）

**任务**: 审查 `sendRequest` 的线程安全性和返回契约一致性。

**分析**:
- 跨文件审查任务，涉及实现侧 + 消费侧
- 需要深度阅读代码理解调用链
- 适合双执行线：Claude Code 做主分析，Codex 做交叉验证

**任务分配**:

| | Claude Code（主执行线） | Codex（第二执行线） |
|---|---|---|
| **目标** | 审查 sendRequest 的实现、线程安全、返回契约 | 从调用方角度验证使用是否一致 |
| **范围** | NetworkCore 源码（实现侧） | 业务层调用点（消费侧） |
| **权限** | 只读 | 只读 |
| **输出** | 审查报告：问题清单 P0-P3 | 验证报告：调用方是否正确处理返回值 |

### Phase 1: Claude Code 分析结果

Claude Code 读取了 `NetworkClient+BizLegacy.swift` 和 `NetworkClient.swift`，发现：

| # | 严重度 | 问题 | 位置 |
|---|--------|------|------|
| 1 | 中等 | `enableTokenRefreshRetry` 无同步保护，数据竞争 | `NetworkClient.swift:22` |
| 2 | 中等 | 闭包版 `send` 保证主线程回调，async 版不保证，契约不一致 | `NetworkClient.swift:49` vs `107` |
| 3 | 低 | token 刷新路径线程跳跃嵌套过深 | `NetworkClient.swift:59-74` |
| 4 | 低 | 重试跳过所有插件，可能影响请求正确性 | `NetworkClient.swift:67` |
| 5 | 信息 | `sendRequest` 只读 `"msg"` 字段，错误信息可能丢失 | `BizLegacy.swift:29` |

### Phase 2: Hermes 消费侧验证

Hermes 手动验证了 28 个调用点：

| 维度 | 发现 | 优先级 |
|------|------|--------|
| **调用模式** | ✅ 28 个调用点都使用 `checkError(dic:success:message:)` 检查业务错误 | — |
| **返回值使用** | ✅ 调用方直接访问 `dictionary`，假设是完整 JSON body，与实现一致 | — |
| **闭包捕获** | ⚠️ 部分调用点使用 `self.` 而非 `[weak self]` | P2 |
| **@discardableResult** | ✅ 正确 — 大部分调用点用 `_ =` 忽略返回值 | — |

### Phase 3: 综合评估（Hermes）

| 问题 | 优先级 | 描述 | 修复建议 |
|------|--------|------|----------|
| enableTokenRefreshRetry 数据竞争 | P2 | 可变公开属性无同步保护 | 改为 `@Atomic` 或 `let` |
| async 版不保证主线程 | P2 | `performSend` 的 continuation 在任意线程 resume | 文档标注或加 `@MainActor` |
| 闭包强引用 | P2 | 部分调用点未使用 `[weak self]` | 全局扫描修复 |

**验收结论**: ✅ 通过 — 无 P0/P1 问题，P2 为代码健壮性改进。

### 流程总结

| Phase | 工具 | 状态 | 耗时 |
|-------|------|------|------|
| Phase 0: 需求拆分 | Hermes | ✅ 完成 | ~1min |
| Phase 1: 实现审查 | Claude Code (stream-json) | ✅ 完成 | ~2min |
| Phase 2: 消费侧验证 | Hermes 手动 | ✅ 完成 | ~2min |
| Phase 3: 汇总验收 | Hermes | ✅ 完成 | ~1min |

## 设计决策闭环（控制论视角）

**原则**: 先出方案对比，用户选择，再动手实现。避免开环设计。

```
用户需求 → Hermes 分析 → 生成 2-3 个方案 → 用户选择 → Claude Code 实现 → Codex 验证 → Hermes 汇总
```

**✗ 开环设计**: 直接做一个方案交差，无反馈
**✓ 闭环设计**: 多方案对比 → 用户选择 → 实现 → 验证

## 错误处理原则（控制论视角）

**原则**: 遇到意外结果时停下来分析根因，不机械重试。

```swift
// ✗ 机械重试
func retryRequest() {
    networkClient.send(request) // 可能无限重试
}

// ✓ 根因分析
func handleFailure(_ error: Error) {
    switch error {
    case .networkTimeout:
        retryWithBackoff()        // 网络超时 → 重试
    case .invalidToken:
        refreshTokenThenRetry()   // Token 过期 → 刷新后重试
    case .serverError:
        reportError(error)        // 服务端错误 → 不重试，上报
    default:
        logError(error)           // 未知错误 → 记录日志
    }
}
```

**原则**: 选择最稳妥的修正路径，而非最快的。

## 信息分层存储规则

### → memory（底层逻辑/控制论原理）

- 用户偏好中可归纳为"元规则"的部分
- 环境事实中影响决策的稳定约束
- 工具的底层行为规律（非具体命令）

### → skill（操作流程/具体方法）

- 具体的操作步骤和命令序列
- 特定项目的配置和架构细节
- 踩坑记录和解决方案
- 可复用的工作流模板

### 判断标准

问自己："换了项目/环境，这条知识还成立吗？"
- 成立 → memory（如：用户偏好先出方案再动手）
- 不成立 → skill（如：项目的端口架构）

## 每日 skill 整理流程

工作结束时按控制论框架重新组织 skill：

1. **反馈回路类** — 含验证步骤、测试、确认环节的 skill
2. **稳定性类** — 含防护措施、回滚方案、容错设计的 skill
3. **信息流类** — 数据采集、传输、格式转换相关 skill
4. **优化类** — 性能提升、流程精简、自动化相关 skill
5. **层次类** — 架构设计、模块划分、抽象层级相关 skill

整理时检查：
- 每个 skill 是否包含反馈验证步骤？没有就补
- 是否有冗余 skill 可以合并？
- 底层逻辑是否泄露到了 skill 中？（应该在 memory）
- 新发现的底层规律是否已提炼到 memory？

## 文档风格偏好

**用户偏好简洁、结构化的文档风格**，参考 cybernetics-framework 仓库：

- README: 一句话描述 → 适用场景（列表）→ 核心原理（表格）→ 快速使用（命令）→ 文件结构（树形图）→ License
- SKILL: YAML frontmatter → 核心原理（列表）→ 流程/规则（分层）→ 示例（✓/✗ 对比）
- 避免冗长的段落，优先使用表格、列表、代码块

## 常见 Pitfalls

### 工作流执行 Pitfalls

1. **Claude Code 长任务被 terminal 安全机制阻断** — Hermes 的 `terminal` 工具对超过约 30 秒无输出的命令会触发 `BLOCKED: Command timed out without user response`。
   **解决方案（按优先级）**:
   - **stream-json 模式**: `--output-format stream-json --verbose` 持续产生输出，防止阻断
   - **后台模式**: `terminal(command="...", background=true, notify_on_complete=true)`
   - **delegate_task**: 委托子代理执行，子代理有自己的终端会话
   - **拆分任务**: 将大任务拆分为 ≤ 30s 的小步骤

2. **Claude Code 与 Codex 并行执行** — 双执行线并行时，用 `terminal(background=true)` 启动两个后台任务，用 `process(action="poll")` 检查进度

3. **中文路径符号链接** — Claude Code 不支持中文 workdir 路径，`/tmp` 下创建符号链接可能被 macOS 沙箱阻止。**修复**: 用 `~/.hermes/tmp/` 替代：
   ```bash
   mkdir -p ~/.hermes/tmp
   ln -sf "/Users/xxx/Desktop/项目/MyApp" ~/.hermes/tmp/myapp
   cd ~/.hermes/tmp/myapp && claude -p "task"
   ```

4. **Codex 未安装或不在 PATH** — `codex` 二进制可能安装在 `~/.hermes/node/bin/` 但不在 PATH 中。**修复**:
   ```bash
   npm install -g @openai/codex
   ln -sf ~/.hermes/node/bin/codex ~/.local/bin/codex
   codex --version  # 验证
   ```

### iOS 开发 Pitfalls

5. **重构前不理解原栈** — 用户批评："有些问题反复修改，没有理解原网络栈，重构后还要修改多次，多处对应不上"。**根因**: 没有先完整理解原栈就开始重构。**修复**: 强制执行分析流程：①画原栈数据流；②标注关键决策点；③创建完整对照表（原栈 vs 新栈）；④确认分工后再动手。如果对照表有 3 个以上"未实现"项，停止编码，先补充分析。详见 `ios-network-layer-migration` skill 的 "Pre-Migration Analysis" 章节
6. **中文路径问题** — Claude Code 的 `workdir` 参数不支持中文路径，需要创建符号链接: `ln -sf "/path/with/中文/project" /tmp/project`
7. **枚举关联值比较** — 有关联值的 enum 不能用 `==`，必须用 `if case` 模式匹配
8. **泛型闭包类型推断** — 泛型参数在闭包捕获后可能丢失，需要显式传入类型
9. **@MainActor 级联** — 调用 @MainActor 方法的函数自己也必须是 @MainActor
10. **defer + DispatchQueue.main.async** — defer 在闭包外部执行，需要在 async block 内部手动调用
11. **Agora SDK `didAudioSubscribeStateChange` 不触发** — Agora SDK 4.6.2 在当前配置下不会触发此回调，即使设置了 `enableAudioVolumeIndication`（joinChannel 前后均调用）和 `enableAudioRecordingOrPlayout = true`。**修复**: 使用 `remoteUserDidJoin` 作为握手触发备用方案，在 `remoteUserDidJoin` 中调用 `handshakeGate.markAudioSubscribed()` + `performHandshakeIfReady()`
12. **SPM iOS-only 包命令行编译失败** — `swift build` 默认编译 macOS，但 `Package.swift` 只声明 `.iOS(.v16)` 平台时会报 SmartCodable macOS 版本不兼容。**修复**: 用 `xcodebuild -sdk iphonesimulator` 编译，或用 `swiftc -parse -sdk $(xcrun --sdk iphonesimulator --show-sdk-path) -target arm64-apple-ios16.0-simulator` 做语法检查。不要用 `swift build` 编译 iOS-only 包
13. **Swift extension 访问 private 属性** — Swift extension 无法访问 `private` 属性。**修复**: 将需要跨文件访问的属性改为 `internal`（默认），或添加 `internal` 计算属性暴露给 extension。常见于 `NetworkClient.session`、`NetworkClient.networkClient` 等场景
14. **SmartCodable 类型歧义** — 同一 Module 中两个文件定义了同名类型（如 `JSONResponse`），Xcode 报 "ambiguous for type lookup"。**修复**: 搜索确认只有一个定义，删除重复。常见于从旧代码复制 Model 时忘记删除原定义
15. **SmartCodable conformance** — `SmartCodableX` 要求同时满足 `Decodable + Encodable`，响应 Model 只需解码时应使用 `SmartCodable.SmartDecodable`（仅需 `Decodable + init()`）。**错误**: `Type 'X' does not conform to protocol 'SmartDecodable'`。**修复**: `typealias SmartDecodable = SmartCodable.SmartDecodable` + `required init() { ... }`
16. **SPM 扩展中泛型约束** — extension 中调用 `send(_:)` 时，如果 target 的 `Response` 类型是泛型参数，编译器无法推断 `response.json`。**修复**: 用 `some Protocol` 语法约束参数类型，或显式声明 `(result: Result<ConcreteType, HTTPError>)` 帮助编译器推断
17. **眼镜 BLE 音频流不停止** — 语音会话结束后眼镜持续发送 `type: 3, len: 656` 音频帧。**原因**: `VoiceCaptureController.stopCapture()` 发送的 BLE 停止命令可能未成功执行。**排查**: 搜索日志 `GlassVoiceCapture` 确认是否发送了 `mic ok=false`、`AI status finished`、`micPickup stop`、`voice enter disable` 命令
18. **stale currentModel 导致断连设备显示连接态** — 多设备场景下 `applyCurrentBleConnectionState` 的兜底分支会保留 `currentModel.value` 的旧 connected 态，即使 BLE live 已切换到另一设备。**修复**: 兜底分支增加 live 设备一致性检查，如果 `liveHeadset` 已指向另一设备则 reset 而非 preserve
19. **pair_connectionStatus 条件过严导致仓电量不刷新** — `handleDeviceAddedSuccess` 中 `startBoxObserve()` 被 `pair_connectionStatus == .connected` 前置条件阻断，新搜索添加的设备仓电量从不刷新。**修复**: 移除前置条件，`startBoxObserve` 内部有 `isBoxLiveConnected` + `fetchedBoxInitialStateKeys` 自行保护
20. **isBoxAddDevice MAC 混淆导致仓回调归属错误** — 仓添加设备的 `mac_adress` 实为仓 MAC，`connect()` 用此 MAC 发起重连时实际连的是仓，`connectionStatusBlock` 收到 `.HeadsetCharging` 回调后可能错误更新耳机主连接态。**修复**: 仓回调走独立路径 `applyBoxConnectedCallback`，仅写 pair 态不写 connectionStatus
21. **debounced 订阅竞态导致顶栏 icon 错位** — `viewWillAppear` 的 `refreshOnAppear` 和搜索页 `addSuccess` 闭包（0.2s 延迟）存在竞态。中间窗口 dataSource debounced 订阅可能用旧顺序刷新顶栏，后续 `handleDeviceAddedSuccess` 更新 dataSource 后 `reloadData()` 和之前的 `reloadItems` 冲突，导致第一个 cell 的 icon 停留在旧设备图片。**修复**: 新增 `deviceListDirectRefresh`（`PublishRelay<Void>`），`handleDeviceAddedSuccess` 完成后直接通知控制器刷新顶栏，绕过 200ms debounce。通用模式：**当 debounced 订阅和非 debounced 事件存在时序竞争时，为关键路径增加不经 debounce 的直接刷新通道**
22. **GitHub 仓库隐私清理（多轮扫描）** — 发布到公开仓库前必须**多轮扫描**，每轮聚焦不同维度：第一轮：个人路径（`/Users/xxx/...`→`~/Projects/...`）；第二轮：项目名/品牌名（YourApp→YourApp, OtherApp→OtherApp）；第三轮：文件名中的项目名（需 `git mv` 重命名）；第四轮：版本历史中的本地工作细节、提交信息中的项目引用；第五轮：references 目录中的项目特定内容。**关键教训**: 一次扫描不够，用户会发现遗漏，需要多次清理+重新推送
23. **gh repo delete 需要 delete_repo scope** — `gh auth refresh -h github.com -s delete_repo` 需要浏览器授权（device flow），CLI 中会超时。**修复**: 手动在 GitHub 网页 Settings → Danger Zone 删除，或让用户在浏览器中完成 device flow 授权
24. **SmartCodable 6.x `SmartCodableX` vs `SmartDecodable`** — NetworkCore 的 `typealias SmartDecodable = SmartCodableX` 要求同时满足 `Decodable + Encodable`，但 `JSONResponse` 等只解码不编码的类型无法满足 `Encodable`。**修复**: 改为 `typealias SmartDecodable = SmartCodable.SmartDecodable`（仅需 `Decodable + init()`），并给 `JSONResponse` 添加 `required init()`
25. **AFNetworking → Alamofire 迁移** — 网络层迁移模式：模块拆分（NetworkCore 通用 + ServiceType 业务）、@_exported import、Target 协议设计、Sign 签名收敛
26. **NSKeyedUnarchiver 类名迁移** — 移动类到不同模块时，旧归档数据的类名不匹配会导致解码失败。**修复**: 捕获异常后清除旧缓存数据，让用户重新登录
27. **API uid 参数类型** — 旧 ServiceLayerAPI 接受 Int 类型的 uid，新 API 定义使用 String。所有 uid 参数必须用 `"\(userId)"` 转换
28. **Swift 模块依赖不自动传递 import** — SPM 模块 A 依赖模块 B，模块 C 依赖模块 A，但 C 的文件仍需 `import B` 才能使用 B 的类型。**修复**: 在模块 A 入口使用 `@_exported import B`，或在 Package.swift 中显式声明依赖，或给需要的类型添加 `public typealias`
29. **`fatalError` 作为协议默认实现的风险** — 协议扩展中的 `fatalError()` 默认实现会在 mock/自定义实现未覆盖时崩溃。**修复**: 使用独立 capability 协议（如 `NetworkUploadCapable`），仅在具体实现类中提供 upload/download 方法，协议扩展不提供默认实现
30. **Swift Package 模块拆分** — 将单体 SPM 模块拆分为通用库+业务层时：(a) 业务层入口用 `@_exported import 通用库`，消费者只需 `import 业务层`；(b) 如果通用库依赖 Alamofire，用 `public typealias HTTPMethod = Alamofire.HTTPMethod` 导出，业务层不再直接依赖 Alamofire；(c) 业务特定的 Plugin（如 SignPlugin）放在业务层，通用 Plugin（Auth/Logger/RequestID）留在通用库；(d) 配置类不要用 extension 给通用库的类添加业务属性，用独立的 `XxxConfiguration` enum 持有
31. **NSLock + @MainActor 混用死锁** — `@MainActor` 方法内部调用 `NSLock.lock()` 可能死锁（其他线程持锁时等待主线程）。**修复**: (a) 分离锁：`authLock` 保护 authPlugin，`withLock` 保护其他配置；(b) 不要在 `@MainActor` 方法内持有 NSLock 的同时依赖 MainActor；(c) 共享可变状态用计算属性+锁包装，不要直接暴露 `var`
32. **Swift Package 编译验证** — SPM 命令行 `swift build` 默认编译 macOS，iOS-only 包会报平台版本错误。**修复**: 用 `swiftc -parse -sdk $(xcrun --sdk iphonesimulator --show-sdk-path) -target arm64-apple-ios16.0-simulator` 做语法检查，或用 Xcode 的 `xcodebuild` 编译
33. **Alamofire API 参数顺序** — `session.download()` 的 `requestModifier` 参数必须在 `to` 之前，否则编译报错 `Argument 'requestModifier' must precede argument 'to'`
34. **系统音量同步到 BLE 设备** — iOS 没有公开 API 直接监听音量键，需要监听私有通知 `AVSystemController_SystemVolumeDidChangeNotification`，通过 `AVAudioSession.sharedInstance().outputVolume` 获取当前音量（0.0-1.0），转换为 0-100 后通过 BLE 同步到设备
35. **iOS 大型网络层迁移模式（AFNetworking → Alamofire）** — 迁移旧 ObjC 网络栈到纯 Swift 时，推荐模式：①在新层创建 `BusinessTarget` 协议统一签名逻辑；②每个 API 域一个枚举（AccountAPI/DeviceAPI/GlassesAPI）；③提供 `sendRequest()` 兼容旧回调签名 `(Bool, [String: Any]?, String?)`；④业务层逐文件迁移 import + 调用。**模块拆分**: 当通用网络库膨胀了业务代码时，拆分为 `NetworkCore`（通用，零业务依赖）+ `ServiceType`（业务层，依赖 NetworkCore）
36. **NetworkCore 线程安全模式** — ①`plugins` 用 `let` 而非 `var`（init 后不可变）；②`authPlugin` 用独立 `authLock` 保护，不与 `withLock` 混用避免死锁；③`voiceChatClientContext` 用计算属性 + `withLock` 保护读写；④`willSend` 不依赖 Alamofire 内部 API（`onURLRequestCreation`），手动构造 URLRequest 调用；⑤token 刷新重试时传 `plugins: []` 避免 willSend 重复触发；⑥所有 completion 回调统一通过 `DispatchQueue.main.async` 派发
37. **协议扩展 fatalError 陷阱** — 协议扩展中的默认实现不应 `fatalError()`，运行时会崩溃。**修复**: 引入子协议（如 `NetworkUploadCapable`），仅在具体类型（`NetworkClient`）中实现 upload/download，调用方通过 `as? NetworkUploadCapable` 检查能力
38. **AnyCodable 递归编码** — `[Any]` 编码时不能 `compactMap { "\($0)" }` 转字符串（丢失类型）。**修复**: 递归编码，对 `[Any]` → `array.map { AnyCodable(value: $0) }`，对 `[String: Any]` → `dict.mapValues { AnyCodable(value: $0) }`，需添加 `init(value: Any)` 构造器
39. **CryptoKit 替代 CommonCrypto** — `CC_MD5` 在 iOS 13+ 废弃。**修复**: `import CryptoKit`，使用 `Insecure.MD5.hash(data:)` 计算 MD5，无需手动管理 buffer
40. **SPM 模块拆分验证流程** — 拆分 SPM 模块后，先用 `swiftc -parse` 做语法检查，再在 Xcode 中 ⌘B 验证编译。**不要**用 `swift build` 编译 iOS-only 包（会报 macOS 平台不兼容）。用户偏好**分阶段验证**：先验证新模块能编译，再迁移业务层，最后清理旧模块
41. **@_exported import 的 shared 实例命名** — 当业务层模块使用 `@_exported import NetworkCore` 时，消费者应调用 `NetworkCore.shared`（通用库的入口），而非 `ServiceType.shared`（业务层 enum 没有 shared 属性）。批量迁移时用 `sed -i '' 's/ServiceType\.shared/NetworkCore.shared/g'` 替换
42. **API Target 枚举参数完整性** — 调用 API Target 时必须提供所有参数（如 `.getRoomList(page: 1, pageSize: 20)`），不能省略。如果旧 API 有默认值而新 Target 没有，需要在调用处显式传入
43. **ObjC 模块移除时的类型迁移** — 移除 ObjC Pod 模块（如 DomainNetwork/ServiceLayer）前，需将业务层依赖的 Model 类型（如 `User.swift`、`UserManager.swift`）复制到 Swift 项目目录，并在 Xcode 中手动添加到 target。否则会报 `Cannot find 'Xxx' in scope`
44. **delegate_task 大批量迁移超时** — Claude Code 委托 49+ 文件迁移会在 10 分钟超时。**修复**: 分批委托（每批 10-15 个文件），或用 `execute_code` 脚本做批量 import 替换，再委托 Claude Code 处理复杂的 API 调用迁移
45. **DomainNetworkCore.multipartFormRequest 替换** — 旧代码使用 `DomainNetworkCore.multipartFormRequest`（如 crash reporting）不应迁移到新网络层，应改用 `URLSession` 直接实现。Crash reporting 应独立于 App 网络层，避免网络层未初始化时无法上报
46. **JSONResponse 类型安全陷阱** — `NetworkTargetType` 的 `associatedtype Response: SmartDecodable` 设计初衷是编译时绑定响应类型，但如果所有 Target 都用 `JSONResponse`（即 `[String: Any]`），类型安全完全失效。**修复**: 为高频 API 定义专用 Response Model，至少登录/用户信息/设备列表应优先迁移
47. **HTTPSession.makeFromEnvironment() 未被调用** — `HTTPSession.makeDefault()` 创建了带 `RetryPolicy` 的 Session，但 `NetworkCore.shared` 使用的是 `Session.default`，重试逻辑完全无效。**修复**: 在 `NetworkCore.shared` 初始化中使用 `HTTPSession.makeFromEnvironment()` 创建 Session
48. **@MainActor 隔离路径不一致** — callback 路径用 `DispatchQueue.main.async { Task { @MainActor in ... } }`，async 路径直接 `await authPlugin.refreshTokenIfNeeded()`。两种路径的 actor 上下文切换方式不同。**修复**: 统一使用 `await MainActor.run { }` 替代 `DispatchQueue.main.async`
49. **BusinessTarget 必须包含完整 headers** — 旧 ServiceLayerAPI 的 headers 包含 Timestamp/Token/Customer/Language，新 BusinessTarget 必须保持一致。如果 headers 缺少字段，服务器返回 "请求参数异常" (code 20101)。**修复**: headers 从 `UserManager.shared.user?.token` 或 `NetworkCoreConfiguration.shared.mergedHeaders(for: .bizAPI)` 获取 token
50. **Swift extension 中 Self 引用其他 extension 的方法** — `extension NSObject` 中的方法不能用 `Self.methodName()` 调用 `extension UIViewController` 中的静态方法。**修复**: 将被调用的方法移到调用方所在的 extension 中，或使用具体类型名 `NSObject.methodName()`
51. **ObjC 模块移除前的类型迁移清单** — 移除 ObjC Pod 模块前，需要迁移的类型：(a) Model 类（User.swift、UserTag.swift）→ 复制到 Swift 项目目录；(b) 单例管理类（UserManager.swift）→ 复制并去掉 ObjC 依赖；(c) 错误码映射（checkErroList）→ 移到 extension 中；(d) 服务器地址配置 → 移到 Bootstrap。**关键**: 复制后必须在 Xcode 中手动添加到 target
52. **Crash reporting 独立于主网络层** — 旧代码使用 `DomainNetworkCore.multipartFormRequest` 上传 crash 日志，不应迁移到新网络层。**修复**: 改用 `URLSession.uploadTask` 直接实现，避免网络层未初始化时无法上报
53. **delegate_task 大批量迁移策略** — 委托 49+ 文件迁移会在 10 分钟超时。**修复**: (a) 用 `execute_code` 做批量 import 替换（sed 命令）；(b) 分批委托（每批 10-15 个文件）；(c) 复杂的 API 调用迁移留给 Claude Code 处理
54. **API path/parameters 必须逐个对照旧代码** — 迁移时不能只看 API 名称推断路径，必须逐个对比旧代码的 `referenceURL`、参数名、参数类型、HTTP method。常见差异：(a) path 前缀（`/api/v2/` vs `/api/v1/`）；(b) 参数名（`serial` vs `serialNo`，`File` vs `file`）；(c) HTTP method（heartbeat GET→POST）；(d) 缺少必填参数（OTA 缺 uid，Live 缺 username）
55. **Glasses chat/upload/translate 走 thirdParty 服务** — 旧代码中 chat、upload、translate 等 API 使用 `Configuration.thirdPartyServer()`（thirdparty.example.com），不是 cloudAPI。**修复**: 创建 `ThirdPartyTarget` 协议，绑定 `.thirdParty` 服务
56. **App 层注入 uid/username 到 NetworkCoreConfiguration** — ServiceType 模块无法访问 App 层的 `UserManager`。**修复**: 在 `NetworkCoreConfiguration` 添加 `currentUid`/`currentUsername` 属性，在 `NetworkCoreBootstrap.configure()` 中注入 `NetworkCoreConfiguration.shared.currentUid = "\(UserManager.shared.user?.userId ?? 0)"`
57. **OTA 参数名 serial vs serialNo** — 旧代码使用 `serial`，新代码可能用 `serialNo`。服务器可能只接受其中一个。**修复**: 对照旧代码确认参数名，OTA getLatestFirmwareVersion 应使用 `"serial": serialNo`
58. **Device API path 修正** — 旧代码路径：`/api/v2/terminals/check`（getDeviceIdentifier）、`/api/v2/terminals/bind`、`/api/v2/terminals/unbind`、`/api/v2/terminals/list`、`/api/v2/terminals/detail`、`/api/v2/terminals/name`（rename）、`/api/v2/terminals/typeList`（requestDeviceTypeList）、`/api/v2/operation/list`（requestGuideList）。不要用 `/api/v2/device/*` 路径
59. **sendRequest 返回契约必须与旧栈一致（根因级 Pitfall）** — 旧栈 `processResult` 成功时返回**完整 JSON body**，App 层按 `dict["code"]`、`dict["data"]`、`dict["msg"]` 解析。如果新栈只返回 `data` 字段，会导致：(a) `UserManager.login(info:)` 解析失败（期望 `info["data"]["user"]`，实际已是内层 data）；(b) `checkError(dic:)` 无法读取 `dic["code"]`，业务错误被跳过；(c) `heartbeat` 互踢检测失效（无 code）。**修复**: `sendRequest` 成功时返回完整 `response.json`，success 判断交给 App 层（旧栈语义：HTTP 成功即 true）。**这是迁移中影响面最大的 Pitfall，一次修复登录、设备列表、发现页、OTA、错误处理等大量调用点**
60. **Cloud API 系统性缺 uid** — 旧栈几乎所有 cloud 请求带 uid 参数，迁移时容易遗漏。缺少 uid 会导致：(a) 服务器返回参数缺失；(b) Sign 签名与服务端不一致（参数参与签名计算）。**修复**: 在 `NetworkCoreConfiguration` 添加 `currentUid`/`currentUsername`，Bootstrap 时注入，各 Target 的 `apiParameters` 中统一添加 `let uid = NetworkCoreConfiguration.shared.currentUid ?? "0"`
61. **Multipart 上传必须用 upload() 不是 sendRequest()** — `sendRequest` 走 JSON POST，文件不会上传。Multipart 上传必须：(a) Target 遵循 `MultipartTarget` 协议；(b) 实现 `buildMultipartFormData(_:)`；(c) 调用 `NetworkCore.shared.upload(target)` 而非 `sendRequest`
62. **enableTokenRefreshRetry 数据竞争** — `NetworkClient.enableTokenRefreshRetry` 是可变公开属性，无同步保护。`send` 方法在回调线程读取，调用方可在任意线程写入。**修复**: 改为 `@Atomic` 或 `let`
63. **闭包版 vs async 版 send 回调线程不一致** — 闭包版 `send` 保证主线程回调（`DispatchQueue.main.async`），async 版 `send` 的 continuation 在 `networkClient` 回调线程 resume，不保证主线程。**修复**: async 版 `performSend` 中用 `@MainActor` 确保恢复到主线程
64. **token 刷新重试跳过所有插件** — `send` 方法 token 刷新重试时传 `plugins: []`，跳过所有插件（包括 logging、analytics、certificate pinning）。如果某些插件对请求正确性有影响（如添加签名 header），重试请求可能失败。**修复**: 区分"一次性插件"和"始终需要的插件"