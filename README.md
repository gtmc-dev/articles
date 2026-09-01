---
skip: true
---

<!-- prettier-ignore -->
<div align="center">

# GTMC Articles

**Article storage for [Graduate Texts in Minecraft](https://techmc.wiki).**

Data source for GTMC chapters, proposed drafts, and translations on the website.

[![Website](https://img.shields.io/badge/site-techmc.wiki-60708F?style=flat-square&labelColor=4A5A78)](https://techmc.wiki) [![Articles](https://img.shields.io/badge/license-CC--BY--NC--SA%204.0-lightgrey?style=flat-square)](LICENSE) [![Markdown](https://img.shields.io/badge/Markdown-GFM-000?style=flat-square&logo=markdown&logoColor=white)](https://github.github.com/gfm/) [![Languages](https://img.shields.io/badge/lang-zh%20%7C%20en-60708F?style=flat-square&labelColor=4A5A78)](#translation)

[Read on the Site](https://techmc.wiki) · [Contributing](CONTRIBUTING.md) · [Roadmap](ROADMAP.md) · [Reviewers](REVIEWERS.md)

<!-- README-I18N:START -->

**English** | [中文](./README.zh.md)

<!-- README-I18N:END -->

</div>

---

## About

This repository holds every article in the GTMC textbook. It is a _data repo_ — pure Markdown plus YAML frontmatter, organized into chapter directories. The [website](https://github.com/techmc-wiki/gtmc) consumes this repo as a Git submodule and renders it through its content pipeline.

> [!NOTE]
> Looking for the website code, the article reader, or the draft editor? Those live in [`techmc-wiki/gtmc`](https://github.com/techmc-wiki/gtmc).

## Submitting an article

You have two paths, both ending in a Pull Request against this repo:

- `>> ON THE SITE` — sign in at [techmc.wiki](https://techmc.wiki), draft in the in-browser editor, and submit. The site opens and syncs a PR here for you.
- `>> LOCAL + GIT` — fork, write Markdown with the right frontmatter, push, and open a PR by hand.

Reviewers approve, request changes, or close the PR. Approval merges the article and it goes live on the next site deploy.

> [!TIP]
> Full submission flow, frontmatter schema, slug rules, and our custom Markdown extensions (callouts, colored text, `:hidden[...]`, `[@name]` mentions, `:advanced` headings, `::litematica{...}` schematics) are documented in [`CONTRIBUTING.md`](CONTRIBUTING.md). Read it before your first PR.

## Repository layout

Each top-level directory is a **chapter**. Inside a chapter, articles are plain Markdown files; subdirectories are sub-chapters (max **3 levels deep**). Every directory has a `README.zh.md` (source) and `README.en.md` (translation) describing the chapter itself.

For the curriculum vision and the v2 release roadmap, see [`ROADMAP.md`](ROADMAP.md).

## Source and translation model

Every article exists as a **source file** (currently `*.zh.md`) plus optional **translations** (`*.en.md`). Frontmatter shape differs between the two:

- **Source** files own the article — `slug`, `title`, `index`, `is-advanced`, `banner`.
- **Translation** files link back to the source — `translates`, `translated-from-revision`, plus translated `title` / `description` / `banner` overrides.

A minimal source frontmatter:

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

A minimal translation frontmatter:

```yaml
---
translates: ./前置知识与树场的基本结构.zh.md
translated-from-revision: 20ad46927f7ac8db7b9c875504c9a899209214af
title: Basics and Structure
---
```

Author and date metadata is **derived from Git history** by the website pipeline — do not add `author`, `date`, or `lastmod` fields. Full schema reference, slug rules, and chapter `README.*.md` shapes are in [`CONTRIBUTING.md`](CONTRIBUTING.md#frontmatter-guide).

## Custom Markdown

On top of GFM, articles support a small set of extensions tailored for technical writing:

| Feature           | Syntax                                                                 |
| ----------------- | ---------------------------------------------------------------------- |
| Colored text      | `:red[text]`, `:bright-blue[text]`                                     |
| Advanced sections | `## Heading :advanced`                                                 |
| Callouts          | `> [!WARNING]`, `[!TIP]`, `[!IMPORTANT]`, `[!CRASH]`, `[!CORRUPTION]`  |
| Hidden text       | `:hidden[spoiler]`                                                     |
| Schematic viewer  | `::litematica{url="https://..."}`                                      |
| People mentions   | `[@BFladderbean]` (resolved against `data/people.json` on the website) |
| Math              | `$inline$`, `$$display$$` (KaTeX)                                      |
| Code highlighting | Standard fenced code blocks (Shiki on the site)                        |

These render on the website but stay readable on GitHub.

## Roles and review

GTMC runs flat: there are **reviewers** and **writers**. Reviewers rotate from the writer team and are responsible for writing standards, fact checks, and merging PRs. Outside contributors and writers are treated identically.

See [`REVIEWERS.md`](REVIEWERS.md) for the active reviewer roster and [`CODE_OF_CONDUCT.zh.md`](CODE_OF_CONDUCT.zh.md) for community norms.

## Git rules

If you draft on the site, the editor handles Git for you. If you push from the command line, follow these conventions so PRs merge cleanly:

- **Conventional-style commits** — `scope: subject`. Example: `entity-ai: add pathfinding walkthrough`. Generic messages like `Update` are not accepted.
- **Squash merges by default** — keeps `main` linear. Rebase merges are reserved for branches with exceptionally well-structured history.
- **No merge commits** — strictly prohibited on contributor branches. Reviewers will remove them.

Full Git guidance lives in [`CONTRIBUTING.md`](CONTRIBUTING.md#git-standards).

## Translation

Until **2026-05-17**, all source articles were written in Chinese and machine-translated into English by Claude Sonnet 4.5 via the [`techmc-wiki/translation`](https://github.com/techmc-wiki/translation) workflow, then tone-revised by MiMo-V2-Pro. The most recent translations track source revision `b881ea81265ca498947f135afd049f52ebd8440b`.

After that date, translations are still pinned to specific source revisions via the `translated-from-revision` field — the website surfaces a "translation may be stale" hint when the source has moved on.

## See also

- [`techmc-wiki/gtmc`](https://github.com/techmc-wiki/gtmc) — the website that renders this content.
- [`techmc-wiki/translation`](https://github.com/techmc-wiki/translation) — the translation workflow.
- [Other GTMC repos](https://github.com/orgs/techmc-wiki/repositories).

---

<div align="center">

<sub>
Articles: <a href="https://creativecommons.org/licenses/by-nc-sa/4.0/">CC BY-NC-SA 4.0</a>
</sub>

</div>
