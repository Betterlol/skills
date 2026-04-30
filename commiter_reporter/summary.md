# 📄 工程系统性项目阶段管理协议（v3.0 - Practical & Agent-Friendly）

---

## 一、设计定位（最终版）

本协议是：

> 👉 **Agent-Oriented Version Control Protocol（面向 Agent 的版本控制协议）**

用于：

* AI Coding Agent
* 自动化工程流程
* 多阶段工程项目管理

---

## 二、设计目标

1. **阶段隔离（Stage Isolation）**
2. **变更可追溯（Traceability）**
3. **可执行性（Agent-friendly）**
4. **与 Git 协同（Git as Source of Truth）**
5. **容错与恢复能力（Fault Tolerance）**
6. **低运行成本（Practical Execution）**

---

## 三、核心模型

### 1. 双轨制（核心）

```text
Agent记录（Intent Layer） = 文件集合（file_set）
Git状态（Source of Truth） = 实际仓库状态
```

规则：

* Agent 决定“哪些文件属于本阶段”
* Git 用于校验与提交

---

### 2. 阶段上下文（Stage Context）

每个阶段必须绑定：

```text
STAGE_ID      = UUID
BASE_COMMIT   = 阶段开始时的 commit
FILE_SET      = 本阶段文件集合
```

---

### 3. 文件集合（File Set）

```text
STEP_NEW_FILES
STEP_MODIFIED_FILES
STEP_DELETED_FILES
```

---

### 4. 文件集合锁（新增）

一旦文件进入 FILE_SET：

* 不得隐式移除
* 不得自动扩展
* 所有提交必须来自该集合

👉 防止阶段污染

---

## 四、核心原则（v3 精简版）

### 1. Git 协同原则

* Git 是唯一真实状态
* Agent 不得假设仓库状态
* 必须在关键节点校验

---

### 2. 最小提交原则

只允许提交：

* FILE_SET 中的文件
* summary 文件

---

### 3. 用户改动保护原则（优化）

阶段开始时存在未提交改动：

默认行为：

```text
不自动纳入
不自动提交
保留在工作区
```

例外：

* 若 agent 能明确判断“高度相关” → 可纳入

仅在冲突时才请求用户输入

---

### 4. diff 隔离原则

diff 必须：

* 基于 BASE_COMMIT
* 仅针对 FILE_SET 文件

```bash
git diff BASE_COMMIT -- <file>
```

---

### 5. 阶段隔离原则

* 一个阶段 = 一个 commit
* 不允许跨阶段提交

---

### 6. 校验策略（优化）

仅在以下时机执行 git 校验：

```text
1. PRE_STAGE_CHECK
2. STAGE_VALIDATE
3. ERROR
```

👉 不要求每次文件操作后执行

---

### 7. 一致性定义（放宽）

允许差异：

* gitignore 文件
* 编译产物
* 缓存文件

要求：

👉 源码文件一致

---

## 五、阶段状态机（v3）

### 状态定义

```text
IDLE
PRE_STAGE_CHECK
STAGE_INIT
STAGE_ACTIVE
STAGE_VALIDATE
STAGE_SUMMARY
STAGE_COMMIT
COMPLETED

ERROR
RECOVERY
```

---

### 状态流转

```text
IDLE
  ↓
PRE_STAGE_CHECK
  ↓
STAGE_INIT
  ↓
STAGE_ACTIVE
  ↓
STAGE_VALIDATE
  ↓
STAGE_SUMMARY
  ↓
STAGE_COMMIT
  ↓
COMPLETED

异常路径：
ANY → ERROR → RECOVERY
```

---

## 六、执行流程

### 1. PRE_STAGE_CHECK

执行：

```bash
git status
```

处理未提交改动：

| 情况   | 行为          |
| ---- | ----------- |
| 明确相关 | 纳入 FILE_SET |
| 明确无关 | 忽略          |
| 不确定  | 忽略          |

---

### 2. STAGE_INIT

```bash
BASE_COMMIT=$(git rev-parse HEAD)
```

初始化：

```text
STAGE_ID
FILE_SET = 空
```

---

### 3. STAGE_ACTIVE

Agent 执行任务，并维护：

```text
NEW / MODIFIED / DELETED
```

规则：

* 每次文件操作必须更新 FILE_SET
* 不允许隐式新增文件

---

### 4. STAGE_VALIDATE（关键）

执行：

```bash
git status
git diff
```

校验：

```text
FILE_SET vs Git 实际变更
```

要求：

* 源码文件一致
* 否则进入 ERROR

---

### 5. STAGE_SUMMARY（优化版）

生成：

```text
step_<stage_id>_<description>.md
```

内容：

```text
1. 阶段描述
2. BASE_COMMIT
3. 变更文件列表
4. 关键差异摘要（非完整 diff）
5. 风险/注意事项
```

👉 不再强制完整代码（避免冗余）

---

### 6. STAGE_COMMIT

```bash
git add <FILE_SET>
git add summary.md
git commit -m "[STAGE_ID] step: <description>"
```

禁止：

```bash
git add .
git commit -a
```

---

### 7. ERROR

触发：

* file_set 不一致
* git 操作失败

---

### 8. RECOVERY

策略：

```text
1. 同步 FILE_SET
2. 或回滚 BASE_COMMIT
3. 或请求用户介入
```

---

## 七、输出要求

阶段完成后输出：

```text
1. summary 文件路径
2. commit hash
3. 阶段概要说明
```

---

## 八、核心规则（给 Agent）

```text
1. commit 是阶段边界
2. FILE_SET 由 agent 显式维护
3. Git 负责校验与提交
4. 不得提交未确认用户改动
5. 不得使用 git add .
6. diff 必须基于 BASE_COMMIT
7. 每阶段必须独立 commit
```

---