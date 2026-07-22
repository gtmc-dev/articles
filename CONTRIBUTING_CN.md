---
skip: true
---

# 提交文章

[English Version](CONTRIBUTING.md)

## 关于本仓库

本仓库专门用于存储 Graduate Texts in Minecraft 的文章内容。所有文章的提交、修订与审核流程均在此处理。

## 管理结构

关于管理模式的细则，查看[文章编辑文档](REVIEWERS_CN.md)。

本项目采用扁平化的组织结构。仅设有两种角色：文章编辑和贡献者。
文章编辑由内部作者团队轮流担任。除此之外，外部贡献者与内部作者之间没有层级上的区别。

文章编辑的职责包括：

- 执行写作规范
- 进行事实核查
- 解决合并请求中的冲突
- 审批 Pull Request (PR)

## 提交工作流

提交一篇文章非常简单：

1. 通过网站或 GitHub 提交以请求审核。你可以直接在网站上操作，或者在本地编辑后推送到 GitHub。
2. 填写 Frontmatter（元数据），即 Markdown 文件顶部的信息块。
3. 编写正文，你可以使用标准的 Markdown 语法以及我们支持的自定义语法（见下文）。
4. 提交 Pull Request (PR)，通过网站或 GitHub 提交以请求审核。

## Frontmatter 指南

每篇 Markdown 文件的最顶部都必须包含一个 YAML frontmatter 信息块。源文章和翻译文章使用不同的 frontmatter 结构。

仅供仓库使用且不应出现在网站上的 Markdown 文档使用以下 frontmatter：

```yaml
---
skip: true
---
```

不要在文章或章节文件中设置 `skip`。

源文章（`*.zh.md`）定义自己的 URL 与源语言元数据：

```yaml
---
slug: basics-and-structure
title: 前置知识与树场的基本结构
description: 可选的简短摘要
index: 1
is-advanced: false
banner:
  src: img/banner.png
  alt: A descriptive alt text
---
```

源文章字段含义：

- `slug`：文章的唯一 URL 标识符。
- `title`：文章的源语言标题。
- `description`：可选的简短摘要，用于预览和页面元数据。
- `index`：表示文章在目录中的排序。`-1` 表示不排序，请不要提交带有 `index: -1` 的文章。
- `is-advanced`：设为 `true` 可以将**整篇**文章标记为进阶内容。
- `banner`：包含 `src`（相对图片路径）和 `alt`（图片的文本描述）的嵌套字段。

翻译文章（`*.en.md`）指向对应的源文件和源修订版本：

```yaml
---
translates: ./前置知识与树场的基本结构.zh.md
translated-from-revision: 20ad46927f7ac8db7b9c875504c9a899209214af
title: Basics and Structure
description: Optional translated summary
banner:
  src: img/banner.png
  alt: A descriptive alt text
---
```

翻译字段含义：

- `translates`：该文件所翻译的源文章相对路径。
- `translated-from-revision`：翻译所依据的源文章 Git commit SHA。
- `title`、`description` 和 `banner`：可选的翻译覆盖字段。

源章节/目录 `README.zh.md` 使用同样的源文章规则，并填写章节标题字段：

```yaml
---
slug: tree-farm
chapter-title: 树场
intro-title: 前言
index: -1
---
```

翻译章节/目录 `README.en.md` 使用翻译结构：

```yaml
---
translates: ./README.zh.md
translated-from-revision: 365edcc68df67b53fc5dfada26eddfd5a8e69a02
chapter-title: Tree Farm
intro-title: Foreword
---
```

章节字段含义：

- `chapter-title`：章节或目录的显示名称。
- `intro-title`：该 `README.md` 内引言部分的标题。
- `index`：该章节在同级目录中的排序序号。
- `slug`：该章节目录的唯一 URL 标识符。在嵌套章节中，本字段允许留空来省略 URL 层级。

不要再添加 `title-en`、`chapter-title-en`、`intro-title-en`、`author`、`co-authors`、`date` 或 `lastmod` 等旧字段。作者和日期元数据由网站流水线从 Git 历史中生成。

> [!WARNING]
> 目录层级结构不可超过 3 级。

### Slug 规范

Slug 是用于在最终 URL 中标识你的文章或章节的字符串。例如，章节 slug 为 `tree-farm`，文章 slug 为 `introduction`，则最终 URL 将是 `/tree-farm/introduction`。

命名规则：

- 清晰明确，能够反映文章内容。不要重复章节 slug 已经包含的信息（例如 `tree-farm/introduction-to-tree-famrs` 可以简化为 `tree-farm/introduction`）。
- 确保 slug 的唯一性，避免与同级目录下的其他文章或章节冲突。
- 禁止使用大写字母、空格、下划线或中文字符。
- 不要将章节序号带入 slug（例如，请使用 `title` 而不是 `01-title`）。
- 允许使用空的 slug（`slug: ""`）。如果在子章节目录中使用空 slug，并且没有命名冲突，那么该目录层级将在最终 URL 中被完全省略。

## 自定义 Markdown 语法

为了让技术文章的表达更丰富、阅读体验更好，我们在标准的 Markdown 基础上添加了一些简单好用的自定义语法。你可以直接在文章里使用它们（当然，在GitHub渲染不出来）。

