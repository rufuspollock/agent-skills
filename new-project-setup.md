# New project setup

Instructions for setting up a new project directory (or bringing an existing one up to standard). Works two ways:

- **Fresh folder**: nothing exists yet — do all steps.
- **Existing project**: some steps already done — detect state, do only what's missing.

Target directory defaults to the current directory. If the user names a different path, `cd` there first (or run commands with that path).

Each step below ends with its own commit (and push, once a remote exists) — don't batch everything into one commit at the end. This keeps history legible and means a failure partway through doesn't leave unrelated changes staged together.

## Step 0: Detect current state

Run before doing anything, and skip steps whose precondition is already met:

```bash
git rev-parse --is-inside-work-tree 2>&1   # is it a git repo?
git remote -v                               # does it have a remote already?
ls README.md AGENTS.md CLAUDE.md 2>&1       # which files already exist?
ls -la .beads 2>&1                          # is beads already set up?
git log --oneline -5 2>&1                   # any commits yet?
```

Report what you found and what you plan to do before acting. If the repo already has a remote and history, most likely only the later steps apply.

## Step 1: README, git init, first commit

- If no `README.md`: stub one (project name as H1, one-line description — ask the user for the description if it's not obvious from context).
- If not a git repo: `git init`.
- Commit:

```bash
git add README.md
git commit -m "docs: stub README"
```

## Step 2: Create GitHub repo and push

Ask the user (skip asking anything already implied):
- Repo name (default: directory name)
- Private or public (**default: private**)

**Remotes use SSH (`git@github.com:...`), not HTTPS.** `gh repo create` has no per-call protocol flag — it uses whatever `gh config get git_protocol` is set to, which may be `https`. Don't rely on that; force it after:

```bash
gh repo create OWNER/REPO --private --source=. --remote=origin --push
git remote set-url origin git@github.com:OWNER/REPO.git
```

(Drop `OWNER/` to create under the authenticated account. Use `--public` if requested. If a remote already exists, skip creating it — but check `git remote -v`; if it's `https://github.com/...`, switch it the same way, then `git push -u origin main` if step 1's commit hasn't been pushed yet.)

## Step 3: AGENTS.md and CLAUDE.md symlink

Do this before beads/changelog — both later steps write into `AGENTS.md`.

- If `AGENTS.md` doesn't exist, create it (a short header is enough; steps 4 and 5 append to it).
- Symlink `CLAUDE.md` to `AGENTS.md` so both assistants read the same file:

```bash
ln -sf AGENTS.md CLAUDE.md
```

- If `CLAUDE.md` already exists as a real file (not a symlink), don't silently overwrite it — check with the user first; it may hold content that should be merged into `AGENTS.md` instead.
- Commit and push:

```bash
git add AGENTS.md CLAUDE.md
git commit -m "docs: add AGENTS.md, symlink CLAUDE.md"
git push
```

## Step 4: Beads

Ask whether to set up Beads for issue tracking (**default: yes**).

If yes, and `.beads` doesn't already exist: follow [[beads-sync-playbook]] in this repo — specifically its "One-time setup in an existing repository" section and the copy/paste setup prompt at the bottom. Key points not to skip:
- `bd init --non-interactive --prefix PROJECT_PREFIX --role maintainer --skip-agents` (prefix = short stable slug from the repo name; `--skip-agents` so it doesn't clobber the `AGENTS.md` from step 3)
- `bd config set export.auto true`
- verify the Dolt remote points at this repo's GitHub origin
- install the recommended sync hooks (`core.hooksPath .beads/hooks`)

Then commit and push, including the Beads-specific push:

```bash
git add -A
git commit -m "chore: set up Beads issue tracking"
git push
bd dolt push
```

If `.beads` already exists, skip — this is an existing setup, not a fresh one.

## Step 5: Changelog instructions

Ask whether to add changelog-writing instructions (**default: yes**).

If yes: fetch `https://raw.githubusercontent.com/life-itself/changelog/main/add-to-agents.md` and add its content to this repo's `AGENTS.md`. Don't paste a stale copy from memory — always fetch fresh, since the upstream file can change.

Commit and push:

```bash
git add AGENTS.md
git commit -m "docs: add changelog instructions to AGENTS.md"
git push
```

## Notes

- This file can grow: if a step needs its own detailed playbook (like beads), keep that as its own Markdown file in this repo and link to it with `[[wikilink]]`-style reference, rather than inlining everything here.
- Always confirm the plan (what's missing vs. already present) before running anything that creates a remote repo or force-pushes.
