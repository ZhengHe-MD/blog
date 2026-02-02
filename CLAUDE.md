# Blog Repository Documentation

This repository contains a Chinese-language technical blog built with [Hexo](https://hexo.io/) and the [NexT theme](https://github.com/next-theme/hexo-theme-next).

## Repository Overview

- **Blog URL**: https://zhenghe-md.github.io/blog
- **Generator**: Hexo v8.1.1
- **Theme**: NexT v8.27.0 (Mist scheme)
- **Language**: Chinese (zh-CN)
- **Deployment**: Automated via GitHub Actions to GitHub Pages

## Directory Structure

```
.
├── source/                    # Blog content and assets
│   ├── _posts/               # 46 published blog posts
│   ├── _drafts/              # 3 draft posts
│   ├── _data/                # Custom theme data (variables.styl)
│   ├── about/                # About page
│   ├── categories/           # Categories listing
│   ├── tags/                 # Tags listing
│   └── images/               # Global images (e.g., avatar.jpeg)
├── themes/                   # Theme directory
│   ├── next/                 # Active NexT theme (v8.9.0)
│   ├── next-old/             # Legacy NexT version
│   └── landscape/            # Default Hexo theme (unused)
├── scaffolds/                # Templates for new content
│   ├── post.md              # New post template
│   ├── draft.md             # Draft template
│   └── page.md              # Page template
├── .github/workflows/        # CI/CD automation
│   └── pages.yml            # GitHub Actions deployment
├── _config.yml              # Hexo site configuration
├── package.json             # Node.js dependencies and scripts
└── public/                  # Generated static site (ignored in git)
```

## Configuration Files

### Site Configuration (`_config.yml`)

Key settings:
- **Title**: ZhengHe
- **URL**: https://zhenghe-md.github.io/blog
- **Theme**: next
- **Post asset folder**: Enabled (each post can have its own asset folder)
- **Pagination**: 10 posts per page
- **Syntax highlighting**: highlight.js enabled
- **Feed**: atom.xml generated

### Theme Configuration (`themes/next/_config.yml`)

Active features:
- **Scheme**: Mist (minimal variant)
- **Menu**: Home, Categories, Archives, About
- **Sidebar**: Disabled
- **Social links**: GitHub and Email
- **Comments**: Disqus (shortname: zhenghe-hexo-blog)
- **Analytics**: Google Analytics (UA-172943223-1)
- **Math**: MathJax enabled (per-page basis)
- **GitHub banner**: Enabled

### Custom Styling (`source/_data/variables.styl`)

Custom CSS overrides:
- Content width: 680px across all breakpoints
- Table font size: 12px

## Content Organization

### Posts

Posts are stored in `source/_posts/` with:
- Markdown format with YAML front-matter
- Date, title, category, and tags metadata
- Associated asset folders (38 posts have dedicated asset directories)

Example front-matter:
```yaml
---
title: 纸牌游戏：24
date: 2022-10-30 21:21:09
category: 实践
---
```

### Pages

Special pages:
- **About**: `/source/about/index.md` - Personal information
- **Categories**: `/source/categories/index.md` - Category listing
- **Tags**: `/source/tags/index.md` - Tag listing

### Drafts

Unpublished drafts in `source/_drafts/`:
- distributed-tracing-history.md
- count-large-events.md
- etcd.md

## Key Dependencies

### Core Dependencies

- **hexo**: v8.1.1 - Static site generator
- **hexo-theme-next**: v8.27.0 - Theme (installed in themes/next directory)
- **hexo-renderer-pandoc**: v0.5.0 - Markdown renderer (requires Pandoc)

### Plugins

- **hexo-filter-mathjax**: v0.9.1 - LaTeX math equations
- **hexo-generator-feed**: v3.0.0 - RSS/Atom feed
- **hexo-excerpt**: v1.3.1 - Post excerpts
- **hexo-tag-cplayer**: v1.0.0 - Audio player tag
- **hexo-tag-embed**: v1.0.0 - YouTube and media embedding

### Generators

- hexo-generator-archive: v2.0.0
- hexo-generator-category: v2.0.0
- hexo-generator-index: v4.0.0
- hexo-generator-tag: v2.0.0

### Renderers

- hexo-renderer-ejs: v2.0.0
- hexo-renderer-stylus: v3.0.1
- hexo-renderer-pandoc: v0.5.0

## Build & Deployment

### Local Development

```bash
# Install dependencies
npm install

# Run local server
npm run server

# Clean generated files
npm run clean

# Build site
npm run build
```

### CI/CD Pipeline

GitHub Actions workflow (`.github/workflows/pages.yml`):

**Trigger**: Push to master branch

**Steps**:
1. Checkout repository with submodules
2. Setup Node.js 16.x
3. Cache npm dependencies
4. Install Pandoc 2.14.0.3 (required for pandoc renderer)
5. Install npm dependencies
6. Build site with `npm run build`
7. Deploy to GitHub Pages using peaceiris/actions-gh-pages@v3

**Output**: Static site deployed to `gh-pages` branch

## NexT Theme Features

### Custom Tags

The theme provides custom tags for rich content:
- `{% button %}` - Styled buttons
- `{% note %}` - Styled note blocks
- `{% tabs %}` - Tab panels
- `{% pdf %}` - Embed PDFs
- `{% mermaid %}` - Mermaid diagrams
- `{% video %}` - Video embedding
- `{% link_grid %}` - Link grids
- `{% grouppicture %}` - Image galleries

### Third-Party Integrations

- **Comments**: Disqus, Gitalk, Utterances (Disqus active)
- **Analytics**: Google Analytics
- **Math**: MathJax for LaTeX equations
- **Search**: Local search supported (not currently enabled)

### Internationalization

Theme supports 26 languages, currently set to Chinese (zh-CN).

## Content Creation

### Create New Post

```bash
hexo new post "Post Title"
```

Creates `source/_posts/post-title.md` with asset folder `source/_posts/post-title/`

### Create New Draft

```bash
hexo new draft "Draft Title"
```

Creates `source/_drafts/draft-title.md`

### Publish Draft

```bash
hexo publish draft "Draft Title"
```

Moves draft from `_drafts/` to `_posts/`

## Content Statistics

- **Published posts**: 46
- **Draft posts**: 3
- **Asset folders**: 38
- **Total post content**: ~36MB
- **Languages supported**: 26
- **Theme plugins**: 15+

## Development Notes

### Pandoc Requirement

The blog uses `hexo-renderer-pandoc` which requires Pandoc to be installed:
- **CI/CD**: Pandoc 2.14.0.3 installed automatically in GitHub Actions
- **Local development**: Install Pandoc separately: `brew install pandoc` (macOS)

### Asset Management

Post assets are stored alongside posts due to `post_asset_folder: true`. When creating a new post, a folder with the same name is created for images and other assets.

### MathJax

Mathematical equations are supported via MathJax. Enable per post with:
```yaml
---
mathjax: true
---
```

### Comments

Disqus comments are enabled globally. Comment section appears at the bottom of each post.

## Maintenance Tasks

### Update Dependencies

```bash
npm update
```

### Update Theme

```bash
npm update hexo-theme-next
```

### Clean Cache

```bash
npm run clean
```

Removes `public/` and `db.json` cache files.

## Troubleshooting

### Build Failures

1. **Pandoc not found**: Install Pandoc (`brew install pandoc`)
2. **Memory issues**: Increase Node.js heap size: `NODE_OPTIONS=--max_old_space_size=4096 npm run build`
3. **Theme errors**: Check `themes/next/_config.yml` for syntax errors

### Deployment Issues

Check GitHub Actions logs at `.github/workflows/pages.yml` runs.

## Resources

- [Hexo Documentation](https://hexo.io/docs/)
- [NexT Theme Documentation](https://theme-next.js.org/)
- [Hexo Renderer Pandoc](https://github.com/wzpan/hexo-renderer-pandoc)
- [MathJax Documentation](https://docs.mathjax.org/)
