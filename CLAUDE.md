# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog and portfolio site for Arunkumar Sampathkumar, built with Jekyll and hosted on Firebase.

## Build & Development Commands

```bash
# Install dependencies
bundle install

# Local development server (http://localhost:4000)
bundle exec jekyll serve

# Production build
JEKYLL_ENV=production bundle exec jekyll build

# Clean build artifacts
bundle exec jekyll clean

# Full production deploy (requires FIREBASE_TOKEN)
bundle exec jekyll clean && JEKYLL_ENV=production bundle exec jekyll build && firebase deploy

# Preview production build locally
firebase serve
```

## Technology Stack

- **Jekyll 3.9.1** with Kramdown (GFM) and Rouge syntax highlighting
- **Material Design Lite (MDL)** for UI components and layout
- **SCSS** for styling (`assets/css/main.scss`)
- **Firebase Hosting** for deployment
- **GitHub Actions** CI on push to `master` (`.github/workflows/publish.yml`)

## Architecture

### Content Collections

Jekyll collections defined in `_config.yml`:

| Collection | Directory | Purpose |
|------------|-----------|---------|
| posts | `_posts/` | Blog articles (Markdown with front matter) |
| apps | `_apps/` | Android app showcases |
| os | `_os/` | Open source project showcases |
| talks | `_talks/` | Speaking engagements |

### Layout Hierarchy

`default.html` is the base layout (MDL drawer + header + content grid). Other layouts extend it:

- `post.html` - Blog posts with reading time, social sharing, Disqus comments
- `blog.html` - Blog listing page
- `portfolio.html` - Homepage with apps, open source, and talks sections

### Key Includes

Custom embeds for use in Markdown content:

- `youtube_embed.html` - Responsive YouTube videos (pass `id` parameter)
- `gfycat_embed.html` - Gfycat video embeds
- `images_caption.html`, `images_center.html`, `images_full_width.html` - Image layouts
- `images_phone_screenshot.html` - Phone-framed screenshots
- `reading_time.html` - Calculates and displays read time

### Front Matter Defaults

Posts automatically get `is_post: true`, `comments: true`, and `navigation_name: Blog`. Pages get `is_post: false` and a default header image.

### Permalink Structure

All content uses `/:title/` (defined in `_config.yml`), producing clean URLs.

## Deployment

CI deploys automatically on push to `master` via GitHub Actions. The workflow installs Ruby, builds the Jekyll site with `JEKYLL_ENV=production`, and deploys the `_site/` directory to Firebase Hosting using `FIREBASE_TOKEN` secret.
