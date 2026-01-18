# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal blog/website built with Jekyll and hosted on GitHub Pages at mlumiste.com. The site uses the [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) theme (via `remote_theme`) with custom overrides and styling.

## Build & Development Commands

### Local Development
```bash
# Install Ruby dependencies
bundle install

# Install Node.js dependencies (for JS minification)
npm install

# Serve the site locally (standard Jekyll)
bundle exec jekyll serve

# Serve with live reload using Rake task
bundle exec rake preview

# Build the site
bundle exec jekyll build
```

### JavaScript Build
```bash
# Uglify and minify JavaScript assets
npm run build:js

# Watch for JS changes and rebuild
npm run watch:js
```

The JS build process concatenates and minifies vendor libraries and custom scripts from `assets/js/` into `assets/js/main.min.js`.

## Repository Structure

### Content Organization

- **`_posts/`** - Blog posts in Markdown with YAML front matter. File naming: `YYYY-MM-DD-title.md`
- **`_layouts/`** - Custom Jekyll layouts that override or extend Minimal Mistakes layouts
- **`_includes/`** - Partial templates for reusable components
- **`_sass/`** - Custom Sass/SCSS styles organized under `minimal-mistakes/` subdirectory
- **`assets/`** - Static assets (images, CSS, JS)
- **`_data/navigation.yml`** - Site navigation menu configuration

### Configuration

- **`_config.yml`** - Main Jekyll configuration with:
  - Site metadata (title, description, author)
  - Minimal Mistakes theme settings and skin (`custom`)
  - Google Analytics (gtag) integration
  - Comment system configuration (Staticman with custom provider)
  - Default post settings (layout, author profile, read time, etc.)
  - Plugin configuration (jekyll-paginate, jekyll-sitemap, jekyll-feed, etc.)

- **`staticman.yml`** - Configuration for Staticman comment system (moderation enabled, reCAPTCHA v2)

### Special Pages

The repository includes several standalone HTML pages with custom functionality:

- **`cyclorank.html`** - Interactive data table using jQuery DataTables for ranking/visualization
- **`tallinn-maps.html`** - Custom maps page
- **`content.html`** - Content listing page
- **`index.html`** - Homepage with author bio (uses `archive` layout)

These pages are complete HTML documents, not processed through Jekyll's typical page rendering (though they may use layouts).

## Theme Architecture

This site uses Jekyll's `remote_theme` feature to pull the Minimal Mistakes theme from GitHub. Local customizations are layered on top:

1. **Theme files** are fetched from `mmistakes/minimal-mistakes` repository
2. **Local overrides** in `_layouts/`, `_includes/`, and `_sass/` take precedence
3. **Custom skin** is defined in `_config.yml` as `minimal-mistakes_skin: "custom"`

When modifying theme files, copy the original from the Minimal Mistakes repo and place it in the corresponding local directory before editing.

## Content Guidelines

### Blog Post Front Matter
Posts should include standard YAML front matter:
```yaml
---
layout: single
classes: narrow  # optional, for narrower content width
title: "Post Title"
date: YYYY-MM-DD
categories: technical  # or other categories
comments: false
---
```

### Math Support
Posts support LaTeX math rendering via MathJax/KaTeX. Use standard LaTeX syntax with `$$` delimiters for display math.

## Deployment

The site automatically deploys to GitHub Pages when changes are pushed to the master branch. No manual deployment steps are required.

## Important Notes

- **Jekyll version**: Uses GitHub Pages compatible versions (defined in Gemfile.lock)
- **Ruby gems**: The site requires both `minimal-mistakes-jekyll` gem and various Jekyll plugins
- **Timezone**: Set to `Europe/Tallinn` in `_config.yml`
- **Permalinks**: Uses `/:categories/:title/` structure
- **HTML compression**: Enabled for production via `compress_html` in `_config.yml`
