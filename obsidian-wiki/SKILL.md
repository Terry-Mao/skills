---
name: obsidian-wiki
description: >-
  为 obsidian 构建、维护、查询 Wiki 知识库。
  当以下情况时使用：
  (1) 用户提到"wiki"、"知识库"
  (2) 用户要求文章增加到wiki/知识库中
  (3) 用户要求整理 obsidian wiki/知识库
  (4) 用户要求查询、搜索 obsidian wiki/知识库
---

# Obsidian Wiki

为 Obsidian 文章构建 Wiki 知识体系：提取摘要、深写概念、构建索引。

## 使用场景

- 整理已有文章，提取概念并联网搜索完善
- 整理已有文章，写入结构化摘要
- 构建原文 → 摘要 → 概念的三层索引
- 定期健康检查，补全和更新概念库

## 前置条件

- 文章已保存在 Obsidian Vault 中，如果用户不指定 Vault，则为默认
- **文章已经过 `obsidian-backlinks` 处理**（已有 YAML frontmatter、tags、双链）
- 用户确认提取的概念列表（或授权自动判断）

## 目录结构

| 用途         | 路径                                          |
| ---------- | ------------------------------------------- |
| 资源目录（原文）   | `<原文目录>`                                    |
| Wiki 根目录   | `<原文目录>/Wiki/`                              |
| 摘要目录       | `<Wiki 根目录>/Summaries/`                     |
| 概念目录       | `<Wiki 根目录>/Concepts/`                      |
| 索引文件       | `<Wiki 根目录>/INDEX.md`                       |
| 日志文件       | `<Wiki 根目录>/LOG.md`                         |
| 已处理清单      | `<Wiki 根目录>/.done.files`                    |
| 概念归一化配置    | `.concepts.yaml`（workspace 中）               |
| Vault 本地目录 | 通过 `obsidian vault info="path" silent` 命令获取 |

## 核心规则

- ⚠️ **禁止修改资源目录中的原文件**
- ⚠️ 概念过滤标准统一使用 `.concepts.yaml` 的 `excluded` 列表（与 obsidian-backlinks 共享）
- 操作 Obsidian 使用 `obsidian` CLI（参考 `SKILL: obsidian-cli`，注意带 `silent` 参数）
- 摘要只提炼、不创作；概念页可深度扩展 + 联网增强
- 一个标准概念 = 唯一一篇概念文章
- 所有页面必须带 References，形成可溯源链路
- 禁止泛概念独立成页（AI、模型、算法 等）

### 双链写入原则（摘要 & 概念文章统一遵守）

概念文章内部严格使用 `.concepts.yaml` standard 标准名，不新增别名

1. 只对**第一次出现**的术语添加 `[[...]]`
2. 同一概念**后续出现一律不处理、不链接**
3. 标题中的概念算作第一次出现
4. 已存在的双链不再重复处理

---

## 增加文章

### 步骤 0：去重检查

读取已处理清单，跳过已处理的文章：

```bash
obsidian read path="<Wiki 根目录>/.done.files" silent
```

`.done.files` 格式（每行一条）：
```
2026-04-14|文章路径.md|5 concepts|summary
```
格式：`日期|文章路径|概念数|处理类型`

如果目标文章路径已存在于清单中，跳过并告知用户。

### 步骤 1：读取文章 + 提取概念 + 写摘要（一次读取）

**一次性读取原文**，同时完成概念提取和摘要编写，避免重复读取：

```bash
obsidian read path="文章路径.md" silent
```

#### 1a. 提取候选概念（结构化解析，不需要语义理解）

从已经过 backlinks 增强的文章中**纯结构化提取**：

1. **tags**：从 YAML frontmatter 的 `tags` 字段提取
2. **双链**：用正则 `\[\[([^\]|]+)(?:\|[^\]]+)?\]\]` 提取所有 `[[概念]]` 和 `[[标准名|别名]]` 中的标准名
3. **去重合并**：tags 和双链中的概念取并集
4. **过滤**：读取 `.concepts.yaml` 的 `excluded` 列表，排除泛概念

输出**候选概念列表**，等待用户确认（除非用户已授权自动判断）。

#### 1b. 写结构化摘要

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

直接使用 `write` 工具操作文件，避免 Obsidian CLI 对于 \t、\n 的转义处理问题：

```
write(path="<Vault 本地目录>/<摘要目录>/文章名 - Summary.md", content="...")
```

### 步骤 2：编写概念文章（核心步骤）

对步骤 1a 确认的每个概念，执行以下流程：

#### 2a. 联网搜索（必须）

使用 minimax-search 搜索该概念的最新内容：

```bash
mcporter call minimax_coding.web_search query="<概念名> 最新进展"
```

提取 10 篇相关结果的标题、链接、摘要。

#### 2b. 编写或更新概念文章

**检查概念文章是否已存在**：

