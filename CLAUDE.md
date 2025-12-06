# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal blog built with Hugo, using the hugo-blog-awesome theme (added as a git submodule).

## Common Commands

```bash
# Start development server with live reload
hugo server -D

# Build the site for production (outputs to public/)
hugo

# Create a new blog post
hugo new posts/post-name.md
```

## Project Structure

- `content/posts/` - Blog posts in Markdown with TOML front matter
- `hugo.toml` - Main site configuration (theme, menus, social links)
- `archetypes/default.md` - Template for new content
- `themes/hugo-blog-awesome/` - Theme (git submodule, do not modify)
- `static/` - Static assets served as-is
- `assets/` - Assets processed by Hugo (e.g., avatar image)

## Content Format

Blog posts use TOML front matter (+++):
```markdown
+++
date = '2025-12-05T19:18:58+01:00'
draft = true
title = 'Post Title'
+++

Content here...
```

Set `draft = false` to publish a post.
