---
name: podcast-to-article
description: "Transform podcast episodes into structured Chinese article write-ups using Podwise CLI data (transcript, mindmap, chapters, summary, highlights). Use when: user wants to write an article from a podcast episode, convert a podcast into a blog post, or create detailed Chinese content from podcast audio. Requires: Podwise CLI configured with API key."
---

# Podcast to Article

Transform podcast episodes into structured, detailed Chinese articles using Podwise CLI.

## Prerequisites

Before starting, load [references/podwise-cli.md](references/podwise-cli.md) for CLI syntax reference, then verify:

```bash
podwise --help
podwise config show
```

## Workflow

### 1. Collect Structure Data (Phase 1 — lightweight)

Given a Podwise episode URL, fetch **only the lightweight structural data** first:

```bash
# Parallel — these are all small outputs
podwise get summary <url>
podwise get mindmap <url>
podwise get chapters <url>
podwise get highlights <url>
```

**DO NOT** fetch transcript yet. The mindmap provides the structural backbone.

Also **search the web for the original podcast URL** at this stage.

### 2. Write Article Skeleton

Using ONLY summary + mindmap + chapters + highlights (already in context), write the article skeleton to a file:

- Title, frontmatter, introduction, 核心观点速览（by speaker）
- For each mindmap chapter: `##` heading, timestamp line, `###` sub-headings with brief placeholder notes
- Source footer with original URL

Save this skeleton file. This ensures the structure is locked in before touching the heavy transcript.

### 3. Collect Transcript (Phase 2 — on-demand)

Fetch transcript and save to a temp file — **do NOT read it into context yet**:

```bash
podwise get transcript <url> > /tmp/podcast_transcript.txt
```

### 4. Fill Content Chapter by Chapter

For each chapter in the skeleton, **extract only the relevant transcript segment** using the chapter's timestamp range:

```bash
# Extract transcript lines for a chapter starting at 06:38 and ending at 11:25
sed -n '/\[06:3/,/\[11:2/p' /tmp/podcast_transcript.txt
```

**Key principle: never load the full transcript into context.** Extract only the slice you need for the current chapter.

**⚠️ 每拉一段 transcript，必须在当轮立刻消化并写入文件。** 不要连续多轮拉取 transcript 囤积在上下文里——上下文会爆。正确流程：拉 Ch1 → 立刻 edit 写入文件 → 拉 Ch2 → 立刻 edit 写入文件。

For each chapter slice:
1. Read the extracted transcript segment (one tool call per chapter)
2. Select the best dialogue exchanges — don't dump everything, pick the most impactful ones
3. Apply ASR corrections (see below)
4. `edit` the skeleton file to replace the chapter's placeholder with actual content (dialogue + sub-headings + minimal commentary)

**以上 4 步必须在同一轮完成，不要拆成多轮工具调用。**

Repeat for each chapter. Context stays small because you only hold one chapter's transcript at a time.

### 5. Append Full Transcript

After all chapters are filled, append the full transcript as a collapsible `<details>` section. Use `exec` to process the raw transcript file into the correct format (add blank lines between entries, map speaker names), then append to the article file.

**⚠️ 必须用 `<details>` 包裹完整 transcript，不要直接追加到正文末尾。** 格式如下：

```markdown
---

<details>
<summary>📖 完整访谈记录（点击展开）</summary>

**[00:00] Speaker Name**：内容……

</details>
```

This is the last step — the formatted transcript is large but by now the article body is already written and saved.

**用 Python/sed 脚本自动格式化 transcript**，不要直接 `cat` 追加。必须：
1. 每条 `[HH:MM]` 时间戳行前加空行
2. 格式改为 `**[HH:MM] Speaker**：内容`
3. 包在 `<details>` 标签里

示例 Python 脚本见上文。

### 6. Publish

```bash
cd <repo> && git pull
# Article file is already written incrementally
git add <article-file> && git commit -m "Add: <article title>" && git push
```

**写完后必须给出文章链接**，格式：`https://github.com/<user>/<repo>/blob/main/<filename>`

## Article Structure

Follow the mindmap structure strictly:

1. **Title**: Must be compelling and indicate the content format. **标题必须是单一聚焦的主题概括，不要罗列多个关键词拼凑。**
   - **错误示范**：`黄金算法、AI吸血鬼与自杀式共情 | ...`（罗列式，意识流）
   - **正确示范**：`AI 时代的 Builder 文化 | ...`（单一主题，概括性强）
   - 播客访谈（有主持人+嘉宾对话）→ `核心主题 | 嘉宾名（身份）深度访谈`
   - YouTube 演讲/工作坊（单人主讲）→ `核心主题 | 演讲者名（身份）演讲`
   - 其他格式（圆桌讨论、辩论等）→ 根据实际情况命名
   - **选标题时问自己：如果读者只看标题，能不能一句话说清楚这篇讲什么？如果不能，就换一个更聚焦的。**
   Example: `AI 时代的 Builder 文化 | Marc Andreessen（a16z）深度访谈`
   Example: `AI 编程不是氛围编程 | Matt Pocock（AI Hero）演讲`
