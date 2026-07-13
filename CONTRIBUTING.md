# Submitting an Article

[中文版](CONTRIBUTING_CN.md)

## What is this repository?

This repository securely stores all the article content for the Graduate Texts in Minecraft project. Submissions and revisions of articles are processed here.

## Administration & Roles

This project adopts a flat organizational structure. There are only two roles: reviewers (admins) and writers (users).
Reviewers rotate among the internal writer team. There is no hierarchical difference between an outside contributor and a writer.

Reviewers are responsible for:

- Enforcing writing standards
- Fact checks
- Resolving conflicts on pull requests
- Approving pull requests (PRs)

## Submission Workflow

Submitting an article is straightforward:

1. Submit via the website or GitHub to request a review. You can operate directly on the website, or edit locally and push to GitHub.
2. Fill out the frontmatter (the metadata block) at the top of your markdown file.
3. Write your article using standard Markdown alongside our supported custom syntax (see below).
4. Submit a Pull Request (PR) through the website or GitHub to request a review.

## Frontmatter Guide

Every Markdown file requires a YAML frontmatter block at the very top of the file. Source articles and translations use different frontmatter shapes.

Source articles (`*.zh.md`) define their own URL and source-language metadata:

```yaml
---
slug: basics-and-structure
title: 前置知识与树场的基本结构
description: Optional short summary
index: 1
is-advanced: false
banner:
  src: img/banner.png
  alt: A descriptive alt text
---
```

Source field breakdown:

- `slug`: The unique URL identifier for the article.
- `title`: The primary source-language title of the article.
- `description`: Optional short summary used by previews and metadata.
- `index`: An integer representing the order of the article in its directory. `-1` means unordered, please do not submit articles with `index: -1`.
- `appendix`: Optional. Set to `true` to display this article after an appendix separator and use appendix numbering. The filename, title, and slug are not used for detection.
- `is-advanced`: Set to `true` to mark the _entire_ article as advanced content.
- `banner`: A nested object containing `src` (relative image path) and `alt` (accessible description of the image).

Translations (`*.en.md`) link back to their source file and source revision:

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

Translation field breakdown:

- `translates`: Relative path to the source article this file translates.
- `translated-from-revision`: Git commit SHA of the source revision used for the translation.
- `title`, `description`, and `banner`: Optional translated overrides.

Source chapter/directory `README.zh.md` files use the same source rules, with chapter-specific titles:

```yaml
---
slug: tree-farm
chapter-title: 树场
intro-title: 前言
index: -1
appendix: true
---
```

Translated chapter/directory `README.en.md` files use the translation shape:

```yaml
---
translates: ./README.zh.md
translated-from-revision: 365edcc68df67b53fc5dfada26eddfd5a8e69a02
chapter-title: Tree Farm
intro-title: Foreword
---
```

Chapter field breakdown:

- `chapter-title`: The display name for the chapter/directory.
- `intro-title`: The title for the introductory text inside the `README.md`.
- `index`: Ordering integer for the chapter among its siblings.
- `slug`: The unique URL identifier for the chapter directory. In nested chapters, this field can be left empty to omit the URL level.
- `appendix`: Optional. Set to `true` on a source chapter `README.zh.md` to display an appendix separator and apply appendix treatment to every descendant. A standalone appendix sets the same field in its own source article. Multiple appendices are allowed, and detection never depends on the filename, title, or slug. Translations inherit the field from their source file.

Do not add legacy fields such as `title-en`, `chapter-title-en`, `intro-title-en`, `author`, `co-authors`, `date`, or `lastmod`. Author and date metadata is derived from Git history by the website pipeline.

> [!WARNING]
> The directory structure must not exceed 3 levels of depth.

### Slug Rules

A "slug" is the string used to identify your article or chapter in the final URL. For example, if the chapter slug is `tree-farm` and the article slug is `introduction`, the final URL will be `/tree-farm/introduction`.

Naming Rules:

- Be clear and explicit, reflecting the content of the article. Do not repeat information already included in the chapter slug (e.g., `tree-farm/introduction-to-tree-farms` can be simplified to `tree-farm/introduction`).
- Ensure the uniqueness of the slug to avoid conflicts with other articles or chapters in the same directory.
- Do not use uppercase letters, spaces, underscores, or Chinese characters.
- Do not include chapter or ordering numbers in the slug (e.g., use `title` instead of `01-title`).
- Empty slugs (`slug: ""`) are permitted. If used on a subsection directory, that directory level will be omitted from the final URL entirely (assuming there are no conflicts).

