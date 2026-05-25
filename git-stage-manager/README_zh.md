# Git Stage Manager

**面向 AI 智能体的版本控制协议** —— 一套严谨的、状态机驱动的协议，让 AI 编码智能体能够安全、可追溯地完成单阶段 Git 提交。

## 概述

Git Stage Manager 解决了一个核心问题：当 AI 智能体修改代码后，如何确保它只提交自己改动的文件——不多不少——同时保护用户已有的未提交工作不被误伤？

它定义了一种 **双轨模型**：

- **智能体意图层**（`FILE_SET` + `ACTION_LOG` + `STAGE_CONTEXT`）—— 智能体声明它动了哪些文件以及为什么
- **Git 事实源层**（Source of Truth）—— 每一步都通过实际仓库状态进行验证

## 核心概念

| 概念 | 说明 |
|---|---|
| **阶段隔离** | 一个阶段 = 一个提交，杜绝跨阶段代码泄漏 |
| **FILE_SET** | 显式声明本阶段新增、修改、删除的文件列表 |
| **ACTION_LOG** | 记录每个文件操作及原因，与 FILE_SET 一一对应 |
| **证据驱动摘要** | 摘要文件内容必须来自真实的 `git diff`、`cat` 输出，而非智能体记忆 |
| **用户变更保护** | 已有的未提交改动永远不会被自动纳入 |

## 状态机

```
IDLE → PRE_STAGE_CHECK → STAGE_INIT → STAGE_ACTIVE → STAGE_VALIDATE
  → STAGE_SUMMARY → SUMMARY_VALIDATE → STAGE_COMMIT → COMPLETED
                                                      ↓
                                              ERROR / RECOVERY
```

每个状态都有预定义的校验检查和允许的 Git 命令，确保执行的确定性和可审计性。

## 允许的 Git 命令

仅允许以下命令：

- `git status`
- `git diff`（相对于 `BASE_COMMIT`）
- `git rev-parse HEAD`
- `git add <file>`（仅显式指定的文件）
- `git commit -m "<message>"`
- `git log -1 --format=%H`

禁止使用：`git add .`、`git commit -a`、`git push`、`git reset --hard`、`git stash`。

## 执行流程

1. **PRE_STAGE_CHECK** — 检查是否存在冲突，快照已有变更
2. **STAGE_INIT** — 记录 `BASE_COMMIT`，初始化空的 `FILE_SET` 和 `ACTION_LOG`
3. **STAGE_ACTIVE** — 执行代码修改，填充 `FILE_SET` 和 `ACTION_LOG`
4. **STAGE_VALIDATE** — 断言 `FILE_SET` 与实际 Git 变更完全一致
5. **STAGE_SUMMARY** — 基于真实证据生成摘要文件
6. **SUMMARY_VALIDATE** — 验证摘要的完整性和正确性
7. **STAGE_COMMIT** — 暂存声明的文件 + 摘要文件，创建提交
8. **COMPLETED** — 输出提交哈希和摘要文件路径

## 提交格式

```
[<STAGE_ID>][<STAGE_TYPE>] <description>
```

## 使用方式

这是一个为 AI 编码智能体设计的 **skill**（技能包），适用于 Claude、Codex、Gemini、OpenCode 等工具。在对工程某个阶段进行可控的单次提交时加载此技能即可。

## 文件说明

| 文件 | 说明 |
|---|---|
| `SKILL.md` | 正式协议规范（英文，398 行） |
| `summary.md` | 设计文档 v4.0（中文，493 行） |

## 协议

MIT
