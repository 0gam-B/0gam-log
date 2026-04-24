+++
date = '2026-04-24T09:00:00+09:00'
draft = false
title = 'Setting up a Hugo blog on GitHub Pages: what I learned the hard way'
tags = ['hugo', 'github-pages', 'github-actions', 'blogging']
categories = ['devops']
summary = 'A walkthrough of setting up a Hugo + PaperMod blog with automated GitHub Pages deployment — including the specific errors I hit and how I got past them.'
+++

I spent an evening setting up a Hugo blog with GitHub Pages.
The "happy path" in most tutorials took me about 30 minutes.
The errors no one warned me about took another 2 hours.

This post is what I wish I had read before I started.

## What I wanted

A personal tech blog where:

- I write in Markdown and `git push` to publish
- The site is hosted for free
- I own the content and can move platforms later
- It looks clean and is fast

Hugo + PaperMod + GitHub Pages + GitHub Actions fits all four.

## The happy path

Here's the sequence that, in theory, should just work:

```bash
# 1. Install Hugo (must be the "extended" build)
brew install hugo

# 2. Create a new site
hugo new site my-blog
cd my-blog
git init
git branch -M main

# 3. Add a theme as a submodule (not a clone!)
git submodule add --depth=1 \
  https://github.com/adityatelange/hugo-PaperMod.git \
  themes/PaperMod

# 4. Point hugo.toml at the theme and set baseURL
# 5. Write a first post
hugo new content posts/hello.md

# 6. Create a GitHub repo, push, enable Pages with Actions as source
# 7. Add a workflow that builds Hugo and deploys to Pages
```

That's the version in the tutorials. Here's what actually happened.

## Problem 1: `hugo v0.146.0 or greater is required`

My workflow pinned `HUGO_VERSION: 0.125.0`, which was the version in the tutorial I copied from.

PaperMod had since updated to require Hugo 0.146.0+. The GitHub Actions runner dutifully installed the old Hugo, then PaperMod refused to build.

The error cascade was long, but the *real* error was the first line:

```
ERROR => hugo v0.146.0 or greater is required for hugo-PaperMod to build
```

Everything after that — template render failures, `google_analytics.html not found`, taxonomy errors — was just fallout from the old Hugo misreading new templates.

**Fix:** Match the workflow's `HUGO_VERSION` to a recent Hugo release. I used `0.160.1` to match my local install.

```yaml
# .github/workflows/hugo.yml
env:
  HUGO_VERSION: 0.160.1  # was 0.125.0
```

**Lesson:** Pin versions, but revisit them. Themes move faster than your workflow file.

## Problem 2: `git push` asks for a password I don't have

I signed up for GitHub using Google OAuth, so I never set a GitHub password. When I ran my first `git push`, the terminal asked for a username and password — and I had nothing to type.

Turns out: **GitHub web login and Git CLI authentication are completely separate.** Your Google OAuth session only works in the browser. The CLI needs its own credentials.

Two ways out:

**Option A — GitHub CLI (what I did, recommended):**

```bash
brew install gh
gh auth login
# Choose: GitHub.com → HTTPS → Yes (authenticate Git) → Login with web browser
```

This gives you a one-time code in the terminal, opens the browser, you paste the code, approve, done. Git is now authenticated permanently.

**Option B — Personal Access Token:**

Generate one at *GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)*, scope `repo`, and paste it as the password when Git asks.

**Lesson:** OAuth logins don't give you CLI credentials. If you've never set a GitHub password, use `gh` or a token — not your Google password.

## Problem 3: I bumped the version, but the build still used the old one

After fixing Problem 1, I pushed and watched Actions run again. Same error. Same old Hugo version in the logs.

I stared at the workflow file for five minutes. The number was right. `0.160.1`. So why was the runner downloading `0.125.0`?

Because my commit hadn't actually made it to GitHub.

In my local editor, the file said `0.160.1`. On GitHub's server, it still said `0.125.0`. I had edited and committed, but the push had silently failed earlier in the session — and I never re-ran it.

The way I caught it:

```bash
# Check local file
grep HUGO_VERSION .github/workflows/hugo.yml
# 0.160.1 ✓

# Check what's on GitHub — open in browser:
# https://github.com/<user>/<repo>/blob/main/.github/workflows/hugo.yml
# Still showed 0.125.0 ✗
```

That mismatch told me the push was the issue, not the edit.

**Fix:**

```bash
git status          # saw the commit was local-only
git push            # actually sent it this time
```

**Lesson:** When a fix "doesn't work," check whether the fix actually shipped. The GitHub web view of the file is the source of truth for what the runner sees — not your local editor.

## Final working setup

`.github/workflows/hugo.yml`:

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.160.1
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Build with Hugo
        env:
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

`hugo.toml`:

```toml
baseURL = 'https://<your-github-id>.github.io/<repo-name>/'
languageCode = 'en-us'
title = '<your blog title>'
theme = 'PaperMod'

[params]
  author = '<you>'
  description = "..."
  ShowReadingTime = true
  ShowCodeCopyButtons = true

[markup.highlight]
  style = 'github-dark'
  lineNos = true
```

Local Hugo version (for parity with the runner):

```
$ hugo version
hugo v0.160.1+extended+withdeploy darwin/arm64
```

## Lessons learned

1. **Read the first error, not the last.** Template errors are often downstream of something more fundamental (like a version mismatch).
2. **Submodules, not clones.** A cloned theme folder is invisible to GitHub Actions by default.
3. **Local Hugo and Actions Hugo must match.** "Works on my machine" is a symptom of this mismatch.
4. **GitHub web login ≠ Git CLI auth.** If you OAuth in, install `gh` or mint a token.
5. **Check whether your fix shipped.** Open the file on github.com, not just in your editor.
6. **GitHub Pages needs `Source: GitHub Actions` set explicitly.** The default "Deploy from a branch" silently ignores your workflow.

---

If any of this helps you unstick a deploy, that's why this post exists.