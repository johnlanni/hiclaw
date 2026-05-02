---
name: paper-review
description: "Transform academic papers into structured Chinese review articles. Use when: user wants to write an article from an arXiv paper, convert a research paper into a blog post, or create detailed Chinese paper analysis. Supports Zread (via mcporter) for associated GitHub repo analysis."
---

# Paper Review

Transform academic papers into structured, detailed Chinese review articles.

## Workflow

### 1. Fetch Paper Content

Given an arXiv paper URL or ID:

```bash
# Prefer HTML version for structured content (headings, figures, tables)
# URL format: https://arxiv.org/html/{paper_id}
# Abstract page: https://arxiv.org/abs/{paper_id}
```

Use `web_fetch` to retrieve the paper. If HTML version is truncated, fetch additional sections as needed.

**Key data to extract:**
- Title, authors, institutions
- Abstract
- All section headings and their content
- Key formulas, tables, figures descriptions
- Experimental results
- Related work and references

### 2. Write Article with Core Insights Summary

Write the article to a file in the target repo's `papers/` directory. Structure:

1. **核心观点总结** — 在开头，3-5 个要点概括论文的核心创意/贡献
2. **分章节论文内容** — 主体部分表述论文原文内容，用中文撰写，英文标题保留
3. **GitHub 项目分析（附录）** — 如果论文有对应 GitHub 仓库，用 Zread 分析后附加

**Writing principle: 主体表述论文内容，附加内容只做方便理解的总结，不放个人评论。**

### 3. Analyze GitHub Repository (if available)

If the paper has an associated GitHub repo, use **mcporter + Zread** for deep code analysis:

#### 3.1 Configure Zread (first time only)

```bash
mcporter config add zread \
  --url https://open.bigmodel.cn/api/mcp/zread/mcp \
  --transport http \
  --header 'Authorization=Bearer <API_KEY>'
```

#### 3.2 Call Zread tools

```bash
# Get repo directory structure
mcporter call zread.get_repo_structure repo_name=owner/repo --output json

# Search for specific topics (repeat for different aspects)
mcporter call zread.search_doc repo_name=owner/repo query='architecture design' language=zh --output json
mcporter call zread.search_doc repo_name=owner/repo query='core algorithm implementation' language=zh --output json
mcplorer call zread.search_doc repo_name=owner/repo query='configuration and usage' language=zh --output json

# Read specific files
mcporter call zread.read_file repo_name=owner/repo file_path=docs/architecture.md --output json
```

**Search queries should cover:**
- Architecture and system design
- Core algorithm/data structure implementation
- Configuration and usage patterns
- Testing and evaluation
- Any unique features mentioned in the paper

### 4. Publish

```bash
cd <repo> && git pull
git add papers/<filename>.md && git commit -m "Add paper review: <short title>" && git push
```

**写完后必须给出文章链接。**

## Article Structure

```markdown
# 中文标题（点明核心贡献）

> 论文：[English Title](arxiv_url)
> 作者：Author names
> 机构：Institutions
> 项目：[project_url](link)（如有）

---

## 💡 核心观点

2-3 句话概述论文主张，然后列出 3-5 个核心贡献点：

1. **贡献一** — 一句话概括 + 简要说明
2. **贡献二** — ...
3. **贡献三** — ...

实验结果摘要（如果有）。

---

## 1. 引言/问题背景

论文要解决的问题、现有方法的局限性、研究动机。

## 2. 方法（按论文章节组织）

### 2.1 子模块一
...

### 2.2 子模块二
...

## 3. 实验结果

...

## 4. 局限性与讨论

**注意：这部分只表述论文自身指出的局限性，不加个人评论。**

---

## 附录：GitHub 项目代码分析（基于 Zread）

> 源码仓库：[github_url](link)
> 分析工具：[智谱 Zread](zread_url)

### A. 技术栈
...

### B. 系统架构
...

### C. 核心模块详解
...

（根据 Zread 分析结果组织，覆盖架构、核心实现、配置使用等维度）
```

## Writing Guidelines

**Language**: 中文撰写，英文专有名词和标题保留原文。

**主体是论文内容**：
- 文章主体部分表述论文的内容
- 附加内容只做方便理解的总结（如类比说明、概念映射）
- **不放个人评论、评价、"我的思考"等主观内容**
- 局限性讨论只写论文自身指出的局限

**段落处理**：
- 按论文章节分段落处理总结
- 每个 `##` 对应论文的一个主要章节
- 用 `###` 拆分子主题
- 长章节适当拆分，短章节可合并

**中文摘要 vs 英文标题**：
- 所有论述用中文
- 论文标题、方法名、模型名保留英文
- 技术术语首次出现时附英文原文：如"标准操作流程（SOP）"

**代码/公式**：
- 关键公式用 LaTeX 格式保留
- 代码片段或伪代码保留在代码块中
- 数据流图/架构图用文字描述或 Mermaid 图

**格式要求**：
- Discord 发送时不用 markdown 表格
- 文件中使用 markdown 表格
- 链接使用标准 markdown 格式

## Context Management

**论文可能很长，需要分段处理：**

1. `web_fetch` 可能截断长论文 — 需要多次 fetch 获取完整内容
2. Zread 返回内容较长 — 按主题分次搜索，不要一次搜所有内容
3. 写文件时逐步 `edit`，不要一次性写完整篇文章（可能超时）

## Quality Checklist

Before finalizing, verify:
- [ ] 开头有核心观点总结（3-5 个要点）
- [ ] 主体按论文章节组织，表述论文内容
- [ ] 没有个人评论/评价（"我的思考"、"值得借鉴"等）
- [ ] 局限性部分只包含论文自身指出的内容
- [ ] 如有 GitHub 仓库，附录包含 Zread 分析
- [ ] 中文撰写，英文术语/标题保留
- [ ] 已 push 到 target repo 的 `papers/` 目录
- [ ] 给出了文章链接
