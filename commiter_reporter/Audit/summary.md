# 📄 工程系统性项目阶段管理协议（v4.0 - Evidence-Driven & Executable）

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
7. **证据驱动（Evidence-Driven Execution）**

---

## 三、核心模型

### 1. 双轨制（核心）

```text
Agent记录（Intent Layer） = FILE_SET + ACTION_LOG + STAGE_CONTEXT
Git状态（Source of Truth） = 仓库真实状态
```

---

### 2. 阶段上下文（Stage Context）

```text
STAGE_ID
BASE_COMMIT
STAGE_TYPE
FILE_SET
ACTION_LOG
SUMMARY_FILE
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
ADD_TO_FILE_SET(file, action, reason)
```

必须满足：

* 当前阶段直接操作
  或
* 用户显式要求

禁止：

* 基于 git status 自动纳入
* 基于语义猜测纳入
* 纳入无关用户改动

---

### 5. ACTION_LOG

```text
ACTION_LOG = [
  {
    file: "a.py",
    action: "modify",
    reason: "fix parser bug"
  }
]
```

作用：

* 行为解释（Explainability）
* summary 生成依据
* 调试 agent 行为

---

### 6. SUMMARY_FILE（新增核心对象）

```text
step_<stage_id>_<description>.md
```

特性：

* 不属于 FILE_SET
* 不参与阶段逻辑
* 仅作为**输出与审计载体**

---

## 四、核心原则（v4.0）

### 1. Git 协同原则

* Git 是唯一真实状态
* Agent 不得假设仓库状态
* 所有验证基于 Git

---

### 2. 最小提交原则

仅提交：

* FILE_SET 中的文件
* SUMMARY_FILE

---

### 3. 用户改动保护原则（严格）

阶段开始存在未提交改动：

```text
不纳入
不提交
不修改
```

仅允许：

```text
1. 本阶段直接修改
2. 用户显式要求
```

禁止：

👉 基于“相关性”自动纳入

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

### 7. 证据生成原则（核心升级）

```text
Summary 中的内容必须来自真实系统状态
```

要求：

* 必须通过工具 / 命令 / 脚本生成
* 必须反映文件系统或 Git 实际状态

允许方式：

```text
- shell 命令（cat, git diff 等）
- 自定义脚本
- 文件读取工具
```

禁止：

```text
- 手动编写代码内容
- 从记忆重构文件
- 摘要代替原始内容
```

👉 Summary = Evidence，而非描述

---

### 8. Summary 隔离原则（新增）

```text
SUMMARY_FILE 不属于 FILE_SET
```

规则：

* 不参与 diff
* 不参与 FILE_SET
* 禁止递归包含

---

### 9. Summary 一致性原则（新增）

必须满足：

```text
1. Summary 中所有文件 ∈ FILE_SET
2. 所有 diff 必须基于 BASE_COMMIT
3. 不允许全仓 diff
4. 不允许包含无关文件
```

---

### 10. 校验策略（分层）

执行时机：

```text
1. PRE_STAGE_CHECK
2. STAGE_VALIDATE
3. SUMMARY_VALIDATE
4. ERROR
```

优化：

* commit 前仅做轻量校验（git status）
* 避免重复高成本 diff

---

## 五、阶段状态机

```text
IDLE
PRE_STAGE_CHECK
STAGE_INIT
STAGE_ACTIVE
STAGE_VALIDATE
STAGE_SUMMARY
SUMMARY_VALIDATE
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

* 检测未提交改动
* 不自动纳入
* 冲突需用户确认

---

### 2. STAGE_INIT

```bash
git rev-parse HEAD
```

初始化：

```text
STAGE_ID
BASE_COMMIT
FILE_SET = 空
ACTION_LOG = []
SUMMARY_FILE = unset
```

---

### 3. STAGE_ACTIVE

规则：

* 每次文件操作必须：

```text
1. 更新 FILE_SET
2. 写入 ACTION_LOG
```

---

### 4. STAGE_VALIDATE

执行：

```bash
git status
git diff
```

必须满足：

```text
1. FILE_SET ⊆ git changes
2. git changes ⊆ FILE_SET
3. 每个 FILE_SET 文件有 ACTION_LOG
4. 无未记录源码改动
5. 无未跟踪源码文件（除非忽略）
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

内容结构：

```text
1. 阶段描述
2. STAGE 信息
3. 新文件列表
4. 新文件完整内容（必须真实读取）
5. 修改文件列表
6. 修改 diff（基于 BASE_COMMIT）
7. 删除文件（可选）
8. ACTION_LOG
9. 风险说明
```

---

### 6. SUMMARY_VALIDATE

验证：

```text
1. 所有新文件包含完整内容
2. 所有修改文件包含 diff
3. diff 基于 BASE_COMMIT
4. 所有文件属于 FILE_SET
5. 无全仓 diff
6. 无递归 summary
```

否则：

```text
→ 修复 summary
```

---

### 7. STAGE_COMMIT

执行：

```bash
git status
git add <FILE_SET>
git add <SUMMARY_FILE>
git commit -m "[STAGE_ID][TYPE] <description>"
```

禁止：

```bash
git add .
git commit -a
```

---

### 8. ERROR

触发：

* FILE_SET 不一致
* Git 状态异常
* summary 不合法

---

### 9. RECOVERY

策略：

```text
1. 修复 FILE_SET
2. 重新校验
3. 请求用户介入
4. 必要时安全中止
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
7. Summary 必须由证据生成
8. validate 必须严格通过
```

---

## 九、设计总结

本协议实现：

👉 Agent 可控执行
👉 Git 强一致校验
👉 Evidence 驱动输出
👉 可审计工程流程

本质定位：

👉 **Agent Runtime Control Layer（带证据约束的执行层）**
