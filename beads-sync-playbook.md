# Beads setup and sync playbook

Use this for personal repositories where Beads should sync across your own machines through the repository’s GitHub remote.

## Model

Each clone has a local embedded Dolt database. The repository tracks Beads metadata/configuration, the JSONL export (`.beads/issues.jsonl`) and Dolt’s versioned database data; machine-local database/runtime files stay ignored. Cross-machine synchronization happens through the Beads Dolt remote, not by importing the JSONL export.

The JSONL export is enabled because it is useful for inspection, migration, and tools such as viewers. It is not the backup or sync mechanism, but **it must be committed**: `.beads/issues.jsonl` is a tracked file. It may be empty (0 bytes) until the first issue exists. `bd init` commits its own files before the first export runs, so the file shows as untracked afterwards; `git add` it and commit.

## One-time setup in an existing repository

Run from the repository root. Replace `PROJECT_PREFIX` with a short stable prefix, such as `myrepo`.

```bash
bd init --non-interactive \
  --prefix PROJECT_PREFIX \
  --role maintainer \
  --skip-agents

bd config set export.auto true
bd dolt remote list
bd context
git status --short .beads   # .beads/issues.jsonl untracked? git add and commit it
```

`bd init` normally detects the GitHub `origin` and configures the Dolt remote. Verify it explicitly. The expected remote format is usually:

```text
git+ssh://git@github.com/OWNER/REPOSITORY.git
```

If it was not detected or is wrong:

```bash
bd config set sync.remote git+ssh://git@github.com/OWNER/REPOSITORY.git
bd dolt remote add origin git+ssh://git@github.com/OWNER/REPOSITORY.git
```

Do not use `--stealth` for a repository where the Beads database should be shared with your other machines. Do not use JSONL import/export as the normal sync workflow.

## Recommended hooks

The Beads-managed hooks are installed in `.beads/hooks` and Git is configured with:

```bash
git config core.hooksPath .beads/hooks
```

**Rule: `git push` must also push Beads.** In any repo with Beads installed, a `git push` should push the Beads Dolt data (`refs/dolt/data`) too. Agents asked to "push" or "publish" do both.

**The managed hooks did not do this when last checked (2026-09-23).** `bd hooks run pre-push` only handled backup/export; `refs/dolt/data` on the remote stayed unchanged after `git push`. Re-check in a new repo, in case a newer `bd` does it natively, by comparing `git ls-remote origin refs/dolt/data` before and after a `git push` that has a pending bead change.

So append this to `.beads/hooks/pre-push`, **after** the `# --- END BEADS INTEGRATION ---` marker so that `bd hooks install` leaves it alone. The hooks directory is tracked, so commit it and every clone gets it:

```sh
# --- Dolt sync on git push (local addition, outside the managed block) ---
# bd's managed pre-push hook does not push the Beads Dolt data, so push it here. Never blocks git push; the env guard prevents
# recursion if bd's git-backed remote triggers this hook itself.
if command -v bd >/dev/null 2>&1 && [ -z "$BD_DOLT_PUSH_IN_HOOK" ]; then
  export BD_DOLT_PUSH_IN_HOOK=1
  if ! bd dolt push >&2; then
    echo >&2 "beads: 'bd dolt push' failed; git push continues. Run 'bd dolt push' by hand."
  fi
fi
```

The push runs in the foreground (a few seconds), so its result shows up in the `git push` output. It never fails the git push.

Pulls are not automated yet. Also verify whether `post-merge` / `post-checkout` actually run `bd dolt pull`; if not, add the same kind of guarded wrapper (`bd dolt pull` that never fails the git operation; for `post-checkout`, only when `$3 = 1`). Until then, run `bd dolt pull` after `git pull`.

Also add a line to the repo's `AGENTS.md`/`CLAUDE.md` (and the push step in any repo skill) saying that `git push` also pushes Beads through the hook, and that `bd dolt push` is the manual fallback.

## Normal workflow on either machine

At the start of a session:

```bash
git pull
bd dolt pull
bd status
```

Work with issues normally:

```bash
bd ready
bd create "Short issue title"
bd update PREFIX-abc123 --claim
bd close PREFIX-abc123 --reason "Implemented"
```

At the end of a session:

```bash
bd dolt push
git add -A
git commit -m "Describe the work"
git push
```

With the pre-push hook above, `git push` also pushes Beads, so the explicit `bd dolt push` only confirms it (and covers Beads-only changes when there is no git commit to push).

## New machine or fresh clone

```bash
git clone git@github.com:OWNER/REPOSITORY.git
cd REPOSITORY
bd doctor
bd dolt remote list
bd dolt pull
bd status
```

If Beads says the database is not initialized, make sure the clone contains the tracked `.beads/config.yaml` and `.beads/metadata.json`. Then run:

```bash
bd bootstrap
bd dolt pull
```

Do not run `bd init` over an existing clone unless you have confirmed it is genuinely missing its Beads setup. Reinitialization can require explicit local/remote safety confirmation.

## Verification notes

