# Local setup (for future me)

Personal notes for working on this blog across multiple machines.
Full writeup: https://0gam-B.github.io/0gam-log/posts/setting-up-hugo-blog-github-pages/

## New machine bootstrap
1. `brew install hugo gh`
2. `gh auth login`
3. `git clone --recurse-submodules https://github.com/0gam-B/0gam-log.git`
4. `cd 0gam-log && hugo server`

## Every session
- Start: `git pull`
- End: `git add . && git commit -m "..." && git push`

## Hugo version
- Local/Actions both on 0.160.1+
- Bump via `.github/workflows/hugo.yml` and `brew upgrade hugo`