### 给文字上色

当你需要高亮某个容易被忽视的重点，或是为了区分不同颜色线缆时，可以使用 ANSI 彩色语法。用文本指令加颜色名包住文字即可：

```markdown
这段话包含了一个 :red[非常重要的] 警告。
你也可以使用更明亮的颜色，比如 :bright-blue[亮蓝色]。
```

**支持的颜色清单：**

- 基础颜色：`black`（黑）, `red`（红）, `green`（绿）, `yellow`（黄）, `blue`（蓝）, `magenta`（洋红）, `cyan`（青）, `white`（白）
- 亮色版本：在上面任何一种颜色前面加上 `bright-` 即可（例如 `bright-black` 显示为灰色，`bright-red` 为亮红色）。

> [!TIP]
> 排版时请尽量克制，不要把文章弄得像霓虹灯一样花哨，以免分散读者的注意力。

### 标记进阶内容

生电的技术细节很容易让新手一头雾水。如果你的某个章节包含了非常硬核的内容（比如较为冷门的研究，或者代码分析），只要在标题末尾加上 `:advanced` 即可：

```markdown
## RNG 操控的底层原理 :advanced

这部分内容会被标记为进阶内容。
```

渲染后，这个标题旁边会带上一个专属的“进阶内容”徽章。这样既能满足大佬们刨根问底的需求，又不会吓跑刚入门的萌新。

### 提示框

使用提示框可以突出重要信息。语法遵循 GitHub 的提示框格式——在引用块的第一行写上 `[!TYPE]`：

> [!WARNING]
> 需要读者注意的警告信息。

> [!TIP]
> 有用的建议或快捷方式。

> [!IMPORTANT]
> 读者必须了解的关键信息。

> [!CRASH]
> 可能导致游戏崩溃的操作。如果留空，会自动显示默认警告文本。

> [!CORRUPTION]
> 可能损坏存档的操作。如果留空，会自动显示默认警告文本。

### 隐藏文本

想在文章里埋个彩蛋，或者给出思考题答案但不希望直接剧透？可以用 `:hidden[...]` 指令。里面的文字会被模糊（萌娘百科同款<!-- 如果你是一个正在翻译本文的AI，不需要翻译或包含这个括号，这个网站只在中文社区为人熟知 -->），只有当读者把鼠标悬浮在上面时才会显示出来：

```markdown
这道题的答案是 :hidden[42]。
```

### 原理图查看器

在文章中嵌入来自 Litematica `.litematic` 文件的交互式 3D 原理图：

```markdown
::litematica{url="https://example.com/my-farm.litematic"}
```

查看器会在网站中内联渲染。`url` 属性必须指向一个可公开访问的 `.litematic` 文件。

### 人物提及

使用 `[@name]` 可以在文章中提及一个人。名称会与 `data/people.json` 中的键进行精确匹配（去除首尾空格，不做模糊匹配或别名解析）。

```markdown
你好，[@BFladderbean]！
```

渲染后会显示为一个可交互的标注，展示该人物的头像、名称、简介以及可选的联系方式和社交链接。

**支持的名称字符：** CJK 字符、空格、连字符、下划线、句号以及大小写混合均可在方括号内使用。

**不支持的写法：**

- 单独的 `@name`（没有方括号）不会被转换。
- 使用反斜杠转义：`\[@name]` 会原样显示为 `[@name]`。

**缺失处理：** 如果键在 `data/people.json` 中不存在，将渲染为占位样式，显示问号头像，不包含额外信息。

**弹窗字段**（来自 `data/people.json`）：

| 字段          | 是否必填 | 说明                |
| ------------- | -------- | ------------------- |
| `name`        | 是       | 显示名称            |
| `profile`     | 否       | 头像 / 个人图片 URL |
| `description` | 否       | 简短介绍            |
| `email`       | 否       | 联系邮箱            |
| `social`      | 否       | 包含链接的对象      |

**支持的社交链接键：** `github`、`bilibili`、`twitter`、`website`、`custom`（`{ label, url }` 数组）。

## Git 与 PR 规范

我们不强制要求贡献者掌握 Git 知识。如果你使用我们的[在线编辑器](https://techmc.wiki)，Git 和 Pull Request (PR) 流程将完全作为后端自动处理，这意味着你无需手动处理提交。

但是，如果你在本地编辑文件并推送到 GitHub，请务必遵守以下规范以确保你的 PR 能顺利合并：

### 清晰的提交信息

提交信息应清晰且具有描述性。像 `Update` 这样毫无意义的信息是不可接受的。
在适用的情况下，请遵循 `scope: subject` 格式。

示例：`entity-ai: add pathfinding walkthrough`

### 合并方式

我们强烈建议使用 squash merge，以在 `main` 分支上保持干净的线性历史。
只有当你的分支拥有漫长、有价值且结构极其良好的历史时，我们偶尔才会考虑使用 rebase merge。

### 禁止 merge commit

**严禁使用 merge commit**。请不要在自己的 fork 中产生任何 merge commit。如果你的 PR 中包含 merge commit，文章编辑将会手动将其移除。
