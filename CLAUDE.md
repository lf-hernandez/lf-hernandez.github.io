# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Personal blog and portfolio site at https://lfhernandez.com, built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme (included as a git submodule). Deployed automatically to GitHub Pages on every push to `main`.

## Commands

**Local development server:**
```bash
hugo server -D
```

**Production build** (mirrors CI):
```bash
hugo --gc --minify --baseURL "https://lfhernandez.com/"
```

Hugo extended version 0.160.1 is required (extended for Dart Sass support). No npm or other dependencies are needed.

After cloning, initialize the PaperMod submodule:
```bash
git submodule update --init --recursive
```

## Content

Posts live in `content/posts/` as Markdown files. Front matter format:

```yaml
---
title: "Post Title"
date: 2024-04-30T22:16:22-04:00
summary: "Short description shown in listings"
tags: ["tag1"]
categories: ["category1"]
---
```

- `buildDrafts: false` — set `draft: true` in front matter to prevent a post from being published
- The timezone used in CI is `America/Los_Angeles`; dates in front matter should match expected publish ordering

Special pages (`about.md`, `search.md`, `archives.md`) use dedicated Hugo layouts configured in `hugo.yaml` and should not be moved.

## Custom Shortcodes

`layouts/shortcodes/unsafe.html` — injects raw HTML. Used for embedding Credly badge widgets. Syntax:

```
{{< unsafe >}}
<raw html here>
{{< /unsafe >}}
```

## Theme

PaperMod is pinned as a git submodule at `themes/PaperMod`. To update it:
```bash
cd themes/PaperMod && git pull
```

Theme features in use: client-side search (Fuse.js), reading time, table of contents, code copy buttons, breadcrumbs, archives, auto dark/light mode.

## Commit Convention

Recent commits follow the pattern `<type>: <description>` — types seen in history: `ci`, `theme`, `content`, `post`, `ci/cd`.

## Deployment

GitHub Actions (`.github/workflows/hugo.yaml`) builds and deploys to GitHub Pages automatically on push to `main`. No manual deploy step is needed.
