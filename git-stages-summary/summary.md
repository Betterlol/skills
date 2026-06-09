# Git Stages Summary

## 设计目标

`Git Stages Summary` 是一种面向 LLM 的项目上下文压缩协议（Project Context Compression Protocol）。

其核心目标是在保留完整实现证据的前提下，将多个开发阶段的成果浓缩为一份结构化、低 Token 成本、可跨会话传递的项目总结文档，使网页端或上下文受限的 LLM 能快速理解项目实现内容与演进过程。

---

## 解决的问题

在实际开发过程中，一个功能通常会经历多个阶段：

* 需求分析
* 架构设计
* 功能实现
* 重构优化
* Bug 修复
* 测试验证

虽然每个阶段都可能产生总结文档，但当项目规模扩大后：

* 阶段文档数量持续增长
* 历史对话上下文难以保留
* 新的 LLM 会话无法快速继承项目知识
* 阅读全部开发记录成本过高

最终导致：

> LLM 很难在有限上下文窗口内快速理解项目当前状态。

---

## 核心思路

本 Skill 采用双层模型：

### Intent Layer（意图层）

来源于开发阶段总结。

记录：

* 阶段目标
* 涉及文件
* 关键设计决策
* 实现思路

用于帮助 LLM 快速理解：

> 为什么做这次修改。

---

### Evidence Layer（证据层）

来源于：

```bash
git diff <START_COMMIT>..<END_COMMIT>
```

记录：

* 实际代码变更
* 文件增删改
* 真实实现细节

作为：

> 唯一权威事实来源（Single Source of Truth）。

用于帮助 LLM 理解：

> 实际代码到底改了什么。

---

## 输入

Skill 接收两类输入：

### 1. 阶段总结

每个阶段仅保留精简信息：

* 阶段名称
* 阶段目标
* 文件变更列表
* 关键决策说明

不需要读取详细开发日志。

---

### 2. Git Diff

指定 Commit Range 的完整代码差异：

```bash
git diff <START_COMMIT>..<END_COMMIT>
```

用于提供完整实现证据。

---

## 输出

最终生成统一 Release Summary：

### 1. Release Metadata

记录：

* Release ID
* Commit Range
* 阶段数量
* 统计信息

---

### 2. High-Level Intent

整体实现目标概述。

用于快速建立项目认知。

---

### 3. Stage-by-Stage Overview

按阶段展示：

* Goal
* Files
* Key Decisions

保持轻量化，不包含代码细节。

用于理解开发过程。

---

### 4. Complete Diff

完整 Git Diff。

作为最终实现证据。

用于理解代码层面的真实变更。

---

### 5. Current Status

项目当前状态：

* 测试情况
* 已知风险
* 依赖变化
* 注意事项

---

## 文件过滤能力

支持用户自定义过滤规则：

例如：

```text
*.md
*.json
*.yaml
*.snap
*.map
```

或：

```text
仅保留 src/*
仅保留 lib/*
```

用于：

* 排除文档噪声
* 排除配置文件
* 聚焦业务逻辑
* 降低上下文体积

过滤仅影响输出内容，不修改 Git 数据。

---

## 适用场景

### 项目交接

让新的开发者或新的 LLM 会话快速理解项目。

### Context Transfer

将大型项目迁移到新的上下文窗口。

### Release Review

审查某个版本的完整实现内容。

### 长周期开发

避免持续积累的大量阶段文档导致上下文膨胀。

### Web LLM 输入

为网页端 LLM 提供高密度项目知识包。

---

## 核心原则

1. 阶段总结负责解释「为什么」
2. Git Diff 负责证明「改了什么」
3. Intent 与 Evidence 必须同时存在
4. Git Diff 是唯一权威实现依据
5. 不从记忆重构代码内容
6. 所有证据必须来自真实 Git 状态
7. 优先降低 Token 消耗
8. 保持跨会话可迁移性

---

## 一句话总结

> Git Stages Summary 是一种面向 LLM 的项目上下文压缩协议，通过聚合阶段总结（Intent）与 Git Diff（Evidence），在保留完整实现依据的同时，以最低 Token 成本实现项目理解、交接与上下文传递。
