# git-branch-sweep

Classify and delete "done" local git branches — while protecting unpushed work you'd cry to lose.

`git branch --merged` only knows about branches merged with a real merge commit. But most GitHub repos **squash- or rebase-merge** PRs, so the merged commit has a different SHA and `--merged` never lists the branch. You end up with dozens of stale local branches and no safe way to bulk-delete them.

`git-branch-sweep` fixes that. It cross-checks **three** signals to decide what's truly done:

- **GitHub PR state** via `gh` — catches squash/rebase merges that `git branch --merged` misses.
- **git's own merged-into-base test** — catches plain merges even offline.
- **upstream tracking** (`gone` / `ahead`) — catches branches whose remote was deleted.

…and it **never offers to delete local-only work** (a branch with commits but no upstream — a spike you never pushed). Those are the branches you can't get back, so they stay under `keep`.

```
$ git branch-sweep

repo: acme-api · on develop · bases: origin/develop origin/main

safe to delete (3)
  fix/login-race         PR merged on GitHub     merged, 2 days ago
  feature/csv-export     PR merged + in base     merged, 5 days ago
  chore/bump-deps        merged into base branch 1 week ago

review (deletable with --all) (1)
  spike/new-cache        upstream gone (squash/rebase-merged?)  last commit 3 days ago

others - review leftovers, not yours (deletable with --all) (1)
  colleague/their-pr     review leftover - by Jane Dev          last commit 1 day ago

keep (2)
  wip/experiment         local-only, unmerged - spike (protected)  last commit 4 hours ago
  feature/in-progress    active / open work                        last commit 1 hour ago

# → opens an fzf multi-select over safe + review + others; nothing deleted until you confirm
```

## Install

### Homebrew (recommended)

```sh
brew install NaoyaMiyagawa/tap/git-branch-sweep
```

This pulls in `bash`, `gh`, and `fzf` for you.

### Manual (single file, no package manager)

```sh
curl -fsSL https://raw.githubusercontent.com/NaoyaMiyagawa/git-branch-sweep/main/git-branch-sweep \
  -o ~/.local/bin/git-branch-sweep && chmod +x ~/.local/bin/git-branch-sweep
```

Any executable named `git-branch-sweep` on your `PATH` is automatically invokable as `git branch-sweep`.

**Requirements:** `bash` ≥ 4 (macOS ships 3.2 — `brew install bash`; the script re-execs under a modern bash automatically), `git`. Optional: [`gh`](https://cli.github.com/) for the PR cross-check, [`fzf`](https://github.com/junegunn/fzf) for the interactive picker. Without either, the tool degrades gracefully (`--no-gh`, and `--yes` for non-interactive deletes).

## Usage

```
git branch-sweep [options]
```

| Option | Effect |
| --- | --- |
| _(none)_ | classify, print the buckets, then open an fzf multi-select over safe + review + others |
| `-n`, `--dry-run` | classify & print only; delete nothing |
| `-y`, `--yes` | delete the **safe** bucket without prompting |
| `--all` | also include the **review** + **others** buckets (with `-y`, or in the picker) |
| `--protect G` | never touch branches matching glob `G` (repeatable) |
| `--no-gh` | skip the GitHub PR lookup (offline / non-GitHub repo) |
| `--no-fetch` | skip the `git fetch --prune` refresh |
| `-h`, `--help` | show help |

> `git branch-sweep --help` is intercepted by git (it looks for a man page). Use `git branch-sweep -h` or `git-branch-sweep --help`.

## Buckets

| Bucket | Meaning | Deleted by |
| --- | --- | --- |
| **safe** | merged into a base branch **or** its PR is merged, no unpushed commits | picker, `--yes` |
| **review** | upstream branch gone (squash/rebase-merged?), or merged-but-ahead | picker, `--yes --all` |
| **others** | tracks a remote but no commit is yours — a code-review leftover | picker, `--yes --all` |
| **keep** | local-only spike (unmerged, no upstream) **or** active/open work | never |
| **protected** | matched a protect glob, or is current/`develop`/`main`/`master` | never (delete by hand) |

Base branches are detected automatically: `origin/develop`, `origin/main`, `origin/master` (falling back to local branches of the same name).

## Configuration

### Protected branches

The current branch and `develop` / `main` / `master` are always protected. Add your own protected globs (e.g. long-lived worktree homes) via any of the following — later sources add to earlier ones:

```sh
# 1. Environment (space- or colon-separated globs)
export GIT_BRANCH_SWEEP_PROTECT='local/* release/*'

# 2. git config (repeatable, per-repo or --global)
git config --add branch-sweep.protect 'local/*'

# 3. Flag (repeatable)
git branch-sweep --protect 'local/*'
```

Glob-protected branches are still listed under **protected** so you can delete them by hand.

### Ownership (the "others" bucket)

A branch is "yours" if any commit unique to it was authored or committed by your `git config user.email`. Branches that are entirely someone else's work — checked out to review a PR — land in **others**. Add extra identities (e.g. a work email):

```sh
git config --add branch-sweep.email you@work.com
```

If no email is known, the check disables itself and those branches stay under **keep**.

## Safety

- Local-only spikes are **never** offered for deletion — deleting one would lose the only copy.
- Deletion tries `git branch -d` first (safe fast-forward check); only falls back to `-D` for squash/rebase-merged branches that are provably merged via PR or base.
- `--dry-run` and the interactive picker mean nothing is deleted without your explicit confirmation.

## License

[MIT](LICENSE) © Naoya Miyagawa
