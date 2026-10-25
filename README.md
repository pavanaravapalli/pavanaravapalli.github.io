# pavanaravapalli.github.io

Personal site and blog for Pavan Aravapalli, built with Jekyll for GitHub Pages. Live address: https://pavanaravapalli.github.io

## One-time setup

1. On GitHub, create a **public** repository named exactly `pavanaravapalli.github.io`.
2. Copy all files from this folder into the repository root and push:

   ```
   git init
   git add .
   git commit -m "Initial site"
   git branch -M main
   git remote add origin https://github.com/pavanaravapalli/pavanaravapalli.github.io.git
   git push -u origin main
   ```
3. In the repository go to **Settings > Pages**, set **Source** to "Deploy from a branch", branch `main`, folder `/ (root)`.
4. Wait a minute or two. The site appears at https://pavanaravapalli.github.io

## To do

- The "Contact" menu link and footer link go to your LinkedIn profile. Change `linkedin_username` in `_config.yml` if it ever changes.
- Topics for the home page filter buttons are listed under `topics:` in `_config.yml`. A post joins a topic when its `categories:` line contains the topic's `key`, for example `categories: [cloudstack, troubleshooting]`.

## Publishing a new post

Create a file in `_posts/` named `YYYY-MM-DD-short-title.md` with this content:

````
---
layout: post
title: "Your post title"
date: 2026-11-01 10:00:00 +0530
categories: [cloudstack]
---

Your Markdown content here. Fenced code blocks work.
````

Then run `git add . && git commit -m "New post" && git push`. The site updates in about a minute. The home page lists all posts, newest first.

Drafts live in `_drafts/` and are never published. To preview them locally, run `bundle exec jekyll serve --drafts`.

## Structure

- `index.html`: home page with intro, UEFI section and the list of posts
- `about.html`: about page
- `_layouts/`: page templates
- `assets/favicon.svg`, `favicon-64.png`, `apple-touch-icon.png`: the `>_` tab icon (the header icon is drawn inline in `_layouts/default.html`)
- `_config.yml` `title:`: the browser tab title of the home page
- `assets/css/style.css`: all styling (dark by default, follows the visitor's light/dark setting, with a toggle in the header)

## Preview locally (optional)

Use Ruby 3.3 (newer versions don't work with the GitHub Pages gems):

```
bundle install
bundle exec jekyll serve --livereload
```

Open http://localhost:4000

## Adding a custom domain later

Add your domain under **Settings > Pages > Custom domain** and set the DNS records GitHub shows. Old `github.io` links redirect automatically.
