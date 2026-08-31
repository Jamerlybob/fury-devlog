# Reviving Fury — dev log

A Hugo site (PaperMod theme) documenting an ongoing project to reverse-engineer
and revive *Fury*, a 2007 MMO. Deploys automatically to GitHub Pages via GitHub
Actions on every push to `main`.

## First-time setup

1. **Create a new GitHub repo** (e.g. `fury-devlog`), then turn this folder
   into a git repo and push it:
   ```bash
   cd fury-devlog
   git init
   git add -A
   git commit -m "Initial site setup"
   git branch -M main
   git remote add origin https://github.com/jamerlybob/fury-devlog.git
   git push -u origin main
   ```
   The PaperMod theme is included directly as plain files under
   `themes/PaperMod` (not a submodule), so there's no extra setup step —
   it'll just work once pushed. If you ever want to update the theme later,
   you can re-download it from
   [github.com/adityatelange/hugo-PaperMod](https://github.com/adityatelange/hugo-PaperMod)
   and replace that folder.

2. **Enable GitHub Pages via Actions**: in your repo, go to
   *Settings → Pages → Build and deployment → Source*, and select
   **GitHub Actions**. You don't need to pick a branch — the workflow handles it.

3. Your username (`jamerlybob`) is already filled in throughout `hugo.yaml` —
   nothing to change there.

4. Push again (or just push your username fix) — the Actions tab will show
   the build running, and your site will be live at
   `https://jamerlybob.github.io/fury-devlog/` a minute or two later.

## Writing a new post

```bash
hugo new content posts/my-post-slug.md
```

This uses the template in `archetypes/posts.md` (the "where I left off / what
I tried / what broke / what I learned / next up" structure). New posts are
created with `draft: true` — set it to `false` when you're ready to publish.

## Local preview

```bash
hugo server --buildDrafts
```

Then open `http://localhost:1313/`.

## Structure

```
content/
  posts/       — the dev log itself
  glossary/    — running reference for recurring terms
  about.md     — about page
archetypes/
  posts.md     — template for new posts
hugo.yaml      — site config, theme params, nav menu
.github/workflows/hugo.yml — auto-deploy to GitHub Pages on push
```
