# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal tech blog (ethan6077.github.io) built with Jekyll and hosted on GitHub Pages. Based on the "Forever Jekyll" theme. Content focuses on Scala/functional programming topics.

## Development Commands

```bash
# Install dependencies
bundle install

# Serve locally with live reload
bundle exec jekyll serve

# Build static site (output to _site/)
bundle exec jekyll build
```

Local site runs at http://localhost:4000 by default.

## Writing Blog Posts

Posts go in `_posts/` with filename format: `YYYY-MM-DD-slug-title.md`

Required front matter:
```yaml
---
layout: post
title: Post Title Here
categories: [category1, category2]
---
```

Use `<!--more-->` to mark the excerpt boundary for the index page.

## Architecture

- **Theme**: Forever Jekyll (vendored, not a gem theme — layouts/includes/sass are all local)
- **Layouts**: `_layouts/default.html` → `_layouts/post.html` / `_layouts/page.html`
- **Styling**: `style.scss` imports partials from `_sass/`
- **Config**: `_config.yml` controls site metadata, navigation, footer links, plugins
- **Pagination**: 5 posts per page via `jekyll-paginate`
- **Markdown**: Kramdown with GFM input and Rouge syntax highlighting
