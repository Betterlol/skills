# 📄 工程系统性项目阶段管理协议（v3.1 - Executable & Robust）

---

## 一、设计定位

本协议是：

> 👉 **Agent-Oriented Version Control Protocol（Agent 运行控制层）**

用于：

* AI Coding Agent
* 自动化工程流程
* 多阶段工程项目管理

---

## 二、设计目标

1. 阶段隔离（Stage Isolation）
2. 变更可追溯（Traceability）
3. 可执行性（Agent-friendly）
4. 与 Git 协同（Git as Source of Truth）
5. 容错与恢复能力（Fault Tolerance）
6. 稳定性优先（Deterministic > Smart）

---

## 三、核心模型

### 1. 双轨制（核心）

```text
Agent记录（Intent Layer） = FILE_SET + ACTION_LOG
Git状态（Source of Truth） = 仓库真实状态
```

---

### 2. 阶段上下文（Stage Context）

```text
STAGE_ID      = UUID
BASE_COMMIT   = 阶段开始时 commit
FILE_SET      = 本阶段文件集合
ACTION_LOG    = 操作日志
STAGE_TYPE    = feature | fix | refactor | chore
```

---

### 3. 文件集合（FILE_SET）

```text
STEP_NEW_FILES
STEP_MODIFIED_FILES
STEP_DELETED_FILES
```

---

### 4. FILE_SET 机制（弱锁）

规则：

* 默认锁定（不可隐式变更）
* 允许**显式扩展**（必须记录）

```text
ADD_TO_FILE_SET(file, reason)
```

必须满足：

* 当前阶段直接操作
  或
* 明确依赖关系（如重构传播）

---

### 5. ACTION_LOG（新增）

```text
ACTION_LOG = [
  {
    file: "a.py",
    action: "modify",
    reason: "fix bug in parser"
  }
]
```

作用：

* 可解释性
* 自动生成 commit message
* 调试 agent 行为

---

## 四、核心原则（v3.1）

### 1. Git 协同原则

* Git 是唯一真实状态
* Agent 不得假设仓库状态
* 仅在关键节点校验

---

### 2. 最小提交原则

仅提交：

* FILE_SET 中的文件
* summary 文件

---

### 3. 用户改动保护原则（严格）

阶段开始存在未提交改动：

默认：

```text
不纳入
不提交
不修改
```

仅允许纳入条件：

```text
1. 本阶段直接修改该文件
2. 用户显式要求
```

👉 不允许基于“语义判断相关性”

---

### 4. FILE_SET 控制原则

禁止：

* 隐式新增文件
* 未记录修改

允许：

* 显式扩展（必须记录 ACTION_LOG）

---

### 5. diff 隔离原则

```bash
git diff BASE_COMMIT -- <file>
```

仅允许 FILE_SET 文件参与

---

### 6. 阶段隔离原则

* 一个阶段 = 一个 commit
* 不允许跨阶段提交

---

### 7. 校验策略（优化）

仅在以下时机执行：

```text
1. PRE_STAGE_CHECK
2. STAGE_VALIDATE
3. ERROR
```

---

## 五、阶段状态机

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

## 六、执行流程（关键）

---

### 1. PRE_STAGE_CHECK

```bash
git status
```

处理：

* 不自动纳入任何已有改动

---

### 2. STAGE_INIT

```bash
BASE_COMMIT=$(git rev-parse HEAD)
```

初始化：

```text
STAGE_ID
FILE_SET = 空
ACTION_LOG = []
```

---

### 3. STAGE_ACTIVE

规则：

* 每次文件操作必须：

```text
1. 更新 FILE_SET
2. 写入 ACTION_LOG
```

示例：

```text
ADD_TO_FILE_SET("a.py", "modify parser logic")
```

---

### 4. STAGE_VALIDATE（强化版）

执行：

```bash
git status
git diff
```

必须满足：

```text
1. FILE_SET ⊆ git changes
2. git changes ⊆ FILE_SET（源码文件）
3. 无未记录源码改动
4. 无未跟踪源码文件（除非忽略）
```

否则：

```text
→ ERROR
```

---

### 5. STAGE_SUMMARY

生成：

```text
step_<stage_id>_<description>.md
```

内容：

```text
1. 阶段描述
2. STAGE_TYPE
3. BASE_COMMIT
4. FILE_SET
5. ACTION_LOG（简要）
6. 关键差异摘要
7. 风险说明
```

---

### 6. STAGE_COMMIT

```bash
git add <FILE_SET>
git add summary.md
git commit -m "[STAGE_ID][TYPE] <description>"
```

禁止：

```bash
git add .
git commit -a
```

---

### 7. ERROR

触发：

* FILE_SET 不一致
* git 状态异常

---

### 8. RECOVERY

策略：

```text
1. 同步 FILE_SET（优先）
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
2. FILE_SET 必须显式维护
3. ACTION_LOG 必须记录
4. Git 是唯一真实状态
5. 不得提交用户未确认改动
6. 不得使用 git add .
7. validate 必须严格通过
```

---

## 九、设计总结

本协议实现：

👉 Agent 可控执行
👉 Git 强一致校验
👉 可恢复工程流程

本质定位：

👉 **Agent Runtime Control Layer（代理运行控制层）**