## Custom Markdown Syntax

We provide a few custom Markdown enhancements to make technical writing more expressive and engaging. You can use these special tags directly in your articles (though, of course, they won't render on GitHub).

### Colored Text

When you need to highlight an easy-to-miss warning or distinguish different colored cables, you can use our ANSI color syntax. Wrap your text in a text directive with the color name:

```markdown
This is a :red[very important] warning.
You can also use :bright-blue[brighter colors].
```

**Supported Colors:**

- Standard: `black`, `red`, `green`, `yellow`, `blue`, `magenta`, `cyan`, `white`
- Bright variants: `bright-black` (gray), `bright-red`, `bright-green`, `bright-yellow`, `bright-blue`, `bright-magenta`, `bright-cyan`, `bright-white`

> [!TIP]
> Use colors with restraint—otherwise the page starts to look like a neon sign.

### Advanced Content Sections

Deep technical mechanics can easily overwhelm beginners. If a section contains very hardcore content (such as relatively obscure research or code analysis), append `:advanced` to the heading:

```markdown
## How the RNG Manipulation Works :advanced

This section will be marked as advanced content.
```

When rendered, this heading will get a special "Advanced" badge. This keeps the article approachable for beginners while satisfying experts who want to dig deeper.

### Callout Blocks

Use callout blocks to draw attention to important information. The syntax follows GitHub's alert format — a blockquote whose first line is `[!TYPE]`:

> [!WARNING]
> Something the reader should be cautious about.

> [!TIP]
> A helpful suggestion or shortcut.

> [!IMPORTANT]
> Critical information the reader must not miss.

> [!CRASH]
> Steps that may cause the game to crash. If left empty, a default warning is shown automatically.

> [!CORRUPTION]
> Steps that may corrupt save files. If left empty, a default warning is shown automatically.

### Hidden Text

Want to hide an easter egg or the answer to a puzzle without spoiling it immediately? Use the `:hidden[...]` directive. The text will be blacked out, revealing itself only when the reader hovers over it:

```markdown
The answer to the puzzle is :hidden[42].
```

### Schematic Viewer

Embed an interactive 3D schematic from a Litematica `.litematic` file:

```markdown
::litematica{url="https://example.com/my-farm.litematic"}
```

The viewer renders inline on the website. The `url` attribute must point to a publicly accessible `.litematic` file.

### People Mentions

Use `[@name]` to mention a person in your article. The name is matched directly against keys in `data/people.json` (trimmed, exact match only — no fuzzy matching or aliases).

```markdown
Hello [@BFladderbean]!
```

This renders as an interactive annotation showing the person's profile image, name, description, and optional contact/social links.

**Supported name characters:** CJK characters, spaces, hyphens, underscores, periods, and mixed case are all valid inside the brackets.

**Not supported:**

- Bare `@name` without brackets is not transformed.
- Use a backslash to escape: `\[@name]` renders as literal `[@name]`.

**Fallback behavior:** If the key does not exist in `data/people.json`, a placeholder is rendered with a question-mark avatar and no additional details.

**Popup fields** (from `data/people.json`):

| Field         | Required | Description                |
| ------------- | -------- | -------------------------- |
| `name`        | Yes      | Display name               |
| `profile`     | No       | Avatar / profile image URL |
| `description` | No       | Short bio                  |
| `email`       | No       | Contact email              |
| `social`      | No       | Object with links          |

**Supported social keys:** `github`, `bilibili`, `twitter`, `website`, `custom` (array of `{ label, url }`).

## Git Standards

We do not strictly require contributors to possess Git knowledge. If you use our [online editor](https://techmc.wiki), Git and Pull Requests are handled entirely as the backend, meaning you won't need to manually deal with commits.

However, if you are editing files locally and pushing to GitHub, you must follow these standards to ensure your PR merges smoothly:

### Clear Commit Messages

Commit messages should be clear and descriptive. A generic message like `Update` is not acceptable.
Whenever applicable, follow the `scope: subject` format.

Example: `entity-ai: add pathfinding walkthrough`.

### Merging

We strongly prefer squash merges to maintain a clean and linear history on the `main` branch.
Rebase merges are only considered occasionally if your branch possesses a very long, valuable, and exceptionally well-structured history.

### No Merge Commits

**Merge commits are strictly prohibited**. Do NOT create any merge commits on your own fork. If your PR includes a merge commit, reviewers will remove it.
