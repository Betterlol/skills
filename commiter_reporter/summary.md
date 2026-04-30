# 📄 工程系统性项目阶段管理协议（v2.0 - Executable）

---

## 一、设计目标（升级版）

在 v1 的基础上，v2 目标从“规范行为”升级为“可执行系统”：

1. **阶段隔离（Stage Isolation）**
2. **变更可追溯（Traceability）**
3. **Agent 可控（Deterministic Behavior）**
4. **与 Git 协同（Git as Source of Truth）**
5. **容错与恢复能力（Fault Tolerance）**

---

## 二、核心设计变更（相对于 v1）

### 1. 双轨制（关键）

```text
Agent记录 = 意图（Intent Layer）
Git状态 = 真相（Source of Truth）
```

规则：

* Agent **必须维护文件集合（意图）**
* Git **用于校验和对齐真实状态**

允许：

```bash
git status   # 用于校验
git diff     # 用于验证差异
```

禁止：

```bash
git add .
git commit -a
```

---

### 2. 阶段上下文（Stage Context）

每个阶段必须绑定唯一上下文：

```text
STAGE_ID       = UUID
BASE_COMMIT    = 当前阶段开始时的 commit hash
```

所有操作必须绑定：

```text
(stage_id, base_commit, file_set)
```

---

### 3. 文件集合定义（仍保留，但增强）

```text
STEP_NEW_FILES
STEP_MODIFIED_FILES
STEP_DELETED_FILES
```

新增约束：

* 每次文件操作后：

  * 必须更新集合
  * 必须通过 git 校验一致性

---

### 4. 未提交改动处理（改为显式决策）

阶段开始时：

```text
检测到未提交改动：

1. 纳入本阶段
2. 暂存（stash）
3. 忽略

需要用户选择
```

默认行为：

👉 不自动纳入

---

## 三、核心原则（更新版）

### 1. Git 协同原则

* Git 是唯一真实状态来源
* Agent 不得假设 repo 状态
* 所有提交前必须校验：

```bash
git status
git diff
```

---

### 2. 提交最小化原则

只允许提交：

* 本阶段文件集合
* 阶段 summary 文件

---

### 3. 双重校验原则（新增）

在 commit 前执行：

```text
Agent记录文件集合
vs
git status 实际变更
```

若不一致：

👉 必须修正后才能 commit

---

### 4. 用户改动保护原则（强化）

* 未确认改动：

  * 默认不提交
* 必须显式确认才允许纳入

---

### 5. 阶段强隔离原则

* 一个阶段 = 一个 commit
* 不允许跨阶段合并

---

## 四、阶段状态机（v2）

### 状态定义

```text
IDLE
PRE_STAGE_CHECK
STAGE_INIT          新增
STAGE_ACTIVE
STAGE_VALIDATE      新增（关键）
STAGE_SUMMARY
STAGE_COMMIT
COMPLETED

ERROR               新增
RECOVERY            新增
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
任意阶段 → ERROR → RECOVERY
```

---

## 五、阶段执行细节

### 1. PRE_STAGE_CHECK

执行：

```bash
git status
```

行为：

* 检测未提交改动
* 触发用户决策（纳入 / stash / 忽略）

---

### 2. STAGE_INIT（新增）

初始化：

```text
STAGE_ID
BASE_COMMIT = git rev-parse HEAD
FILE_SET = 空集合
```

---

### 3. STAGE_ACTIVE

Agent 执行任务，并维护：

```text
NEW / MODIFIED / DELETED
```

每次文件操作后必须：

```bash
git status
```

用于校验一致性

---

### 4. STAGE_VALIDATE（关键新增）

执行：

```bash
git status
git diff
```

校验：

```text
Agent记录文件集合
vs
Git实际变更
```

要求：

* 必须完全一致
* 否则进入 ERROR

---

### 5. STAGE_SUMMARY（优化版）

生成文件：

```text
stepX_Y_description.md
```

内容结构：

```text
1. 阶段描述
2. BASE_COMMIT
3. 变更文件列表
4. 关键差异摘要（非完整 diff）
5. 风险或注意事项
```

👉 不再要求完整文件内容（避免冗余）

---

### 6. STAGE_COMMIT

执行：

```bash
git add <file_set>
git add summary.md
git commit -m "[STAGE_ID] stepX_Y: 描述"
```

禁止：

```bash
git add .
```

---

### 7. ERROR（新增）

触发条件：

* 文件集合不一致
* git 操作失败
* 状态异常

---

### 8. RECOVERY（新增）

策略：

```text
1. 重新同步 file_set
2. 或回滚至 BASE_COMMIT
3. 或请求用户介入
```

---

## 六、输出要求（更新）

阶段完成后输出：

```text
1. summary 文件路径
2. commit hash
3. 简要说明
```

---

## 七、设计核心总结（v2）

```text
1. Git 是唯一真实状态
2. Agent 负责意图表达
3. commit = 阶段边界
4. 必须双向校验（Agent vs Git）
5. 不得自动处理用户改动
6. 每阶段必须可恢复
```

---

## 八、设计定位（重要）

本协议不再只是“行为规范”，而是：

👉 Agent 的“版本控制中间层（Agent VCS Layer）”

适用于：

* AI coding agent
* 多 agent 协作系统
* 自动化工程系统

```
```
