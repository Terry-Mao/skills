---
name: obsidian-wiki
description: 为 Obsidian 构建、维护、查询 Wiki 知识库。
---

# Obsidian Wiki

为 Obsidian 文章构建 Wiki 知识体系：提取摘要、深写概念、构建索引。

## 使用场景

- 整理已有文章，提取概念并联网搜索完善
- 整理已有文章，写入结构化摘要
- 构建原文 → 摘要 → 概念的三层索引
- 定期健康检查，补全和更新概念库

## 前置条件

- 文章已保存在 Obsidian Vault 中
- **文章已经过 `obsidian-backlinks` 处理**（已有 YAML frontmatter、tags、双链）
- 用户确认提取的概念列表（或授权自动判断）

## 目录结构

**⚠️ 所有路径均相对于 Obsidian Vault 根目录（= Documents）**

| 用途 | 路径 |
|------|------|
| 资源目录（原文） | `Work/03 - Resources` |
| Wiki 根目录 | `Work/Wiki/` |
| 摘要目录 | `Work/Wiki/Summaries/` |
| 概念目录 | `Work/Wiki/Concepts/` |
| 索引文件 | `Work/Wiki/INDEX.md` |
| 日志文件 | `Work/Wiki/LOG.md` |
| 已处理清单 | `Work/Wiki/.done.files` |
| 概念归一化配置 | `.concepts.yaml`（workspace 中） |

## 核心规则

- ⚠️ **禁止修改资源目录中的原文件**
- ⚠️ 概念过滤标准统一使用 `.concepts.yaml` 的 `excluded` 列表（与 obsidian-backlinks 共享）
- 操作 Obsidian 使用 `obsidian` CLI（参考 `SKILL: obsidian-cli`）

---

## 增加文章

### 步骤 0：去重检查

读取已处理清单，跳过已处理的文章：

```bash
obsidian read path="Work/Wiki/.done.files"
```

`.done.files` 格式（每行一条）：
```
2026-04-14|Work/03 - Resources/RoPE 旋转位置编码.md|5 concepts|summary
```
格式：`日期|文章路径|概念数|处理类型`

如果目标文章路径已存在于清单中，跳过并告知用户。

### 步骤 1：解析文章的 tags 和双链

```bash
obsidian read path="Work/03 - Resources/文章名.md"
```

从已经过 backlinks 增强的文章中提取：

1. **tags**：从 YAML frontmatter 的 `tags` 字段提取
2. **双链**：用正则 `\[\[([^\]|]+)(?:\|[^\]]+)?\]\]` 提取所有 `[[概念]]` 和 `[[标准名|别名]]` 中的标准名
3. **去重合并**：tags 和双链中的概念取并集
4. **过滤**：读取 `.concepts.yaml` 的 `excluded` 列表，排除泛概念

输出**候选概念列表**，等待用户确认（除非用户已授权自动判断）。

### 步骤 2：写结构化摘要

根据文章类型选择摘要框架：

| 文章类型 | 摘要框架 | 判断依据 |
|----------|----------|----------|
| 学术论文 | IMRaD（Introduction / Methods / Results / Discussion） | 有 Abstract、References、实验章节 |
| 技术博客/教程 | WHAT-WHY-HOW（是什么 / 为什么 / 怎么做 / 关键结论） | 有代码示例、步骤说明 |
| 概念解析 | CONCEPT-MAP（定义 / 核心机制 / 对比 / 应用场景） | 围绕单一术语展开 |
| 实践笔记 | TIL（场景 / 问题 / 方案 / 收获） | 短篇、经验导向 |

摘要要求：
- 300-600 字，保留关键数据和结论
- 保留原文中的双链 `[[概念]]`
- YAML frontmatter 包含 `source_path` 指向原文

写入摘要目录：

```bash
obsidian create path="Work/Wiki/Summaries/文章名 - Summary.md" content="..." silent overwrite
```

### 步骤 3：编写概念文章（核心步骤）

对步骤 1 确认的每个概念，执行以下流程：

#### 3a. 概念适合性判断

