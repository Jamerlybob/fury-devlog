# Getting Started — A Noob-Friendly Guide

This is the full walkthrough: getting this site online, and then actually
writing and publishing posts (text formatting, images, gifs) day to day.
Nothing here assumes you've used Git before.

---

## Part 1: One-time setup

### 1. Install Git (if you don't have it)

Check if you already have it:
```bash
git --version
```
If that errors, install it from [git-scm.com/downloads](https://git-scm.com/downloads)
(Windows: download and run the installer, defaults are fine).

Then tell Git who you are (one-time, only needed once ever on your machine):
```bash
git config --global user.name "James"
git config --global user.email "your-email@example.com"
```
Use the same email as your GitHub account if you can — not required, but tidier.

### 2. Create the GitHub repo

1. Go to [github.com/new](https://github.com/new)
2. Repository name: `fury-devlog`
3. Leave it **Public** (needs to be public for free GitHub Pages)
4. **Don't** tick "Add a README" — we already have one
5. Click **Create repository**

GitHub will show you a page with some commands — ignore those, use the ones below instead (they're tailored to this project).

### 3. Push this folder to that repo

Open a terminal, `cd` into the unzipped `fury-devlog` folder, then run these one at a time:

```bash
cd fury-devlog
git init
git add -A
git commit -m "Initial site setup"
git branch -M main
git remote add origin https://github.com/jamerlybob/fury-devlog.git
git push -u origin main
```

**What each line actually does**, since it helps to know:
- `git init` — turns this folder into a Git repo (starts tracking changes)
- `git add -A` — stages *all* files, meaning "include these in the next save point"
- `git commit -m "..."` — actually creates the save point (a "commit"), with a short message describing it
- `git branch -M main` — names your main branch `main` (GitHub's default)
- `git remote add origin ...` — tells Git "this is the GitHub repo this folder belongs to"
- `git push -u origin main` — uploads your commit to GitHub

It'll probably prompt you to log in — follow whatever prompt appears (browser login is easiest if offered).

### 4. Turn on GitHub Pages

1. On your repo page on GitHub, click **Settings** (top right of the repo, not your account settings)
2. In the left sidebar, click **Pages**
3. Under "Build and deployment" → **Source**, choose **GitHub Actions**
4. That's it — no branch to pick, nothing else to configure

### 5. Watch it deploy

Click the **Actions** tab on your repo. You should see a workflow running (yellow dot → green check when done, takes 1-2 minutes). Once it's green, your site is live at:

```
https://jamerlybob.github.io/fury-devlog/
```

---

## Part 2: Writing and publishing posts (the ongoing workflow)

### The day-to-day loop

Every time you want to write or update a post, the pattern is always:

1. Write/edit the post file
2. `git add -A`
3. `git commit -m "describe what changed"`
4. `git push`

That push is what triggers the auto-deploy — a minute or two later, the live site updates itself. You never touch GitHub Pages settings again after the one-time setup above.

### Creating a new post

From inside the `fury-devlog` folder:
```bash
hugo new content posts/decompiling-goskills.md
```
(Pick your own filename — lowercase, hyphens instead of spaces, no spaces or special characters.)

This creates the file pre-filled with the template structure (Where I left off / What I tried / What broke / What I learned / Next up) and `draft: true` at the top.

Open it in any text editor (VS Code is a good free one) and write.

**When you're ready to publish**, change `draft: true` to `draft: false` at the top of the file. Draft posts never get built into the live site, so you can write in progress without anyone seeing it.

### Previewing before you publish

Run this from the `fury-devlog` folder:
```bash
hugo server --buildDrafts
```
Then open `http://localhost:1313/` in your browser. It live-updates as you save the file — great for checking formatting before pushing. Press `Ctrl+C` in the terminal to stop it when you're done.

### Formatting your post — Markdown basics

Posts are written in **Markdown**, a plain-text formatting language. Here's what you actually need:

**Headings (this is your "font size" control)**
```markdown
## Big heading
### Medium heading
#### Small heading
```
More `#` = smaller text. Don't use a single `#` — that's reserved for the post title itself. Start with `##` for your main sections (this matches the archetype template already).

**Bold and italic**
```markdown
**this is bold**
*this is italic*
```

**Lists**
```markdown
- bullet one
- bullet two

1. numbered one
2. numbered two
```

**Links**
```markdown
[link text](https://example.com)
```

**Code**
Inline: `` `like this` `` (single backticks)

Block, with syntax highlighting — put the language name right after the first three backticks:
````markdown
```csharp
public class Example { }
```
````

**Quotes**
```markdown
> A quoted line, useful for pulling out an error message or a notable line
```

### Adding images and gifs

1. **Where to put the file**: create a folder per post inside `static/images/`, e.g. `static/images/decompiling-goskills/`, and drop your image or gif in there. Keeping one folder per post stops things getting messy as you add more.

2. **Reference it in the post** like this:
   ```markdown
   ![Screenshot of UE Explorer showing the GOSkills class tree](/images/decompiling-goskills/class-tree.png)
   ```
   The bit in `[square brackets]` is **alt text** — a short description of what's in the image. Always write a real one (not just "screenshot") — it matters for accessibility and it's what shows if the image fails to load.

3. **Gifs work exactly the same way** — just reference the `.gif` file the same as a `.png` or `.jpg`. No special syntax needed.

4. **File size tip**: compress before uploading, especially gifs (they get huge fast). Free tools: [tinypng.com](https://tinypng.com) for images, [ezgif.com](https://ezgif.com) for gifs. Aim to keep individual files under a couple MB so the site stays fast to load.

5. **Recording gifs**: [ScreenToGif](https://www.screentogif.com/) (Windows, free) is the easiest way to record a short clip and export straight to `.gif`. For longer clips, record with **OBS Studio** as an `.mp4` first, then convert the interesting few seconds with ezgif.com.

### General tips

- **Commit often, in small chunks** — don't wait until a whole post is finished to commit. `git commit` after each meaningful chunk of writing, with a short honest message (`"draft: decompiling section"` is fine — it doesn't need to be fancy).
- **You can always undo a draft** — since `draft: true` posts never appear on the live site, feel free to start a post, walk away for a week, and come back. Nothing is public until you flip that flag.
- **Keep a scratch file** while you're actually doing the reverse-engineering work — a plain `.txt` or `.md` file where you jot down what you tried and what broke *in the moment*. Write the actual polished post from that afterward — trying to remember a whole week of debugging from memory is much harder than editing your own notes.
- **Tags**: the `tags: ["fury", ""]` line at the top of each post — add more as relevant (`"networking"`, `"unrealscript"`, `"wireshark"`) — they automatically build a browsable tag page, no extra setup needed.
- **Series**: every post already has `series: ["Reviving Fury"]` at the top — leave that as-is so they group together automatically.
- If a push ever fails with something like "rejected — updates were rejected," run `git pull` first, then try `git push` again. This just means GitHub has something your local folder doesn't (usually only happens if you edit directly on GitHub's website too).

### Quick command cheat-sheet

| What you want to do | Command |
|---|---|
| Start a new post | `hugo new content posts/my-slug.md` |
| Preview locally | `hugo server --buildDrafts` |
| Save your changes | `git add -A` then `git commit -m "message"` |
| Publish (upload + deploy) | `git push` |
| Check what's changed but not committed | `git status` |
