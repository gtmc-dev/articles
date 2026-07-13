<!-- prettier-ignore -->
<div align="center">

# GTMC Articles

**[Graduate Texts in Minecraft](https://techmc.wiki) 的文章存储仓库。**

GTMC 网站章节、待审草稿与翻译的数据源。

[![Website](https://img.shields.io/badge/site-techmc.wiki-60708F?style=flat-square&labelColor=4A5A78)](https://techmc.wiki) [![Articles](https://img.shields.io/badge/license-CC--BY--NC--SA%204.0-lightgrey?style=flat-square)](LICENSE) [![Markdown](https://img.shields.io/badge/Markdown-GFM-000?style=flat-square&logo=markdown&logoColor=white)](https://github.github.com/gfm/) [![Languages](https://img.shields.io/badge/lang-zh%20%7C%20en-60708F?style=flat-square&labelColor=4A5A78)](#翻译)

[在网站上阅读](https://techmc.wiki) · [贡献指南](CONTRIBUTING_CN.md) · [路线图](ROADMAP.md) · [评审者](REVIEWERS_CN.zh.md)

<!-- README-I18N:START -->

[English](./README.md) | **汉语**

<!-- README-I18N:END -->

</div>

---

## 关于

本仓库存放 GTMC 教科书的全部文章。它是一个**内容仓库**——纯 Markdown 加 YAML frontmatter，按章节目录组织。[网站仓库](https://github.com/gtmc-dev/gtmc)以 Git 子模块的方式引入本仓库，并通过其内容流水线进行渲染。

> [!NOTE]
> 寻找网站代码、文章阅读器或草稿编辑器？它们在 [`gtmc-dev/gtmc`](https://github.com/gtmc-dev/gtmc)。

## 投稿

有两条路径，最终都会向本仓库发起一个 Pull Request：

- `>> ON THE SITE` —— 在 [techmc.wiki](https://techmc.wiki) 登录，使用站内编辑器撰写并提交。网站会自动开 PR 并保持同步。
- `>> LOCAL + GIT` —— Fork 仓库，本地以正确的 frontmatter 撰写 Markdown，推送后手动开 PR。

评审者会批准、请求修改或关闭 PR。批准并合并后，文章会随下一次站点部署上线。

> [!TIP]
> 完整的投稿流程、frontmatter Schema、slug 规则，以及我们的 Markdown 扩展（callout、彩色文本、`<hidden>`、`[@name]` 提及、`[!ADVANCED]` 标题）均记录在 [`CONTRIBUTING_CN.md`](CONTRIBUTING_CN.md) 中。首次投稿前请先阅读。

## 仓库结构

每个一级目录是一个**章节**。章节内的文章是普通 Markdown 文件；子目录则是子章节（最多嵌套 **3 级**）。每个目录都有 `README.zh.md`（源语言）与 `README.en.md`（译文）来描述章节本身。

完整的课程蓝图与 Beta 之后的计划，见 [`ROADMAP.md`](ROADMAP.md)。

## 源文与译文模型

每篇文章包含一个**源文件**（目前是 `*.zh.md`）以及可选的**译文**（`*.en.md`）。两者的 frontmatter 形态不同：

- **源文件** 拥有文章本体——`slug`、`title`、`index`、`is-advanced`、`banner`。
- **译文** 通过 `translates`、`translated-from-revision` 字段链回源文件，并可覆盖译文版本的 `title` / `description` / `banner`。

源文件的最小化 frontmatter：

```yaml
---
slug: basics-and-structure
title: 前置知识与树场的基本结构
index: 1
is-advanced: false
banner:
  src: img/banner.png
  alt: A descriptive alt text
---
```

译文的最小化 frontmatter：

```yaml
---
translates: ./前置知识与树场的基本结构.zh.md
translated-from-revision: 20ad46927f7ac8db7b9c875504c9a899209214af
title: Basics and Structure
---
```

作者与时间信息**由网站流水线从 Git 历史中推导**——请勿添加 `author`、`date`、`lastmod` 等字段。完整 Schema、slug 规则、章节 `README.*.md` 形态见 [`CONTRIBUTING_CN.md`](CONTRIBUTING_CN.md)。

## Markdown 扩展

在 GFM 之上，我们为技术写作提供少量扩展：

| 功能         | 语法                                                                  |
| ------------ | --------------------------------------------------------------------- |
| 彩色文本     | `[red]text[/red]`、`[bright-blue]text[/bright-blue]`                  |
| 进阶段落标记 | `## 标题 [!ADVANCED]`                                                 |
| Callout      | `> [!WARNING]`、`[!TIP]`、`[!IMPORTANT]`、`[!CRASH]`、`[!CORRUPTION]` |
| 隐藏文本     | `<hidden>spoiler</hidden>`                                            |
| 人物提及     | `[@BFladderbean]`（在网站上根据 `data/people.json` 解析）             |
| 数学公式     | `$inline$`、`$$display$$`（KaTeX）                                    |
| 代码高亮     | 标准代码栅栏（站内由 Shiki 渲染）                                     |

这些语法在网站上会被渲染，同时仍保持在 GitHub 上的可读性。

## 角色与评审

GTMC 采用扁平结构：只有**评审者**与**作者**两类角色。评审者从作者团队中轮值产生，负责把控写作规范、事实核查并合并 PR。外部贡献者与作者享受同等待遇。

当前评审者名单见 [`REVIEWERS_CN.zh.md`](REVIEWERS_CN.zh.md)，社区行为准则见 [`CODE_OF_CONDUCT_CN.zh.md`](CODE_OF_CONDUCT_CN.zh.md)。

## Git 规范

如果你在网站上撰稿，编辑器会替你处理 Git。如果选择从命令行推送，请遵循以下规范，便于 PR 顺利合入：

- **遵循 Conventional 风格的提交信息** —— `scope: subject`。例：`entity-ai: add pathfinding walkthrough`。`Update` 之类的笼统信息不被接受。
- **默认 squash 合并** —— 让 `main` 保持线性历史。Rebase 合并仅在分支具备格式优良且有价值的历史时才考虑。
- **禁止 merge commit** —— 贡献者分支严格禁止 merge commit。出现的话评审者会替你移除。

完整的 Git 规范见 [`CONTRIBUTING_CN.md`](CONTRIBUTING_CN.md)。

## 翻译

**2026 年 5 月 17 日**之前，所有源文均以中文撰写，再经 Claude Sonnet 4.5 通过 [`gtmc-dev/translation`](https://github.com/gtmc-dev/translation) 工作流机器翻译为英文，并由 MiMo-V2-Pro 进行语气润色。最近一批译文对齐的源文提交为 `b881ea81265ca498947f135afd049f52ebd8440b`。

此日期之后的译文仍通过 `translated-from-revision` 字段绑定到具体的源文提交——当源文已变化时，网站会提示读者「译文可能已过时」。

## 另见

- [`gtmc-dev/gtmc`](https://github.com/gtmc-dev/gtmc) —— 渲染本内容的网站。
- [`gtmc-dev/translation`](https://github.com/gtmc-dev/translation) —— 翻译工作流。
- [其他 GTMC 仓库](https://github.com/orgs/gtmc-dev/repositories)。

---

<div align="center">

<sub>
文章：<a href="https://creativecommons.org/licenses/by-nc-sa/4.0/">CC BY-NC-SA 4.0</a>
</sub>

</div>