适合写独立文章的概念：
- ✅ 核心领域术语（如 RoPE、KV Cache、Flash Attention）
- ✅ 核心技术机制（如 Attention Sink、Sliding Window）
- ✅ 架构/协议（如 Transformer、MoE、Ring Attention）
- ✅ 关键方法论（如 Position Interpolation、NTK-Aware Scaling）

**不适合的**：统一使用 `.concepts.yaml` 的 `excluded` 列表判断

#### 3b. 联网搜索（必须）

使用 minimax-search 搜索该概念的最新内容：

```bash
mcporter call minimax_coding.web_search query="<概念名> 原理 最新进展 2025 2026"
```

提取 10 篇相关结果的标题、链接、摘要。

#### 3c. 编写或更新概念文章

**检查概念文章是否已存在**：

```bash
obsidian read path="Work/Wiki/Concepts/概念名.md"
```

- **已存在**：结合联网搜索的新内容，补充新论证、新观点、新进展，保留原有内容
- **不存在**：写一篇完整文章，要求：
  - 1500 字以上
  - 结构：定义 → 核心原理（含数学推导/伪代码）→ 工程实践 → 对比方案 → 最新进展 → References
  - 清晰的原理解释和证明
  - 使用 `obsidian-backlinks` 增强该概念文章的双链和 tags

文章尾部 **必须包含 References**：
```markdown
## References

- [[原文标题]] — Obsidian 内部文章
- [论文/博客标题](https://url) — 外部链接
```

写入概念目录：

```bash
obsidian create path="Work/Wiki/Concepts/概念名.md" content="..." silent overwrite
```

#### 3d. 并行策略

- 概念文章之间**互相独立**，适合并行处理
- **当概念数 ≥ 3 时，使用 subagent 并行编写**
- 每个 subagent 负责：联网搜索 + 编写/更新一篇概念文章 + backlinks 增强
- 主 agent 负责：汇总结果、更新索引、记录日志

### 步骤 4：更新索引

INDEX.md 使用以下固定结构：

```markdown
# Wiki 索引

> 自动生成，请勿手动编辑。最后更新：YYYY-MM-DD

## 按分类

### 模型架构
- [[概念A]] — 一句话描述
- [[概念B]] — 一句话描述

### 训练方法
- ...

### 推理优化
- ...

### 位置编码
- ...

（按需增加分类）

## 摘要

- [[文章A - Summary]] ← [[文章A]]
- [[文章B - Summary]] ← [[文章B]]

## 统计

- 概念文章：XX 篇
- 摘要文章：XX 篇
- 最后更新：YYYY-MM-DD
```

分类不是固定的，根据概念的 tags 自动归类。一个概念可以出现在多个分类下。

```bash
obsidian create path="Work/Wiki/INDEX.md" content="..." silent overwrite
```

### 步骤 5：记录已处理文件

追加到 `.done.files`：

```bash
obsidian append path="Work/Wiki/.done.files" content="2026-04-14|Work/03 - Resources/文章名.md|5 concepts|summary"
```

### 步骤 6：记录操作日志

追加到 `LOG.md`：

```markdown
## 2026-04-14 16:00

**处理文章**：[[RoPE 旋转位置编码]]
**摘要**：[[RoPE 旋转位置编码 - Summary]]
**新增概念**：[[RoPE]]、[[旋转矩阵]]、[[Position Interpolation]]
**更新概念**：[[Transformer]]（补充 RoPE 相关内容）
**联网搜索**：10 条结果，5 条采纳
```

```bash
obsidian append path="Work/Wiki/LOG.md" content="..."
```

---

## 健康检查

定期扫描 Wiki 概念库，检查和补全内容。

### 检查项

1. **内容新鲜度**：对每篇概念文章，联网搜索最新进展，有重要更新则补充
2. **引用完整性**：检查 References 是否包含 Obsidian 内部链接和外部 URL
3. **双链有效性**：检查文章中的 `[[概念]]` 是否对应已有的概念文章，不存在则标记为待创建
4. **索引一致性**：确保所有概念文章和摘要文章都已收录在 INDEX.md 中

### 输出

- 更新需要更新的概念文章
- 创建缺失的概念文章
- 更新 INDEX.md
- 记录操作日志到 LOG.md
