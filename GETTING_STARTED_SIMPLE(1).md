# Fury Dev Log — Total Beginner Guide

## Your links

- **Live blog:** https://jamerlybob.github.io/fury-devlog/
- **Code/repo (where everything actually lives):** https://github.com/Jamerlybob/fury-devlog

This assumes you know **nothing**. Every command below goes in one place only:
**your WSL terminal** (see Part 0 if you're not sure what that means or how to
open it). Don't use PowerShell, Git Bash, or CMD for this project — pick WSL
and stick with it, always.

---

## Part 0: Opening the right terminal, every single time

1. Press the **Windows key**, type `Ubuntu`, press Enter.
2. A black window opens with a prompt that looks like:
   ```
   james@Lats:~$
   ```
   That's WSL. This is the only terminal you should use for this project.
3. Move into your project folder (you'll do this every time you open a new
   terminal window):
   ```bash
   cd /mnt/c/Users/james/OneDrive/Desktop/Coding/fury-devlog
   ```
   Tip: type `cd /mnt/c/Users/james/OneDrive/Desktop/Coding/fury` then press
   **Tab** — it'll autocomplete the rest for you.

You'll know you're in the right place if `ls` (list files) shows things like
`content`, `hugo.yaml`, `themes`.

---

## Part 1: One-time setup (you only ever do this once, on this computer)

If you've already done these steps before, skip to Part 2.

1. **Check Git is installed:**
   ```bash
   git --version
   ```
   If you see a version number, it's installed. If not:
   ```bash
   sudo apt update && sudo apt install git -y
   ```

2. **Tell Git who you are** (needed once, ever):
   ```bash
   git config --global user.email "jamesdalymate@gmail.com"
   git config --global user.name "Jamerlybob"
   ```

3. **Check Hugo is installed:**
   ```bash
   hugo version
   ```
   If it says "command not found", ask me and I'll walk you through
   reinstalling it — this usually means you're accidentally in a different
   terminal (see Part 0).

4. **Install the GitHub CLI** (handles login for you, no passwords to juggle):
   ```bash
   sudo apt update && sudo apt install gh -y
   ```

5. **Log in to GitHub:**
   ```bash
   gh auth login
   ```
   Answer: `GitHub.com` → `HTTPS` → `Yes` → `Login with a web browser`.
   It shows you a code and a link — open the link in your normal browser,
   paste the code, click approve. Done — you won't need to log in again.

---

## Part 2: Your actual everyday workflow

Every time you sit down to work on the blog, the pattern is always the same
four things, in this order:

1. **Write or edit a post**
2. `git add -A` (stage the changes)
3. `git commit -m "a short description"` (save a checkpoint)
4. `git push` (upload it — this is what makes the live site update)

That's it. Everything below is detail on how to do step 1, plus what to do
when something goes wrong.

---

## Part 3: Creating a new post

```bash
hugo new content posts/your-post-name-here/index.md
```

Rules for `your-post-name-here`:
- all lowercase
- use hyphens instead of spaces (`decompiling-goskills`, not `Decompiling GOSkills`)
- no special characters, no apostrophes

This creates a **folder** for your post with a file called `index.md` inside
it. Any images for that post will live in this same folder later (see Part 6).

### Editing the post