2. **Source intro block** (right after title, before introduction): A brief paragraph introducing the episode source, speaker identities, and content format:
   - 播客访谈：`*本文基于 [播客名] 第 X 期对话整理（发布于...）。嘉宾：**嘉宾名**（身份简介）。*[原始播客链接](url)*`
   - 演讲/工作坊：`*本文基于 [演讲者名] 的 [活动名] 演讲整理（发布于...）。[演讲者名] 是 [身份简介]。*[原始链接](url)*`
   **必须包含发布日期**，以说明资讯的时效性。**必须根据实际来源类型准确描述**——不要把演讲写成访谈，也不要把访谈写成演讲。
3. **Introduction** (1-2 paragraphs in *italics*): Editor's insight — distill the core tension or theme of the episode.
3. **核心观点速览** (Key Viewpoints by Speaker): 按角色整理，每人 3-5 个核心观点，用加粗列表。格式：
   ```markdown
   ## 核心观点速览

   **嘉宾名（身份）**

   - 观点一
   - 观点二
   - ...

   **主持人名（身份）**

   - 观点一
   - ...
   ```
   每个观点要简洁有力，突出该角色独有的视角和判断。不要写成泛泛的总结，要体现个人立场。
4. **Body sections**: One section per mindmap chapter:
   - **`##` heading** must be insight-driven — a punchy phrase that captures the core takeaway
   - On the **next line** after the heading, add the timestamp in italics: `*本期播客 XX:XX 开始*`
   - **`###` sub-headings** inside each chapter provide navigation anchors
   - Each sub-section should have its own narrative arc, not just a wall of dialogue
   - **Each `###` sub-section ends with a blockquoted summary** of each speaker's core viewpoint in that section, format:
     ```
     > **Aaron Levie** 认为 xxx；**Martin Casado** 强调 xxx；**Steven Sinofsky** 指出 xxx。
     ```
     Use bold speaker names, one sentence per person. Only include speakers who contributed in that section.
5. **Full transcript appendix**: Complete transcript in a collapsible `<details>` section:
   ```markdown
   ---

   <details>
   <summary>完整对话记录（原文）</summary>

   **[00:00] Speaker Name**：原始对话内容……

   **[00:05] Speaker Name**：原始对话内容……

   </details>

   ---
   ```
   Transcript formatting rules:
   - Keep original language (body is translated, transcript serves as reference)
   - Preserve `[HH:MM]` timestamps from each transcript line
   - **⚠️ CRITICAL: 必须每行对话之间空一行** — 这是硬性要求，没有空行的 transcript 不合格。直接 append 原始 transcript 到文件会导致行间无空行，Markdown 渲染时所有内容会挤在一起。**必须使用下方的 Python 脚本格式化后再插入**。
   - Map generic speaker labels to actual names when known
   - **Apply ASR corrections** to transcript

   推荐用 Python 脚本格式化：
   ```python
   import re
   with open('/tmp/raw_transcript.txt') as f:
       lines = f.readlines()
   formatted = []
   for line in lines:
       line = line.strip()
       if not line: continue
       m = re.match(r'\[(\d+:\d+)\]\s*-\s*(.+?):\s*(.*)', line)
       if m:
           ts, speaker, text = m.groups()
           formatted.append(f"**[{ts}] {speaker}**：{text}\n\n")  # 注意双换行
       else:
           formatted.append(f"{line}\n\n")
   with open('/tmp/formatted_transcript.txt', 'w') as f:
       f.writelines(formatted)
   ```
6. **No source footer at the end** — the original podcast link is already in the guest intro block at the top.

## Writing Guidelines

**Language**: Always write in Chinese. Translate all English content.

**No emoji**: Never use emoji anywhere in the article — title, headings, bullet points, or body text. Plain text with bold for emphasis only. Emoji creates an artificial, AI-generated feel.

**Dialogue formatting**: Use bold speaker labels, NOT blockquotes:

```
**王超**：原文对话内容……
```

**Dialogue-first principle**: The article's core value is preserving the podcast's original Q&A rhythm. Rules:
- **Maximize original dialogue** — aim for 60-70% of body content to be direct quotes
- **Preserve the back-and-forth** — show Jacob asks → swyx answers → Jacob follows up. This "breathing rhythm" is what makes it feel like a real conversation
- **Minimize commentary** — editorial commentary should be the exception, not the norm. Only add commentary when context is genuinely needed to bridge topics or provide essential background
- **Commentary must be in *italics*** — all non-dialogue text (scene-setting, transitions, context) must be italicized so it's visually distinct from the dialogue and doesn't interfere with reading flow

**What commentary should do (sparingly):**
- Bridge between topic shifts when the podcast jumps abruptly
- Provide essential context the speakers assume but readers don't know
- Add a single distilled insight at the end of a section

