# b-erdem.github.io

Personal site and writing, built with Jekyll's `minima` theme. GitHub Pages
builds it server-side, so you don't need Ruby installed to publish.

## Add a future post

Drop a Markdown file in `_posts/` named `YYYY-MM-DD-some-title.md` with front matter:

```markdown
---
layout: post
title: "Your title"
date: 2026-07-01
---

Body...
```

Commit and push. Minima lists it on the home page automatically.

## Preview locally (optional)

Needs Ruby and Bundler. Not required to publish.

```bash
bundle install
bundle exec jekyll serve   # http://localhost:4000
```

## Custom domain later (optional)

Add a `CNAME` file containing the domain, point a DNS `CNAME`/`ALIAS` at
`b-erdem.github.io`, and update `url:` in `_config.yml`. You're not locked in.
