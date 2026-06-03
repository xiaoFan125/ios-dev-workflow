# GitHub 仓库隐私清理指南

## 核心原则

发布本地项目/代码到公开 GitHub 仓库前，必须**多轮扫描**。一次扫描不够。

## 清理清单（按轮次）

### 第 1 轮：个人路径

```bash
# 搜索
grep -rn "~\|/Users/developer\|~/Projects" --include="*.swift" --include="*.md" .

# 替换
sed -i '' 's|~/Projects/YourProject|~/Projects/YourProject|g' file.swift
```

### 第 2 轮：项目名/品牌名

```bash
# 搜索（项目特定名称）
grep -rn "SmartGlasses\|SmartEarbuds\|BrandA\|BrandB\|YourProject" --include="*.swift" --include="*.md" .

# 替换
sed -i '' 's|SmartGlasses|SmartGlasses|g' *.md
sed -i '' 's|SmartEarbuds|SmartEarbuds|g' *.md
```

### 第 3 轮：文件名中的项目名

```bash
# 查找需要重命名的文件
find . -name "*SmartGlasses*" -o -name "*BrandC*" -o -name "*mojie*"

# 重命名
mv references/mbglasses-voice-chat-architecture.md references/voice-chat-architecture.md
# 更新所有引用
sed -i '' 's|mbglasses-voice-chat-architecture.md|voice-chat-architecture.md|g' README.md SKILL.md
```

### 第 4 轮：版本历史/提交信息

- 版本历史中的本地工作细节（如"BLE SDK 自动重连抑制"这类具体修复）
- 提交信息中引用其他仓库名

### 第 5 轮：references 目录

references 中的 `.md` 文件经常包含项目特定内容：
- 项目路径
- 项目架构细节
- 品牌名

## 验证方法

```bash
# 最终验证：搜索所有敏感关键词
grep -rn "developer\|xiaoFan\|SmartGlasses\|BrandC\|BrandA\|BrandB\|Desktop/项目" . \
  --include="*.swift" --include="*.md" --include="*.json" \
  --exclude-dir=".git" --exclude-dir=".build"

# 应该返回 0 结果
```

## 常见遗漏

1. **README.md 文件结构表** — 文件名可能包含项目名
2. **README.md 参考文档列表** — 描述中可能包含项目名
3. **SKILL.md 版本历史** — 更新内容可能包含具体项目细节
4. **git log 中的提交信息** — 如果删除旧仓库重新创建，提交历史会被清除
5. **CLAUDE.md / .cursorrules** — 项目根目录的开发配置文件
6. **提交信息中引用其他仓库** — 如 "参考 cybernetics-framework 格式"
7. **版本历史中的本地工作内容** — 如具体的 BLE 修复、SDK 版本号等

## 当隐私问题太多时

如果多轮扫描后仍有遗漏，**删除远程仓库重新创建**比逐个修复更快：

```bash
# 1. 删除远程仓库（需要 delete_repo scope）
gh auth refresh -h github.com -s delete_repo  # 浏览器授权
gh repo delete owner/repo --yes

# 2. 本地重新初始化
rm -rf .git
git init && git checkout -b main

# 3. 重新创建远程仓库
gh repo create repo-name --public --source . --remote origin --push --description "..."
```

**注意**: CLI 中 device flow 会超时，让用户在浏览器中完成授权。

## 流程

```
扫描 → 替换 → 提交 → 推送 → 用户检查 → 发现遗漏 → 再扫描 → 再替换 → 再推送
       ↑                                                              |
       └──────────────────────────────────────────────────────────────┘
```

**关键教训**: 用户一定会发现你遗漏的地方。不要期望一次完成。