**What commentary should NOT do:**
- Summarize what was just said in the dialogue (readers just read it)
- Label who people are (do this once in the intro, not repeatedly)
- Pre-empt or frame every single dialogue exchange (let the dialogue speak for itself)
- Add evaluative statements ("this is a powerful insight", "interestingly")

**Section structure**: For each mindmap chapter:
- Start directly with dialogue or a single italic transition line
- Present dialogue exchanges with bold speaker labels, preserving the Q&A rhythm
- Use `###` sub-headings to break up long dialogue runs — every 5-8 dialogue exchanges should have an anchor
- Cover ALL sub-topics from the mindmap — don't skip any
- End with a brief italic wrap-up only if the section needs closure

**ASR Correction Rules**: Podcast transcripts from ASR contain systematic errors. You MUST correct these. Common patterns:

- Brand/product name errors: `Vive 3 101` → `Web3 101`, `Hackingface` → `HuggingFace`, `Openroid` → `OpenRouter`, `Elpaca` → `Alpaca`
- Technical term errors: `infinity band` → `InfiniBand`, `上下门` → `上下文`, `神仙网络` → `神经网络`, `征留出` → `蒸馏出`
- Entity name errors: `HLZ` → `a16z`, `一等 network` → `Eden Network`
- Context errors: `龙虾` → `OpenHands` (when referring to an agent product), `discount 的` → `Discord 的`
- Homophone errors: `吸入` → `吸纳`, `出版` → `发布`, `根护苗症` → `根正苗红`
- General: Always verify suspicious terms against the podcast context. If unsure, use web search to confirm.

**英语俗语/俚语翻译原则**：不要字面翻译，用中文意译传达实际含义。例如：
- "dog years" → "一年当七年过"（而非"狗年时间"）
- "cargo culting" → "盲目模仿"（而非"货物崇拜"）
- "rubber stamp" → "走过场/走形式"（而非"橡皮图章"）

**专业术语翻译原则**：如果翻译不能做到信达雅，应该保留原文。不要生造中文译名。例如：
- headless → headless（而非"无头化"）
- human in the loop → human in the loop（而非"人类在环"）
- SaaSpocalypse → SaaSpocalypse（而非"SaaS末日论"）
- vibe coding → vibe coding（而非"氛围编程"）
- 判断标准：在中文技术语境中是否有广为接受的译名？如果没有，保留英文

**内容修剪原则**：播客开场的闲聊、玩笑、无实质内容的寒暄应该移除，直接从有信息量的对话开始。不要保留 "大家好"、天气玩笑、发型讨论等无营养内容。

Apply corrections to BOTH the article body dialogue AND the full transcript appendix.

**Depth**: Aim for 8,000+ characters for a 1-hour episode. Don't summarize too aggressively — the goal is to let readers feel like they've listened to the podcast.

## Output Format

Frontmatter for the markdown file:

```yaml
---
title: "<article title — interview format>"
date: "<YYYY-MM-DD>"
tags: [<relevant tags>]
source: "<podcast name and episode title>"
sourceUrl: "<original podcast episode URL, NOT podwise URL>"
---
```

## Context Management Rules

**This is the most important operational part of this skill.**

The #1 failure mode is loading too much data into context and timing out or producing truncated output.

**Rules:**
1. **Never load the full transcript into context at once.** Always use `sed` to extract slices by timestamp.
2. **Structure data first, transcript later.** The skeleton (from mindmap/chapters) is lightweight and should be written to a file before any transcript processing.
3. **One chapter at a time.** Read one transcript slice, edit the file, move on. Context resets between chapters (each tool call is independent).
4. **Transcript appendix last.** The full formatted transcript is large. Append it only after the article body is complete.
5. **Use `exec` + `sed` for transcript slicing**, not `read` with offset/limit. Timestamp-based slicing is more precise and doesn't require knowing line numbers.

## Quality Checklist

Before finalizing, verify:
- [ ] Title follows correct format: 访谈→`深度访谈`，演讲→`演讲`，工作坊→`演讲`
- [ ] No emoji anywhere in the article
- [ ] **核心观点速览** at the top, organized by speaker (3-5 points each)
- [ ] Mindmap structure fully covered (all chapters, all sub-topics)
- [ ] **正文内容 300-500 行**（不含 transcript appendix）。如果原始内容不足 300 行，将全部对话原文放入正文，用斜体解说做桥接和分析。
- [ ] All commentary in *italics*, dialogue in plain text
- [ ] Source intro includes publish date
- [ ] Full transcript in `<details>` section with corrections and blank lines **（不能漏掉 `<details>` 标签！）**
- [ ] Chapter titles (`##`) are insight-driven, not generic
- [ ] Dialogue dominates body content (60-70%), preserves Q&A rhythm
- [ ] Commentary is minimal and all in *italics*
- [ ] Original dialogue with bold speaker labels, ASR corrected
- [ ] All English content translated to Chinese
- [ ] Article length substantial relative to episode duration
- [ ] Source URL uses original podcast link
- [ ] Full transcript in `<details>` section with corrections and blank lines