Open the whole project in **VS Code** (a free code/text editor — install it
from [code.visualstudio.com](https://code.visualstudio.com) if you don't have
it yet). From your WSL terminal, inside the project folder, run:
```bash
code .
```
That opens VS Code with WSL properly connected — this matters, don't just
double-click VS Code from the Windows Start menu for this project.

In the file browser on the left, open `content/posts/your-post-name-here/index.md`
and write there.

**Never use Notepad or Word to edit these files** — they can silently swap
your straight quotes (`"`) for curly ones (`"` `"`), which breaks things.
VS Code doesn't do this.

### Drafts — what they actually mean

At the top of every new post, you'll see:
```yaml
draft: true
```
While this says `true`, the post **will not appear on your live website at
all** — even after you commit and push. This means you can:
- write half a post, stop, come back next week — nothing is public
- push it to GitHub as a work-in-progress — still nobody can see it live

**When you're ready to actually publish it**, change that line to:
```yaml
draft: false
```
Save the file, then do the usual `git add -A` → `git commit` → `git push`.
A minute or two later it'll be live.

---

## Part 4: Previewing before you publish

From your project folder in WSL:
```bash
hugo server --buildDrafts
```
This shows you drafts too (normally hidden), so you can check your work.
It prints a web address, usually:
```
http://localhost:1313/
```
Open that in your browser. It auto-refreshes as you save changes in VS Code —
leave it running in one terminal window while you write in another, or just
stop it when you're done:
- Click back into the terminal window running it
- Press `Ctrl + C`

---

## Part 5: Saving and publishing your work (the Git part)

Once you've written something and want to save/publish it:

```bash
git add -A
git commit -m "wrote decompiling goskills post"
git push
```

**What if `git push` asks for a username/password?**
It shouldn't, if you did the `gh auth login` step in Part 1. If it does
anyway, just run `gh auth login` again.

**What if `git push` says "rejected" or "fetch first"?**
Run:
```bash
git pull
```
A text editor might pop open asking for a commit message — you don't need
to type anything, just save and close it:
- If it looks like a plain screen with a bar at the bottom (this is **Vim**):
  press `Esc`, then type `:wq`, then press Enter.
- If it shows shortcuts at the bottom like `^O Write Out` (this is **Nano**):
  press `Ctrl+O`, then Enter, then `Ctrl+X`.

Then run `git push` again.

**To avoid that editor popping up ever again**, run this once:
```bash
git config --global core.editor "nano"
```

---

## Part 6: Adding images and gifs

Because each post is its own folder, this is simple:

1. Put the image file directly inside that post's folder — e.g.
   `content/posts/decompiling-goskills/screenshot.png`
   (drag it in with your normal Windows file explorer, or paste it there if
   using Obsidian — see Part 7)

2. In your post's text, reference it by filename only:
   ```markdown
   ![What this image shows](screenshot.png)
   ```
   The text in `[square brackets]` describes the image — always write a real
   description, not just "screenshot".

3. Gifs work exactly the same way — just use the `.gif` filename instead.

4. Before adding, compress if it's large: [tinypng.com](https://tinypng.com)
   for images, [ezgif.com](https://ezgif.com) for gifs.

5. To record a gif: [ScreenToGif](https://www.screentogif.com/) (free,
   Windows) — record, trim, export as `.gif` directly.

---

## Part 7: Writing in Obsidian (optional)

If you want to write posts in Obsidian instead of VS Code:

1. Open the whole `fury-devlog` folder as an Obsidian vault
   (Obsidian → "Open folder as vault").
2. Settings → Editor → turn **off** "Use [[Wikilinks]]".
3. Settings → Files and Links → set "Default location for new attachments"
   to **"Same folder as current file"**.
4. Settings → Files and Links → "Excluded files" → add: `themes`, `public`,
   `resources`, `.github` (keeps clutter out of Obsidian's file browser).

Then: still create new posts using the `hugo new content posts/.../index.md`
command in Part 3 first (this fills in the correct template), and just open
that file in Obsidian to write and paste images. Everything else (previewing,
committing, pushing) still happens in your WSL terminal as normal.

---

## Part 8: Revising an old post

1. In VS Code (or Obsidian), open the post's file:
   `content/posts/the-post-name/index.md`
2. Edit it however you like — fix a typo, add a new section, add more images.
3. Save the file, then do the usual:
   ```bash
   git add -A
   git commit -m "revised decompiling goskills post"
   git push
   ```
4. Give it a minute or two — the live site updates itself automatically.

You can revise a post as many times as you want, published or not. There's
no limit and no "final" version — think of it like editing any normal document.

### Un-publishing a post (without deleting it)

If you want a post to disappear from the live site but keep the file around
(e.g. you're not happy with it yet, or it needs more work):

1. Open its `index.md`
2. Change `draft: false` back to `draft: true`
3. `git add -A` → `git commit -m "unpublish post"` → `git push`

It'll vanish from the live site but the file is still there whenever you
want to bring it back (just flip it to `false` again).

---

## Part 9: Deleting a post completely

1. In VS Code's file browser (left sidebar), find the post's folder under
   `content/posts/` — e.g. `content/posts/decompiling-goskills/`
2. Right-click the folder → **Delete** (this removes the post and any
   images inside it, since they all live together in that one folder)
3. Save if prompted, then in your WSL terminal:
   ```bash
   git add -A
   git commit -m "deleted decompiling goskills post"
   git push
   ```
4. It disappears from the live site within a minute or two.

**Careful**: once you've pushed a deletion, the post is gone from the live
site immediately, though old visitors who bookmarked its exact link will
just get a "page not found." If you're not sure, un-publishing (Part 8) is
the safer, reversible option — deleting is permanent (short of digging
through Git history, which is possible but fiddly, so don't rely on it as
an undo button).

---

## Formatting cheat sheet (Markdown)

```markdown
## Big heading
### Medium heading
#### Small heading

**bold text**
*italic text*

- bullet point
- another bullet

1. numbered item
2. another numbered item

[link text](https://example.com)

`short inline code`

​```csharp
a block of code, with the language name right after the first ​```
​```

> a quoted line
```

---

## Full command cheat-sheet

| What you want to do | Command |
|---|---|
| Open your project folder | `cd /mnt/c/Users/james/OneDrive/Desktop/Coding/fury-devlog` |
| Open the project in VS Code | `code .` |
| Start a new post | `hugo new content posts/your-slug/index.md` |
| Preview the site (including drafts) | `hugo server --buildDrafts` |
| Stop the preview | `Ctrl + C` (in the terminal running it) |
| Check what's changed | `git status` |
| Stage all changes | `git add -A` |
| Save a checkpoint | `git commit -m "describe what you did"` |
| Upload / publish | `git push` |
| Get GitHub's latest changes | `git pull` |
| Log in to GitHub (if push ever fails on auth) | `gh auth login` |
| List files in current folder | `ls` |
| Go up one folder | `cd ..` |
| Unpublish a post (keep the file) | Set `draft: true` in its `index.md`, then commit + push |
| Delete a post completely | Delete its folder under `content/posts/`, then commit + push |

---

## Common errors and what they mean

| What you see | What's actually wrong | Fix |
|---|---|---|
| `hugo: command not found` | You're in the wrong terminal (Git Bash, PowerShell, or CMD instead of WSL) | Open the **Ubuntu** app (see Part 0) |
| `Author identity unknown` | Git doesn't know your name/email yet | Run the two `git config --global` commands in Part 1, step 2 |
| `Invalid username or token. Password authentication is not supported` | GitHub no longer accepts plain passwords | Run `gh auth login` (Part 1, step 5) |
| `updates were rejected` / `fetch first` | GitHub has a change your computer doesn't have yet | Run `git pull`, close the editor that pops up (Part 5), then `git push` again |
| Hugo error mentioning YAML, "value is not allowed", weird line numbers | A file (usually `hugo.yaml`) got edited in Notepad/Word and got curly quotes instead of straight ones | Reopen and re-save the file in VS Code, ask me for a clean copy if needed |
| A weird `^[[200~` before your command | A paste glitch in Git Bash specifically | Not a real error — type the command manually instead of pasting, or just use WSL instead |
