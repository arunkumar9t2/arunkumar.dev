# CLAUDE.md - AI Assistant Guide for Arunkumar.dev

This document provides comprehensive guidance for AI assistants working with the Arunkumar.dev codebase. Last updated: 2025-11-17

## Project Overview

**Arunkumar.dev** is a Jekyll-based static website serving as a personal portfolio and technical blog for Arunkumar (@arunkumar_9t2). The site showcases Android development projects, open-source contributions, technical blog posts, and speaking engagements.

- **Production URL:** https://www.arunkumar.dev
- **Hosting:** Firebase Hosting (project: `my-website-fd94d`)
- **CI/CD:** GitHub Actions (auto-deploy on push to `master`)
- **Repository:** Personal portfolio and blog

## Technology Stack

### Core Technologies
- **Jekyll** v3.9.1 - Static site generator
- **Ruby** 2.x - Runtime environment
- **Material Design Lite (MDL)** - UI framework
- **SCSS** - CSS preprocessor
- **kramdown** - Markdown parser with GFM support
- **Rouge** - Syntax highlighting
- **Firebase** - Hosting and deployment

### Key Dependencies
```ruby
# Gemfile
gem "jekyll", "~> 3.9.1"
gem "jekyll-feed"          # RSS feed generation
gem "jekyll-gist"          # GitHub gist embedding
gem "jemoji"               # Emoji support
gem "jekyll-sitemap"       # SEO sitemap
gem "jekyll-seo-tag"       # SEO meta tags
gem "jekyll-compose"       # Helper commands for posts
gem "nokogiri", ">= 1.12.5" # Security requirement
```

### Build Tools
- **bundler** - Ruby dependency management
- **Node.js** 16.x - For Firebase CLI
- **Firebase CLI** - Deployment tool

## Directory Structure

```
arunkumar.dev/
├── _config.yml              # Jekyll configuration (CRITICAL)
├── Gemfile                  # Ruby dependencies
├── firebase.json            # Firebase hosting config
├── .firebaserc             # Firebase project settings
├── README.md               # Basic project documentation
│
├── _layouts/               # Page templates (4 layouts)
│   ├── default.html        # Master layout with MDL structure
│   ├── post.html          # Blog post template
│   ├── blog.html          # Blog listing page
│   └── portfolio.html     # Portfolio homepage
│
├── _includes/              # Reusable components (13 includes)
│   ├── head.html          # Meta tags, styles, analytics
│   ├── footer.html        # Footer content
│   ├── disqus_comments.html  # Comment system
│   ├── google-analytics.html # Analytics (production only)
│   ├── reading_time.html  # Post reading time estimate
│   ├── youtube_embed.html # Responsive YouTube iframes
│   ├── gfycat_embed.html  # GIF embeds
│   └── images_*.html      # Image styling utilities
│
├── _posts/                 # Blog articles (9 posts)
│   └── YYYY-MM-DD-title.md
│
├── _apps/                  # Android apps collection (8 apps)
│   └── app-name.md
│
├── _os/                    # Open source projects (6 projects)
│   └── project-name.md
│
├── _talks/                 # Speaking engagements (4 talks)
│   └── talk-name.md
│
├── assets/
│   ├── css/
│   │   ├── main.scss      # Custom styles (~300+ lines)
│   │   └── syntax_highlight.css  # Code highlighting theme
│   └── images/            # All images (~40+ files)
│       ├── arun.png       # Profile photo
│       ├── about-header.jpg
│       └── [app/post images]
│
├── apps/                   # App-specific content pages
│   ├── lynket/
│   │   └── privacy.md     # Privacy policy
│   └── kolorette/
│       └── privacy.md
│
├── .github/workflows/
│   └── publish.yml        # CI/CD pipeline
│
├── .vscode/
│   └── tasks.json         # VS Code build tasks
│
├── index.md               # Homepage (portfolio)
├── blog.md                # Blog listing page
├── about.md               # About/bio page
└── 404.html               # Custom 404 page
```

## Jekyll Configuration (_config.yml)

