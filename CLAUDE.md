# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Jekyll blog (Korean-language) by 김수명, served at https://sumyeong.kim (custom domain via `CNAME`, also reachable at lifespank.github.io). Built on the **minimal-mistakes-jekyll** theme installed as a gem (not `remote_theme` — see commit `1ce184d`). The repo's purpose is content: most changes are new posts under `_posts/`, not code.

## Common commands

```sh
bundle install              # first-time setup / after Gemfile changes
bundle exec jekyll serve    # local preview at http://localhost:4000 with live reload
bundle exec jekyll build    # one-shot build into _site/
```

`_site/` is the generated output — never hand-edit it. Windows file watching uses the `wdm` gem (already in `Gemfile`).

## Authoring posts

Posts live in `_posts/` and **must** be named `YYYY-MM-DD-<title>.md` (`.markdown` also works). Required front matter pattern, mirroring existing posts:

```yaml
---
title: "제목"
date: 2026-05-03 15:53 +0900     # always +0900 (Asia/Seoul, set in _config.yml)
categories: ['윈도우 드라이버']    # use one of the established categories
tags: 윈도우 드라이버              # space-separated
---
```

Layout/sidebar/comments/sharing defaults are applied automatically by the `defaults` block in `_config.yml` — do **not** repeat `layout: single`, `author_profile`, etc. in post front matter.

### Categories must match the sidebar nav

`_data/navigation.yml` (`cats:`) hard-codes the sidebar categories with anchor links to `/categories/#<slug>`. The established set is:

- 프로그래밍 문제
- 프로젝트
- 안드로이드
- 알고리즘 및 자료구조
- 윈도우 드라이버
- 여러가지

Using a category not in this list will create an orphaned `/categories/#...` page that the sidebar doesn't link to. If a new category is genuinely needed, add it to `_data/navigation.yml` in the same commit.

### Cross-linking & images

- Link to other posts with Liquid: `{% post_url 2026-02-16-예물 시계 구매하기 %}` (filename without extension, exact spaces preserved).
- Images go in `/images/` at repo root and are referenced via `{{ "/images/foo.png" | relative_url }}` — do not hardcode the site URL.

## Site architecture quirks worth knowing

- **Theme overrides are minimal and intentional.** `_includes/head/custom.html` injects `css/custom.css` (the theme's documented hook for custom styles), and `_includes/scripts.html` is a copy of the theme's scripts file with a MathJax 2.7.5 snippet appended at the bottom — so `$...$` and `$$...$$` math works in any post. If you regenerate `scripts.html` from the theme upstream, re-add the MathJax block.
- **Search, comments, locale:** lunr search is on, Disqus shortname is `sumyeongkim`, locale is `ko`. These are configured in `_config.yml`; no extra wiring needed in posts.
- **Plugins** are limited to `jekyll-feed`, `jekyll-include-cache`, `jekyll-sitemap`. The site does not use `github-pages` gem — Jekyll 4.x is used directly, so plugin compatibility is not constrained by GitHub Pages' allow-list.
- **`Gemfile.bak`** is kept around as a snapshot of the prior `remote_theme` setup; leave it unless explicitly cleaning up.
- **Top-level `*.html` / `*.md` pages** (`about.markdown`, `category-archive.md`, `tag-archive.md`, `404.html`, `love42_*`, `naver*.html`, `robots.txt`) are intentional — they're either site pages, archive views, app legal pages, or webmaster verification files. Don't treat them as stray.
