# Running this blog

The full walkthrough moved into the project's single human-facing doc so there's
one place to keep current.

**Read it there:** `../docs/FOR-JAMES.md` in the parent `Fury_Project` repo:

- section 4, "Running the blog, day to day" (WSL setup, new post, preview,
  commit / push, publish, unpublish, delete),
- section 5, "Writing: Markdown, images, diagrams",
- section 8, "The look",
- section 11, "Cheat-sheet and common errors".

Thirty-second version:

```bash
cd /mnt/c/Users/james/OneDrive/Desktop/Coding/Projects/Fury_Project/devlog
# ...edit content/posts/<slug>/index.md...
hugo server --buildDrafts     # preview at http://localhost:1313/
git add -A && git commit -m "..." && git push   # push = deploy
```

All blog commands run in **WSL** (the Ubuntu app), not Git Bash or PowerShell.
To publish a draft, set `draft: false` in its front matter, then commit and push.
Keep post dates in the past or `buildFuture: false` hides them.