```bash
obsidian read path="<概念目录>/概念名.md" silent
```

- **已存在**：结合联网搜索的新内容，补充新论证、新观点、新进展，保留原有内容，增量不覆盖
- **不存在**：写一篇完整文章，要求：
  - 1500 字以上
  - 结构：定义 → 核心原理（含数学推导/伪代码）→ 工程实践 → 对比方案 → 最新进展 → References
  - 清晰的原理解释和证明
  - **直接在生成内容中嵌入 `[[双链]]` 和 YAML tags**（概念文章从零生成，agent 已知所有概念，无需再走 obsidian-backlinks 流程）

文章尾部 **必须包含 References**：
```markdown
## References

- [[原文标题]] — Obsidian 内部文章
- [论文/博客标题](https://url) — 外部链接
```

写入概念目录，先获取 vault 根目录：

```bash
obsidian vault info="path" silent
```

直接使用 `write` 工具操作文件，避免 Obsidian CLI 对于 \t、\n 的转义处理问题：

```
write(path="<Vault 本地目录>/<概念目录>/概念名.md", content="...")
```

#### 2c. 并行策略

- 概念文章之间**互相独立**，适合并行处理
- **当概念数 ≥ 3 时，使用 subagent 并行编写**
- 每个 subagent 负责：联网搜索 + 编写/更新一篇概念文章（含内嵌双链和 tags）
- 主 agent 负责：汇总结果、更新索引、记录日志

### 步骤 3：更新索引

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
obsidian create path="<Wiki 根目录>/INDEX.md" content="..." silent overwrite
```

### 步骤 4：记录已处理文件

追加到 `.done.files`：

```bash
obsidian append path="<Wiki 根目录>/.done.files" content="2026-04-14|文章路径.md|5 concepts|summary" silent
```

### 步骤 5：记录操作日志

追加到 `LOG.md`：

```markdown
## 2026-04-14 16:00
**源路径**：文章路径.md
**处理文章**：[[RoPE 旋转位置编码]]
**摘要**：[[RoPE 旋转位置编码 - Summary]]
**新增概念**：[[RoPE]]、[[旋转矩阵]]、[[Position Interpolation]]
**更新概念**：[[Transformer]]（补充 RoPE 相关内容）
**联网搜索**：10 条结果，5 条采纳
```

```bash
obsidian append path="<Wiki 根目录>/LOG.md" content="..." silent
```

---

## 健康检查

定期扫描 Wiki 概念库，检查和补全内容。

### 步骤 1：扫描现有概念文章

```bash
obsidian list path="<概念目录>" silent
```

获取所有概念文章列表。

### 步骤 2：逐项检查

对每篇概念文章执行以下检查：

#### 2a. 内容新鲜度

- 读取概念文章，检查 YAML frontmatter 中的 `date` 或文章末尾的"最新进展"章节
- **阈值**：超过 90 天未更新的概念，触发联网搜索
- 联网搜索该概念最新进展：
  ```bash
  mcporter call minimax_coding.web_search query="<概念名> 最新进展"
  ```
- 有重要更新（新论文、新版本、重大变更）则补充到文章中

#### 2b. 引用完整性

- 检查 `## References` 是否存在
- 是否至少包含 1 个 Obsidian 内部链接 `[[...]]`
- 是否至少包含 1 个外部 URL `[...](https://...)`
- 缺失则标记为待修复

#### 2c. 双链有效性

- 提取文章中所有 `[[概念]]` 链接
- 检查每个链接是否对应 `<概念目录>` 下已有的概念文章
- 不存在的链接收集为**待创建概念列表**

#### 2d. 索引一致性

- 读取 `<Wiki 根目录>/INDEX.md`
- 对比概念目录和摘要目录的实际文件列表
- 检查是否有文章未收录在索引中
#### 2e. 重复概念检查

- 根据 `.concepts.yaml` 的 aliases 检查是否存在同义不同名文件
- 如存在，标记为 **需要合并**
- 保留标准名，删除别名文件

### 步骤 3：输出检查报告

汇总后输出：

```
📋 Wiki 健康检查报告

✅ 正常概念：XX 篇
⚠️ 需要更新（超过 90 天）：概念A、概念B
❌ 缺少 References：概念C
🔗 断链（概念文章不存在）：概念D、概念E
📑 未收录在索引中：概念F

建议操作：
1. 更新 X 篇过期概念
2. 创建 X 篇缺失概念
3. 修复 X 篇引用
4. 重建索引
```

**等待用户确认**后再执行修复操作。

### 步骤 4：执行修复

根据用户确认的范围：

- **更新过期概念**：联网搜索 + 补充内容（同增加文章的步骤 2c）
- **创建缺失概念**：联网搜索 + 编写新文章（同增加文章的步骤 2b-2c）
- **修复引用**：补全 References 章节
- **重建索引**：同增加文章的步骤 3
- **记录日志**：同增加文章的步骤 5
