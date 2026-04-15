---
name: obsidian-backlinks
description: >-
  为 obsidian 文章自动添加 YAML frontmatter、标签和高质量双向链接。
  当以下情况时使用：
  (1) 用户要求为文章添加双向链接/backlinks
  (2) 用户提到"双链"、"backlinks"、"链接增强"
  (3) 用户要求整理 Obsidian 笔记的标签和 frontmatter
  (4) 用户要求处理概念归一化或知识网络构建
---

# Obsidian 双向链接增强

为 Obsidian 文章自动添加 YAML frontmatter、标签和**高质量双向链接**，构建干净、可读、可扩展的知识网络。

## 前置条件

- 文章已存在于 Obsidian Vault，如果用户不指定 Vault，则为默认
- 用户允许自动识别技术概念（或指定标签范围）
- 概念标准化配置文件：`.concepts.yaml`（不存在则自动创建在 `workspace` 中）
- 格式严格遵循 YAML，示例：
```yaml
concept_mapping:
  - standard: "LLM"
    aliases: ["大模型", "大型语言模型"]
  - standard: "KV Cache"
    aliases: ["kvcache", "KV缓存"]
  - standard: "Pre-Norm"
    aliases: ["prenorm", "pre norm"]
  - standard: "Attention Sink"
    aliases: ["注意力下沉"]
excluded:
  - AI
  - AGI
  - 深度学习
  - 模型
  - 算法
  - API
  - Token
```

## 执行步骤

### 1. 读取文章

使用 Obsidian CLI，读取目标文章：

```bash
obsidian read path="Work/.../文章名.md" slient
```

### 2. 概念识别与归一化

从正文抽取技术术语后执行：

- **统一标准名称**
	- 读取 概念标准化配置文件，构建「别名 → 标准词」映射
	- 匹配到别名则使用：`[[标准词|别名]]` 格式
	    - LLM / 大模型 → `[[LLM|大模型]]`
	    - KV Cache / kvcache → `[[KV Cache|kvcache]]`
	    - Pre-Norm / prenorm → `[[Pre-Norm|prenorm]]`
    - tags 统一使用 **standard 标准名称**
    
- **过滤泛概念**（仅作为标签，不生成双链）
    - AI、AGI、深度学习、模型、算法、API、Token
    - 发现新的泛概念，更新到概念标准化配置文件，需要去重
    
- **自动更新概念库**
	- 新别名：追加到对应标准词的 aliases 并去重
	- 新标准词：在 concept_mapping 下新增条目
	- 自动写入概念标准化配置文件，保持格式整洁无重复

### 3. 按优先级构建链接（严格执行）

**高优先级（⭐⭐⭐）必链**

- 文章主题 / 核心定义
- 领域核心机制
- 关键架构与方法

**中优先级（⭐⭐）建议链**

- 架构、协议、系统结构
- 专业工具、技术框架

**低优先级（⭐）可选链**

- 次要相关概念
- 辅助工具、提及内容

**绝不链接**

- 通用名词：token、向量、API、模型
- 产品名称：GPT-4、Claude、文心一言
- 无独立词条价值的泛概念

### 4. 关键强化：仅首次出现链接

1. 先扫描全文，**记录每个概念第一次出现的位置**
2. 只对**第一次出现**的术语添加 `[[...]]`
3. 同一概念**后续出现一律不处理、不链接**
4. 标题中的概念算作第一次出现
5. 已存在的双链不再重复处理

示例：

- 第 1 次：大模型 → `[[LLM|大模型]]`
- 第 2 次：大模型 → 保持原文「大模型」

### 5. 写入文件

先获取 vault 根目录：

```bash
obsidian vault info="path" silent
```

直接使用 `write` 工具操作文件，避免 Obsidian CLI 对于 \t、\n 的转义处理问题：

```
write(path="<vault根目录>/<文章路径>", content="...")
```

## 输出规范

- ✅ 标准 YAML frontmatter
- ✅ 5～15 个高质量双向链接，取决于文章长度和概念密度
- ✅ 不文末追加相关笔记
- ✅ 不修改原文内容
- ✅ 不产生重复链接
- ✅ 排版清爽、可读性优先

## 示例

**原文：**
```markdown
# Attention Sink 解析 
Attention Sink 是大模型推理过程中的现象...
```

**增强后：**
```markdown
---
title: Attention Sink 解析
date: 2026-04-14
tags:
  - LLM
  - Transformer
  - Attention-Sink
  - 推理优化
category: 技术笔记
---

# [[Attention Sink]] 解析

Attention Sink 是[[LLM|大模型]]推理过程中的现象...
```

## 注意事项

- **先预览，再执行**：输出标签列表和待链接列表，用户确认后写入
- **不修改原文内容**：仅新增 YAML 与双链，不删改正文
- **概念统一归一化**：全库标准名称唯一，由 概念标准化配置文件统一维护
- **兼容已有双链**
