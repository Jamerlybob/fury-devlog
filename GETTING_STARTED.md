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
hugo new content posts/decompiling-goskills/index.md
```
Note the `/index.md` at the end — this creates a **page bundle**: a folder
(`decompiling-goskills/`) with your post inside as `index.md`. Any images or
gifs for that post live in the *same folder*, right next to it. This matters
a lot if you're writing in Obsidian — see the Obsidian section below.

(Pick your own folder name — lowercase, hyphens instead of spaces, no spaces or special characters.)

This creates the file pre-filled with the template structure (Where I left off / What I tried / What broke / What I learned / Next up) and `draft: true` at the top.

Open it in any text editor (VS Code is a good free one) — or Obsidian, see below — and write.

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

Because posts are now page bundles (a folder per post), images just live
**in the same folder as the post itself** — no separate images folder to
manage, no long paths to type.

1. Drop your image or gif directly into the post's folder, e.g.
   `content/posts/decompiling-goskills/class-tree.png`

2. Reference it in the post with just the filename:
   ```markdown
   ![Screenshot of UE Explorer showing the GOSkills class tree](class-tree.png)
   ```
   No `/images/...` path needed — since the image sits right next to the
   post, a bare filename is all Hugo needs.

3. **Gifs work exactly the same way** — just reference the `.gif` file the same as a `.png` or `.jpg`. No special syntax needed.

4. **File size tip**: compress before uploading, especially gifs (they get huge fast). Free tools: [tinypng.com](https://tinypng.com) for images, [ezgif.com](https://ezgif.com) for gifs. Aim to keep individual files under a couple MB so the site stays fast to load.

5. **Recording gifs**: [ScreenToGif](https://www.screentogif.com/) (Windows, free) is the easiest way to record a short clip and export straight to `.gif`. For longer clips, record with **OBS Studio** as an `.mp4` first, then convert the interesting few seconds with ezgif.com.

---

## Part 3: Writing in Obsidian (optional, but recommended)

Obsidian writes plain markdown files, and Hugo reads plain markdown files —
they work well together with a couple of one-time settings changes.

### One-time Obsidian setup

1. **Open the whole `fury-devlog` folder as an Obsidian vault** (Obsidian → "Open folder as vault" → pick `fury-devlog`). This gives you the whole project in Obsidian's file browser, not just a "posts" folder floating on its own.

2. **Turn off Wikilinks**, so Obsidian writes normal markdown links (which Hugo understands) instead of Obsidian's own `[[double-bracket]]` style:
   - Settings → Editor → toggle **"Use [[Wikilinks]]"** off

3. **Set the attachment folder to "same folder as current file"**, so anything you paste or drag in lands right next to the post it belongs to (this is what makes the page-bundle structure above pay off):
   - Settings → Files and Links → **"Default location for new attachments"** → choose **"Same folder as current file"**

4. **Hide the technical folders from cluttering your Obsidian file browser** (theme files, build output, etc. — you'll never edit these):
   - Settings → Files and Links → **"Excluded files"** → add: `themes`, `public`, `resources`, `.github`

### The actual workflow

- Still use `hugo new content posts/your-slug/index.md` from a terminal to create each new post — this makes sure the frontmatter template (title, date, tags, series) is filled in correctly, which Obsidian alone won't do for you.
- Then just open that post folder in Obsidian and write. Paste screenshots straight from your clipboard (`Ctrl+V`) — Obsidian saves the file into the same folder automatically and inserts the correct markdown link for you, because of the attachment setting above.
- Drag-and-drop gifs the same way.
- When you want to preview it properly (exact fonts, layout, dark mode, etc. — not just Obsidian's own preview), run `hugo server --buildDrafts` in a terminal alongside Obsidian and check `http://localhost:1313/`.
- Commit and push as normal (`git add -A`, `git commit -m "..."`, `git push`) once you're happy — Obsidian doesn't need to know about Git at all; you can do that from a terminal or a Git GUI like GitHub Desktop if you prefer not to touch the command line for that part.

### A couple of nice extras once this is set up

- **Backlinks/graph view**: if you link between your own posts or to the glossary using standard markdown links (e.g. `[UnrealScript](/glossary/#unrealscript)`), Obsidian will still generally track these for its backlinks panel, so you get a lightweight map of how your posts connect — a nice side benefit of writing in Obsidian over a plain text editor.
- **Templates**: Obsidian's built-in Templates core plugin can be handy for quick scratch notes (debugging logs, ideas) that *aren't* meant to become posts — keep those outside the `content/` folder entirely (e.g. a `notes/` folder at the project root) so Hugo never tries to build them into pages.

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