### Critical Settings
```yaml
# Site Identity
title: Arunkumar
email: arunkumar@outlook.com
description: Android Engineer. Building @Square
baseurl: ""
url: "https://www.arunkumar.dev"

# Markdown & Syntax
markdown: kramdown
highlighter: rouge
kramdown:
  input: GFM
  syntax_highlighter: rouge

# Permalink Structure
permalink: /:title/  # Clean URLs without dates

# Collections (IMPORTANT)
collections:
  apps:
    output: false  # Not individual pages
  os:
    output: false
  talks:
    output: false

# Plugins
plugins:
  - jekyll-feed
  - jekyll-gist
  - jemoji
  - jekyll-sitemap
  - jekyll-seo-tag

# Social Integration
twitter_username: arunkumar_9t2
github_username: arunkumar-sampathkumar
disqus:
  shortname: arunkumar-dev

# Build Settings
exclude:
  - Gemfile
  - Gemfile.lock
  - node_modules
  - vendor
```

## Layout System

### 1. default.html (Master Layout)
**Purpose:** Base template for all pages with MDL structure

**Key Features:**
- Fixed header with site title
- Left drawer navigation with:
  - Profile avatar and name
  - Social links (Twitter, GitHub, LinkedIn, Play Store, Email, RSS)
  - Main navigation (Portfolio, Blog, About)
- Responsive MDL grid system
- Google Analytics integration (production only)

**Usage:** All other layouts extend this via `layout: default`

### 2. portfolio.html
**Purpose:** Homepage showcasing work

**Sections:**
- Android Projects (loops through `_apps` collection)
- Open Source (loops through `_os` collection)
- Public Speaking (loops through `_talks` collection)
- Academic Publications (hardcoded)

**Card Structure:** MDL cards with image, title, description, action buttons

### 3. blog.html
**Purpose:** Blog listing page

**Features:**
- Grid of post cards (4-col desktop, 4-col tablet, 12-col phone)
- Each card shows: hero image, date, reading time, title, description
- "Read More" CTA button linking to full post

### 4. post.html
**Purpose:** Individual blog post pages

**Features:**
- Full-width hero image
- Title, date, and reading time
- Two-column content layout (8-col content, 2-col padding)
- Disqus comments section
- Schema.org BlogPosting markup for SEO

## Content Management

### Blog Posts (_posts/)

**File Naming Convention:**
```
YYYY-MM-DD-post-title-slug.md
```

**Front Matter Template:**
```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD HH:MM +0530
description: "Brief description for SEO and cards"
categories: [Category1, Category2]
image: /assets/images/post-hero.png
---
```

**Writing Guidelines:**
1. Always include a hero image (recommended: 1200x630px for social sharing)
2. Use descriptive titles (good for SEO)
3. Keep descriptions under 160 characters
4. Use kramdown/GFM markdown syntax
5. Code blocks: Use triple backticks with language identifier
6. Images: Place in `/assets/images/`, reference with absolute path

**Available Includes for Posts:**
```liquid
{% include youtube_embed.html id="VIDEO_ID" %}
{% include gfycat_embed.html id="GIF_ID" width="600" height="400" %}
{% include images_center.html url="/path/to/image.png" %}
{% include images_full_width.html url="/path/to/image.png" %}
{% include images_phone_screenshot.html url="/path/to/screenshot.png" %}
{% include images_caption.html url="/path/to/image.png" description="Caption text" %}
```

### Android Apps (_apps/)

**Front Matter Template:**
```yaml
---
name: "App Name"
hero_image: "https://example.com/hero.png"
description: |
  Markdown description of the app.
  Can be multi-line.
playstore: "https://play.google.com/store/apps/details?id=..."  # Optional
github: "https://github.com/user/repo"  # Optional
youtube: "https://youtube.com/watch?v=..."  # Optional
ordering: 1  # Display order (1-8)
---
```

**Ordering:** Lower numbers appear first in portfolio

### Open Source Projects (_os/)

**Front Matter Template:**
```yaml
---
name: "Project Name"
description: "Brief description of the project"
github: "https://github.com/user/repo"
ordering: 1  # Display order (1-6)
---
```

