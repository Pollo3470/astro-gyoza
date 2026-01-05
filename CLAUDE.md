# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Gyoza is a static blog template built with Astro 4.6 and React. It features dark/light themes, SEO optimization, RSS feeds, Waline comments, and full-text search via Pagefind.

## Commands

| Command            | Description                                |
| ------------------ | ------------------------------------------ |
| `pnpm dev`         | Start dev server at localhost:4321         |
| `pnpm build`       | Type check, build, and index with Pagefind |
| `pnpm preview`     | Preview production build                   |
| `pnpm lint`        | Format code with Prettier                  |
| `pnpm new-post`    | Create a new blog post interactively       |
| `pnpm new-friend`  | Add a friend link                          |
| `pnpm new-project` | Add a project                              |

After cloning, run `pnpm i` which also installs Playwright Chromium (required for Mermaid diagram rendering).

## Architecture

**Framework Stack:** Astro + React + Tailwind CSS + Jotai (state) + Framer Motion (animations)

**Content Collections** (in `src/content/`):

- `posts/` - Blog posts in Markdown with frontmatter (title, date, category, tags, sticky, draft)
- `projects/` - Project showcase (JSON)
- `friends/` - Friend links (JSON)
- `spec/` - Special pages like about, friends, projects (Markdown)

**Site Configuration:** All site settings (title, author, menus, colors, analytics) are in `src/config.json`

**Markdown Pipeline** (custom plugins in `src/plugins/`):

- Remark: reading time, math (KaTeX), embeds, spoilers
- Rehype: code highlighting (Shiki), Mermaid diagrams, heading anchors, image/link processing

**State Management:** Jotai atoms in `src/store/` for theme, viewport, scroll info, and modal stack

**Page Transitions:** Swup handles smooth transitions between pages

## Content Frontmatter Schema

Posts support these frontmatter fields:

```yaml
title: string (required)
date: date (required)
lastMod: date (optional)
summary: string (optional)
cover: string (optional)
category: string (optional)
tags: string[] (default: [])
comments: boolean (default: true)
draft: boolean (default: false)
sticky: number (default: 0, higher = more sticky)
```

## Git Workflow

- Conventional commits enforced via commitlint
- Pre-commit hook runs Prettier via lint-staged
- Deploy to Vercel via GitHub Actions on push to main
