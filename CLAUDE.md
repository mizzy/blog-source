# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a personal blog source repository for mizzy.org, built using Nebel, a custom Go-based static site generator.
The generated static files are deployed to the `mizzy.github.com` repository (symlinked as `public/`).

## Architecture

### Directory Structure

- `posts/` - Markdown blog posts with YAML frontmatter
  - Format: `YYYY-MM-DD-title.markdown`
  - Use simple English filenames without Japanese characters
- `layouts/` - HTML templates using Go template syntax
- `static/` - Static assets (CSS, JS, images, favicon)
- `public/` - Symlink to `../mizzy.github.com` where generated files are output

### Technologies

- **Static Site Generator**: Nebel (custom Go implementation at `../nebel`)
- **Templates**: Go HTML templates
- **Styling**: CSS with TASS (CSS preprocessor) for `.tass` files
- **Content Format**: Markdown with YAML frontmatter

## Common Commands

### Generate the site

```bash
nebel generate
```

### Create a new post

```bash
nebel new "Post Title"
```

### Preview changes locally

After generating, view the files in the `public/` directory with a local web server.

### Run pre-commit checks

```bash
pre-commit run --all-files
```

## Development Workflow

1. **Creating posts**: Use `nebel new` to create posts with proper naming and frontmatter
2. **Post format**: Posts use Markdown with YAML frontmatter including `title` and `date` fields
3. **Building**: Run `nebel generate` to rebuild the site after changes
   - IMPORTANT: Run `nebel generate` after every file update
4. **Deployment**: Generated files are output to the symlinked `public/` directory (points to `../mizzy.github.com`)

## Template System

Templates in `layouts/` use Go template syntax with available variables:

- `.Title` - Post title
- `.Date` - Post date
- `.Path` - Post URL path
- `.ParsedContent` - Rendered post content
- `.NextPost` / `.PrevPost` - Navigation between posts
- `.Index` - Whether rendering the index page

## Important Notes

- The `public/` directory is a symlink to a separate repository for GitHub Pages deployment
- Images for posts are stored in `static/images/YYYY/MM/` following the date structure
- The site uses custom gravatar and social links configured in the templates
- Pre-commit hooks are configured to ensure code quality (trailing whitespace, file endings, markdown linting, etc.)

## 記事を書くときの注意点

記事を書くときは必ず @WRITING.md を参照してください。
また必ず最近の記事をランダムに数記事ピックアップし、それらの文体をできる限り模倣して作るようにしてください。