### Public Talks (_talks/)

**Front Matter Template:**
```yaml
---
name: "Talk Title"
link: "https://youtube.com/watch?v=... or presentation URL"
image: "/assets/images/event.png"
published: "Date, Event Name"
ordering: 1  # Display order (1-4)
---
```

## Development Workflows

### Local Development

**Initial Setup:**
```bash
# Install Ruby (system-specific)
# Install bundler
gem install bundler jekyll

# Install dependencies
bundle install

# Serve locally (auto-reload on changes)
bundle exec jekyll serve
```

**Access:** http://localhost:4000

**VS Code Tasks (configured in .vscode/tasks.json):**
- `Jekyll Serve` (Default: Ctrl/Cmd+Shift+B) - Start development server
- `Firebase Serve` - Test Firebase hosting locally
- `Firebase Deploy` - Build and deploy to production

### Making Changes

**For Blog Posts:**
1. Create new post: `_posts/YYYY-MM-DD-title.md`
2. Add front matter with required fields
3. Write content in Markdown
4. Add hero image to `/assets/images/`
5. Test locally with `bundle exec jekyll serve`
6. Commit and push to `master` for auto-deploy

**For Portfolio Items:**
1. Add new file to appropriate collection (`_apps/`, `_os/`, `_talks/`)
2. Follow front matter template
3. Set `ordering` value
4. Add associated images to `/assets/images/`
5. Test locally
6. Commit and push

**For Styles:**
1. Edit `/assets/css/main.scss`
2. Use SCSS nesting and variables
3. Follow existing MDL color scheme (blue-grey primary)
4. Test responsiveness (desktop, tablet, phone breakpoints)
5. Avoid modifying MDL core styles

**For Layouts/Includes:**
1. Edit files in `_layouts/` or `_includes/`
2. Test across all pages using that layout
3. Maintain MDL structure and classes
4. Preserve responsive grid system
5. Test analytics/comments functionality

### Build Process

**Development Build:**
```bash
bundle exec jekyll build
# Output: _site/
```

**Production Build:**
```bash
JEKYLL_ENV=production bundle exec jekyll build
# Enables: Google Analytics, minification, production assets
```

**Clean Build:**
```bash
bundle exec jekyll clean  # Remove _site/ and caches
bundle exec jekyll build
```

## Deployment

### Automatic Deployment (Recommended)

**Trigger:** Push to `master` branch

**GitHub Actions Workflow (.github/workflows/publish.yml):**
1. Checkout code
2. Setup Ruby 2.x
3. Setup Node 16.x
4. Install build-essential (Ruby native extensions)
5. Install bundler
6. Install Firebase CLI via npm
7. Clean Jekyll build
8. Production build: `JEKYLL_ENV=production bundle exec jekyll build`
9. Deploy: `firebase deploy --token $FIREBASE_TOKEN`

**Required Secret:** `FIREBASE_TOKEN` (stored in GitHub repository secrets)

### Manual Deployment

```bash
# Clean previous build
bundle exec jekyll clean

# Production build
JEKYLL_ENV=production bundle exec jekyll build

# Deploy to Firebase
firebase deploy
```

**Prerequisites:**
- Firebase CLI installed: `npm install -g firebase-tools`
- Authenticated: `firebase login`

### Firebase Configuration

**firebase.json:**
- Public directory: `_site/` (Jekyll output)
- Ignores: Firebase hosting ignores Gemfile, node_modules, CNAME, etc.

**.firebaserc:**
- Default project: `my-website-fd94d`

## Key Conventions

### Coding Style