As last checked, in embedded mode `bd doctor` reported that it is not supported; use `bd context`, `bd status`, and `bd dolt status` instead. `bd config validate` validated the separate federation-backend setting and may report a missing `federation.remote` even when the normal GitHub-backed `sync.remote` is configured correctly. Do not add a fake dolthub/cloud/file URL just to silence that warning.

## If syncing fails

First inspect the state without rewriting anything:

```bash
bd context
bd dolt remote list
bd dolt status
git status --short --branch
```

Then retry the normal sequence:

```bash
git pull
bd dolt pull
bd dolt push
```

If both machines changed Beads concurrently, let Dolt report the conflict and resolve it through the Beads/Dolt workflow. Do not delete `.beads/`, remove the remote, or reinitialize with force as a first response. Keep a copy of any local state before destructive recovery.

If offline, continue working locally. A failed hook pull/push should not block Git operations; synchronize with `bd dolt pull` and `bd dolt push` when connectivity returns.

## `.beads/` already exists in this clone but was never linked to the Dolt remote

This happens when the repo's `.beads/` was committed from a different machine/clone that ran `bd init` independently, and this clone got a working `.beads/embeddeddolt/` some other way (fresh `bd init` here, or an old backup) rather than via `bd bootstrap`. Symptoms:

```bash
bd dolt remote list   # "No remotes configured." even though .beads/config.yaml has sync.remote set
```

or, after adding the remote and pulling/pushing:

```text
Error: merge origin/main: Error 1105: no common ancestor
```

This means the local Dolt commit graph and the remote's were built from two separate `bd init`s — like two unrelated git repos, not two branches of one history. Dolt's merge needs a shared ancestor commit to diff against; with none, it refuses rather than guess.

**Before choosing a recovery path, check whether it actually matters:**

```bash
git fetch origin main -q
git diff origin/main -- .beads/issues.jsonl
```

If this is empty, the git-tracked issue content already matches the remote exactly — only the Dolt binary history diverged, not the data. In that case it's safe to discard the local Dolt db and re-clone:

```bash
bd dolt remote add origin git+ssh://git@github.com/OWNER/REPOSITORY.git   # if not already added
rm -rf .beads/embeddeddolt      # or .beads/dolt/, depending on bd version's embedded-mode layout
bd bootstrap
bd status                       # confirm issue count matches expectations
bd dolt push && bd dolt pull    # confirm sync round-trips cleanly
```

If `git diff` shows real differences, do not discard either side blindly — export both (`bd export`) and reconcile the issues by hand, or treat whichever side has the newer/more-trusted work as authoritative and use `bd dolt push --force` from that side instead.

While setting this up, also check for two easy-to-miss gaps that don't error loudly:

```bash
git config core.hooksPath        # should print .beads/hooks — set it if empty
ls -ld .beads                    # should be 0700; chmod 700 .beads if not
```

Also add `.beads/.auto-import-issues.jsonl` to `.beads/.gitignore` if not already present — it's bd's runtime staging file for auto-import, not the tracked export (`.beads/issues.jsonl`, no leading dot), and differs per machine.

## Copy/paste setup prompt for another repository

```text
Set up Beads in this repository for personal cross-machine syncing.

Requirements:
- Inspect the repo’s existing AGENTS.md/instructions and preserve them.
- Use the installed `bd` version and inspect `bd init --help`, `bd quickstart`, and `bd config --help` before changing anything.
- Initialize embedded Dolt Beads with a stable short issue prefix based on the repository name and `--role maintainer`.
- Do not use stealth mode: `.beads` must be committed and shared.
- Preserve any existing AGENTS.md by passing `--skip-agents` to `bd init`.
- Configure the Beads Dolt sync remote to the repository’s GitHub `origin` using its SSH URL.
- Enable `export.auto true`; explain that JSONL is for interchange/viewers and Dolt remotes are the actual sync mechanism.
- Commit `.beads/issues.jsonl` (tracked, may be empty at first; `bd init` commits before the export runs, so add it afterwards).
- Keep machine-local Dolt databases, sockets, locks, sync state, and logs ignored.
- Ensure Git uses the repo-local `.beads/hooks` via `core.hooksPath`.
- The managed hooks do not sync Dolt. Append the guarded `bd dolt push` from "Recommended hooks" to `.beads/hooks/pre-push`, after the END marker, so `git push` also pushes Beads and never fails because of it. Verify by comparing `git ls-remote origin refs/dolt/data` before and after a push. Add failure-tolerant `bd dolt pull` wrappers for post-merge/post-checkout if those hooks don't already pull.
- Note in `AGENTS.md`/`CLAUDE.md` that `git push` also pushes Beads.
- Verify with `bd context`, `bd dolt status`, `bd dolt remote list`, `bd status`, and Git status. Note the embedded-mode `bd doctor` and federation-only `bd config validate` limitations described in the playbook.
- Do not publish or deploy anything.
- Write or update a local playbook explaining setup, normal pull/work/push, fresh-clone recovery, sync failures, and this prompt.

Before implementation, show the proposed files and commands. After approval, make the changes and report the verification output.
```
