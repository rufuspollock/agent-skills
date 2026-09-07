# Beads setup and sync playbook

Use this for personal repositories where Beads should sync across your own machines through the repository’s GitHub remote.

## Model

Each clone has a local embedded Dolt database. The repository tracks Beads metadata/configuration and Dolt’s versioned database data; machine-local database/runtime files stay ignored. Cross-machine synchronization happens through the Beads Dolt remote, not by importing the JSONL export.

The JSONL export is enabled because it is useful for inspection, migration, and tools such as viewers. It is not the backup or sync mechanism.

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

For convenient personal sync, add these small wrappers after the Beads-managed sections:

- `post-merge`: run `bd dolt pull`, but do not fail `git pull` if the network is unavailable.
- `post-checkout`: run `bd dolt pull` only when the checkout was a branch switch or clone update (`$3 = 1`), and do not fail checkout if it cannot reach the remote.
- `pre-push`: start `bd dolt push` in the background and write output to an ignored `.beads/dolt-push.log`.

Because the pre-push sync is backgrounded, run this when you need confirmation that the Beads push completed:

```bash
bd dolt push
```

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
bd update wilberwiki-abc123 --claim
bd close wilberwiki-abc123 --reason "Implemented"
```

At the end of a session:

```bash
bd dolt push
git add -A
git commit -m "Describe the work"
git push
```

The explicit `bd dolt push` is recommended at session end even when the pre-push hook also starts a background push. It makes handoff and shutdown deterministic.

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

## Verification notes for bd 1.1.2

In embedded mode, `bd doctor` reports that it is not yet supported; use `bd context`, `bd status`, and `bd dolt status` instead. `bd config validate` currently validates the separate federation-backend setting and may report a missing `federation.remote` even when the normal GitHub-backed `sync.remote` is configured correctly. Do not add a fake dolthub/cloud/file URL just to silence that warning.

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
- Keep machine-local Dolt databases, sockets, locks, sync state, and logs ignored.
- Ensure Git uses the repo-local `.beads/hooks` via `core.hooksPath`.
- Add failure-tolerant hooks: pull after `git pull`/merge and branch checkout, and start a non-blocking `bd dolt push` during `git push` with an ignored log file.
- Verify with `bd context`, `bd dolt status`, `bd dolt remote list`, `bd status`, and Git status. Note the embedded-mode `bd doctor` and federation-only `bd config validate` limitations described in the playbook.
- Do not publish or deploy anything.
- Write or update a local playbook explaining setup, normal pull/work/push, fresh-clone recovery, sync failures, and this prompt.

Before implementation, show the proposed files and commands. After approval, make the changes and report the verification output.
```