**HTML/Liquid:**
- Use 2-space indentation
- Follow MDL component patterns
- Keep layouts DRY (Don't Repeat Yourself)
- Use includes for reusable components
- Semantic HTML5 elements

**SCSS:**
- Nest selectors logically
- Use MDL color utilities: `mdl-color--primary`, `mdl-color-text--accent`
- Follow BEM-like naming for custom classes
- Mobile-first responsive design
- Use MDL grid: `mdl-grid`, `mdl-cell`, `mdl-cell--N-col`

**Markdown:**
- Use GitHub Flavored Markdown (GFM)
- Code blocks with language identifiers
- Use includes for special formatting
- Absolute paths for images: `/assets/images/file.png`

### Naming Conventions

**Files:**
- Posts: `YYYY-MM-DD-kebab-case-title.md`
- Collections: `kebab-case-name.md`
- Images: `lowercase-with-hyphens.png` or `descriptive-name.jpg`
- Layouts: `lowercase.html`
- Includes: `snake_case.html` or `kebab-case.html`

**Front Matter:**
- Use double quotes for strings
- ISO 8601 dates: `YYYY-MM-DD HH:MM +ZZZZ`
- Boolean values: `true`/`false` (lowercase)

### Git Workflow

**Branch Strategy:**
- `master` - Production branch (auto-deploys)
- Feature branches: `feature/description` or `claude/session-id`
- Always work on feature branches for AI assistant sessions

**Commit Messages:**
- Use conventional commits style
- Examples:
  - `feat: Add new blog post about Jetpack Compose`
  - `fix: Correct typo in about page`
  - `style: Update hero image for Compass post`
  - `docs: Update README with new dependencies`
  - `chore: Bump nokogiri version for security`

**Before Pushing:**
1. Test locally: `bundle exec jekyll serve`
2. Check build: `JEKYLL_ENV=production bundle exec jekyll build`
3. Verify no broken links or images
4. Review changes: `git diff`
5. Commit with descriptive message
6. Push to feature branch first, then merge to `master`

## Common Tasks

### Adding a New Blog Post

```bash
# 1. Create post file
# File: _posts/2025-11-17-my-new-post.md

---
layout: post
title: "My Awesome New Post"
date: 2025-11-17 10:00 +0530
description: "A brief description of my post"
categories: [Android, Kotlin]
image: /assets/images/my-post-hero.png
---

Your content here...

# 2. Add hero image
# Save to: assets/images/my-post-hero.png

# 3. Test locally
bundle exec jekyll serve

# 4. Commit and push
git add _posts/2025-11-17-my-new-post.md assets/images/my-post-hero.png
git commit -m "feat: Add blog post about [topic]"
git push origin master
```

### Adding a New Android App

```bash
# 1. Create app file
# File: _apps/my-new-app.md

---
name: "My New App"
hero_image: "https://example.com/hero.png"
description: |
  Brief description of the app's purpose and features.
playstore: "https://play.google.com/store/apps/details?id=com.example.app"
github: "https://github.com/user/my-new-app"
ordering: 9
---

# 2. Test locally
bundle exec jekyll serve
# Visit: http://localhost:4000 (portfolio page)

# 3. Commit and push
git add _apps/my-new-app.md
git commit -m "feat: Add My New App to portfolio"
git push origin master
```

### Updating Styles

```bash
# 1. Edit SCSS
# File: assets/css/main.scss

# 2. Test with live reload
bundle exec jekyll serve

# 3. Test production build
JEKYLL_ENV=production bundle exec jekyll build

# 4. Commit
git add assets/css/main.scss
git commit -m "style: Update card styling for better mobile view"
git push origin master
```

### Updating Dependencies

```bash
# 1. Update Gemfile
# Edit: Gemfile

# 2. Update lock file
bundle update

# 3. Test build
bundle exec jekyll build

# 4. Commit both files
git add Gemfile Gemfile.lock
git commit -m "chore: Update jekyll dependencies"
git push origin master
```

### Creating Custom Includes

```bash
# 1. Create include file
# File: _includes/my_custom_component.html

<div class="custom-component">
  {{ include.content }}
</div>

# 2. Use in posts/pages
{% include my_custom_component.html content="Hello World" %}

# 3. Test
bundle exec jekyll serve
```

## Important Files Reference

### Must-Read Files
1. **_config.yml** - All Jekyll configuration
2. **_layouts/default.html** - Master page structure
3. **_includes/head.html** - Meta tags, styles, scripts
4. **assets/css/main.scss** - Custom styling
5. **firebase.json** - Deployment configuration

### Don't Modify (Unless Necessary)
- **Gemfile.lock** - Auto-generated dependency lock
- **.firebaserc** - Firebase project configuration
- **_site/** - Build output (git-ignored, auto-generated)
- **.sass-cache/** - Build cache (git-ignored)

### Safe to Modify
- **_posts/** - All blog content
- **_apps/**, **_os/**, **_talks/** - Portfolio collections
- **assets/images/** - All images
- **assets/css/main.scss** - Custom styles
- **_layouts/**, **_includes/** - Templates (with testing)
- **index.md**, **blog.md**, **about.md** - Main pages

## Testing Checklist

Before deploying changes, verify:

### Content
- [ ] No typos or grammatical errors
- [ ] All links work (internal and external)
- [ ] Images load correctly
- [ ] Code snippets have proper syntax highlighting
- [ ] Front matter is valid YAML

### Visual
- [ ] Responsive on desktop (4-col grid)
- [ ] Responsive on tablet (4-col grid)
- [ ] Responsive on phone (12-col grid)
- [ ] Images scale properly
- [ ] Navigation works on mobile (drawer)
- [ ] Cards render correctly

### Functionality
- [ ] Local build succeeds: `bundle exec jekyll build`
- [ ] Production build succeeds: `JEKYLL_ENV=production bundle exec jekyll build`
- [ ] No Jekyll warnings or errors
- [ ] Analytics loads (production only)
- [ ] Disqus comments load on posts
- [ ] RSS feed validates
- [ ] Sitemap generates correctly

### Performance
- [ ] Images optimized (compress before adding)
- [ ] No large file sizes (keep images < 500KB)
- [ ] Minimal custom JavaScript
- [ ] Leverage MDL/CDN resources

## Tips for AI Assistants

### DO:
1. **Always test locally** before committing
2. **Use existing patterns** - Browse similar files before creating new ones
3. **Follow conventions** - Match existing naming, structure, and style
4. **Preserve responsive design** - Test all breakpoints
5. **Optimize images** - Compress before adding to repository
6. **Write descriptive commits** - Follow conventional commit format
7. **Update this file** - Keep CLAUDE.md current when making structural changes
8. **Check build output** - Watch for Jekyll warnings and errors
9. **Use includes** - Reuse existing components for consistency
10. **Respect the brand** - Maintain professional, technical tone

### DON'T:
1. **Don't push directly to master** - Use feature branches (except for content updates)
2. **Don't modify MDL core** - Only extend with custom SCSS
3. **Don't break the build** - Always verify `jekyll build` succeeds
4. **Don't ignore warnings** - Address Jekyll/bundler warnings
5. **Don't add unnecessary dependencies** - Keep Gemfile minimal
6. **Don't hardcode URLs** - Use `{{ site.url }}` and `{{ site.baseurl }}`
7. **Don't skip front matter** - Every content file needs proper YAML
8. **Don't commit build artifacts** - Never commit `_site/` or `.sass-cache/`
9. **Don't use inline styles** - Always use SCSS classes
10. **Don't forget SEO** - Include meta descriptions, titles, and schema markup

### Quick Reference Commands

```bash
# Development
bundle exec jekyll serve          # Local dev server
bundle exec jekyll build          # Build site
bundle exec jekyll clean          # Clean build artifacts

# Production
JEKYLL_ENV=production bundle exec jekyll build  # Production build

# Dependencies
bundle install                    # Install Ruby dependencies
bundle update                     # Update dependencies

# Deployment
firebase serve                    # Test Firebase hosting locally
firebase deploy                   # Deploy to production

# Git (AI Assistant Sessions)
git checkout -b claude/session-id # Create feature branch
git add .                         # Stage changes
git commit -m "type: description" # Commit with message
git push -u origin claude/session-id  # Push to remote
```

### Understanding Jekyll Liquid

**Common Liquid Syntax:**
```liquid
{{ variable }}                    # Output variable
{{ site.title }}                  # Site config variable
{{ page.title }}                  # Page front matter variable
{{ post.date | date: "%B %d, %Y" }}  # Filter example

{% if condition %}                # Conditional
{% endif %}

{% for post in site.posts %}      # Loop
{% endfor %}

{% include file.html param="value" %}  # Include with params

{{ content }}                     # Page/post content placeholder
```

**Collections Access:**
```liquid
{% for app in site.apps | sort: 'ordering' %}
  {{ app.name }}
  {{ app.description }}
{% endfor %}
```

### Material Design Lite Grid

**Grid System:**
```html
<!-- Container -->
<div class="mdl-grid">
  <!-- Cell: 4-col desktop, 4-col tablet, 12-col phone -->
  <div class="mdl-cell mdl-cell--4-col mdl-cell--4-col-tablet mdl-cell--12-col-phone">
    Content
  </div>
</div>
```

**Common MDL Classes:**
- `mdl-card` - Card container
- `mdl-card__media` - Card image area
- `mdl-card__title` - Card title area
- `mdl-card__supporting-text` - Card body text
- `mdl-card__actions` - Card action buttons
- `mdl-button` - Material button
- `mdl-button--raised` - Raised button style
- `mdl-button--colored` - Colored button
- `mdl-color--primary` - Primary color background
- `mdl-color-text--accent` - Accent color text

### Responsive Breakpoints

```scss
// Desktop: > 840px (default)
.mdl-cell--4-col { width: 33.33%; }

// Tablet: 480px - 839px
.mdl-cell--4-col-tablet { width: 50%; }

// Phone: < 479px
.mdl-cell--12-col-phone { width: 100%; }
```

## Troubleshooting

### Common Issues

**Build Fails:**
- Check `bundle exec jekyll build` output for errors
- Verify front matter YAML is valid (no tabs, proper spacing)
- Ensure all referenced images exist
- Check for liquid syntax errors

**Images Not Loading:**
- Use absolute paths: `/assets/images/file.png`
- Verify file exists in `assets/images/`
- Check file name matches exactly (case-sensitive)
- Ensure image is committed to git

**Styles Not Applying:**
- Clear browser cache
- Check SCSS syntax in `assets/css/main.scss`
- Rebuild: `jekyll clean && jekyll build`
- Verify class names match MDL documentation

**Deployment Fails:**
- Check GitHub Actions logs
- Verify `FIREBASE_TOKEN` secret is set
- Ensure `firebase.json` is valid
- Test local deployment: `firebase deploy`

**Collections Not Showing:**
- Check `_config.yml` has collection defined
- Verify front matter is present
- Ensure `ordering` field is set
- Restart Jekyll server after config changes

### Getting Help

**Resources:**
- Jekyll Documentation: https://jekyllrb.com/docs/
- Material Design Lite: https://getmdl.io/components/
- kramdown Syntax: https://kramdown.gettalong.org/syntax.html
- Firebase Hosting: https://firebase.google.com/docs/hosting
- Liquid Template Language: https://shopify.github.io/liquid/

**Debugging:**
```bash
# Verbose build output
bundle exec jekyll build --verbose

# Show configuration
bundle exec jekyll build --config _config.yml --verbose

# Serve with drafts
bundle exec jekyll serve --drafts

# Check Ruby/Jekyll versions
ruby --version
bundle exec jekyll --version
```

## Project Context

**Owner:** Arunkumar (@arunkumar_9t2)
**Role:** Android Engineer at Square
**Focus:** Android development, Kotlin, Jetpack libraries, open-source
**Content Themes:** Technical deep-dives, Android architecture, performance optimization, Jetpack Compose

**Content Tone:**
- Professional and technical
- In-depth technical explanations
- Code-heavy with practical examples
- Educational and insightful

**Target Audience:**
- Android developers
- Software engineers
- Technical recruiters
- Open-source community

---

**End of CLAUDE.md**

*This document should be updated whenever significant structural changes are made to the codebase. Always refer to this file before making changes to understand the project conventions and workflows.*
