# iOS 开发工作流

Hermes 调度 Claude Code（深度实现）+ Codex（验证/备选）的 iOS 开发范式，遵循控制论六原则。

## 适用场景

- iOS App 的架构设计、编码实现、代码审查
- 需要「AI 辅助 + 人工决策」的开发流程
- Swift 5.9+ / SwiftUI / MVVM 项目
- 团队协作的开发规范和最佳实践

## 核心原理

| # | 原则 | 一句话 |
|---|------|--------|
| 1 | 反馈闭环 | 每个动作必须有验证环节，形成"执行→检测→修正"的闭环 |
| 2 | 系统稳定性 | 稳定优先于性能，宁可慢也不引入不可控风险 |
| 3 | 信息流 | memory 存底层逻辑（信号），skill 存操作流程（通道） |
| 4 | 自调节 | 遇到意外结果时停下来分析根因，不机械重试 |
| 5 | 层次结构 | 战略层（架构）→战术层（模块）→执行层（代码） |
| 6 | 黑箱方法 | 关注输入输出而非内部实现，用可观测结果验证不可观测过程 |

## 三工具分工

| 角色 | 定位 | 职责 |
|------|------|------|
| **Hermes** | 总控调度 | 拆分需求、生成任务说明、分发到 Claude Code/Codex、汇总结果、验收 |
| **Claude Code** | 主执行线 | 深度项目阅读、主线实现、跨文件重构、复杂依赖排查 |
| **Codex** | 第二执行线 | 方案验证、备选实现、批量命令、关键逻辑复核 |

## 工作流

```
Phase 0: 需求拆分 (Hermes)
├── 分析任务复杂度
├── 判断是否需要双执行线
└── 生成结构化任务说明

Phase 1: 深度实现 (Claude Code)
├── 深度阅读项目结构
├── 按任务说明实施变更
└── 生成变更摘要

Phase 2: 验证与备选 (Codex)
├── 对实现进行逻辑复核
├── 生成备选方案（如果需要）
└── 输出验证报告

Phase 3: 汇总验收 (Hermes)
├── 收集两边结果
├── 对比思路、评估风险
└── 生成最终报告

Phase 4: 迭代优化 (Hermes)
├── 修复 → 验证 → 文档
└── 经验提炼（memory/skill 分层）
```

## 快速使用

### Hermes Agent 用户

将 `SKILL.md` 复制到你的 skills 目录：

```bash
mkdir -p ~/.hermes/skills/software-development/ios-dev-workflow
cp SKILL.md ~/.hermes/skills/software-development/ios-dev-workflow/
```

### 其他 AI Agent

直接参考 `SKILL.md` 中的方法论，适配你自己的工具链。

### 前置条件

- macOS + Xcode 16+
- Claude Code CLI（主执行线：深度实现）
- Codex CLI（第二执行线：验证/备选）
- Swift 5.9+

## 文件结构

```
.
├── README.md          ← 你正在看的这个
├── SKILL.md           ← 完整工作流文档（Hermes skill 格式）
└── references/        ← 参考文档
    ├── networkkit-architecture-and-review.md
    ├── network-layer-migration-afnetworking-to-alamofire.md
    ├── github-privacy-sanitization.md
    └── ...
```

## License

MIT
