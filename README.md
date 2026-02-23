# With Enthusiasm

Personal blog by Jaroslav Bachorik. Mostly Java/JVM stuff — serviceability, observability, and performance.

Built with [Jekyll](https://jekyllrb.com/) 4.3 and the [minima](https://github.com/jekyll/minima) theme, hosted on GitHub Pages.

## Prerequisites

- Ruby (with Bundler)
- Jekyll gems: install once with `bundle install`

## Local development

```bash
# Install dependencies (first time or after Gemfile changes)
bundle install

# Serve with live-reload
bundle exec jekyll serve --livereload

# Or use the convenience script
./serve.sh
```

The site will be available at `http://localhost:4000`. Changes to posts and layouts are picked up automatically; changes to `_config.yml` require a server restart.

## Writing a new post

Create a file in `_posts/` named `YYYY-MM-DD-slug.md` with front matter:

```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD
categories: [cat1, cat2]
tags: [tag1, tag2]
---
```

Images go in `assets/images/YYYY-MM-DD-slug/`.

## Project structure

```
_posts/          Blog post sources (Markdown)
_layouts/        Custom layouts (post, tag, notes)
_includes/       Reusable HTML partials
assets/          Images, CSS, and other static files
_config.yml      Jekyll site configuration
```

## License

[MIT](LICENSE)
