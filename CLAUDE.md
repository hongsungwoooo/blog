# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Jekyll blog hosted on GitHub Pages using the **minima** theme. Blog content is written in Korean. Deployed to `https://hongsungwoooo.github.io/blog/`.

## Build & Serve

```bash
bundle install          # Install dependencies
bundle exec jekyll serve  # Local dev server at http://localhost:4000/blog/
bundle exec jekyll build  # Build static site to _site/
```

Requires Ruby and Bundler. Uses the `github-pages` gem for GitHub Pages compatibility.

## Blog Post Conventions

- Posts go in `_posts/` with filename format: `YYYY-MM-DD-slug.md`
- Front matter requires: `layout: post`, `title`, `date`, `categories`, `tags`
- Use `published: false` to mark drafts
- Series posts use bracket prefix in title, e.g. `[Server 시리즈 1-1]`
- Permalink pattern: `/:year/:month/:day/:title/` (configured in `_config.yml`)

## Configuration

- `_config.yml`: Site title, URL (`https://hongsungwoooo.github.io`), baseurl (`/blog`), theme, plugins
- Plugins: `jekyll-feed`, `jekyll-seo-tag`
