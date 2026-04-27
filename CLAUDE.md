# Cluster Permission Downstream/Upstream Sync Guide

## Repository Relationship

This repo (`stolostron/cluster-permission`) is a **downstream fork** of the upstream community project
[`open-cluster-management-io/cluster-permission`](https://github.com/open-cluster-management-io/cluster-permission).

The upstream repo is cloned separately on disk (for example under a dedicated path for community repos).
The downstream adds Red Hat / ACM-specific files and features on top of the shared upstream codebase.

| Attribute | Downstream (this repo) | Upstream |
|-----------|------------------------|----------|
| GitHub | `stolostron/cluster-permission` | `open-cluster-management-io/cluster-permission` |
| Git remote name | `origin` | `upstream` |
| Branch | `main` | `main` |

## Downstream-Only Files

These files exist ONLY in the downstream fork and must be preserved during syncs:

- **`.tekton/`** -- Tekton pipeline configs for Konflux CI (MCE component builds)
- **`renovate.json`** -- Renovate bot config extending stolostron/acm-config
- **`sonar-project.properties`** -- SonarQube code analysis config
- **`Dockerfile.rhtap`** -- RHTAP/Konflux-oriented container build
- **`COMPONENT_NAME`** -- Downstream component naming for builds
- **`rpms.in.yaml`** / **`rpms.lock.yaml`** -- RPM lock inputs for Konflux/cachi2-style builds
- **`OWNERS`** -- May have additional temporary approvers compared to upstream

## Downstream-Only Commits

The downstream fork contains commits that will never exist in upstream:

- **ACM-\* prefixed commits** -- Red Hat ACM bug fixes and features
- **Tekton file additions** -- `.tekton/` pipeline YAML files for Konflux
- **Konflux/RHTAP commits** -- Red Hat CI/CD pipeline integration
- **Renovate config changes** -- Downstream-specific dependency management

## Historical Sync Patterns

Two methods have been used to sync:

### 1. GitHub "Sync Fork" Button (Bulk Merge)

Creates commits like `Merge branch 'open-cluster-management-io:main' into main`. Brings in
all upstream commits since the last sync as a single merge commit. This is the primary method.

### 2. Manual Cherry-Pick Sync PRs (Targeted)

When a specific upstream PR is urgently needed before the next bulk sync, someone creates a
branch like `sync-upstream-pr-NNNN` that cherry-picks or recreates the upstream commit.
This creates **different SHAs** for the same content, which can cause merge conflicts later.

## How to Sync (Step-by-Step)

### Prerequisites

Ensure the `upstream` remote is configured. It can point to a local clone of upstream or
directly to the GitHub URL:

```bash
# Option A: point to a local clone
git remote add upstream /path/to/upstream_cluster-permission

# Option B: point to GitHub directly
git remote add upstream git@github.com:open-cluster-management-io/cluster-permission.git
```

### Step 1: Fetch upstream

```bash
git fetch upstream main
```

### Step 2: Create a sync branch

```bash
git checkout main
git checkout -b sync-upstream-$(date +%Y%m%d)
```

### Step 3: Merge upstream

```bash
git merge upstream/main --no-ff
```

Using `--no-ff` ensures a merge commit is created, which makes the sync point visible in
history. Individual upstream commits are preserved with their original SHAs.

### Step 4: Resolve conflicts

Conflicts typically appear in **go.mod** and **go.sum** due to
dependency version differences from manual cherry-pick syncs or independent bumps.
If the repository ever vendors dependencies, also expect **vendor/modules.txt**.

Resolution strategy:

- For dependencies that exist in both repos: **take upstream's version** (source of truth)
- For downstream-only dependencies (rare): keep them
- Downstream-only files (`.tekton/`, `renovate.json`, etc.) should rarely conflict; keep the downstream copies

To resolve conflicts manually:

1. Open each conflicted file and find `<<<<<<<`, `=======`, `>>>>>>>` markers
2. For each conflict, keep the `upstream/main` version (between `=======` and `>>>>>>>`)
3. Delete the conflict markers and the `HEAD` version
4. Stage resolved files, for example: `git add go.mod go.sum`

For bulk resolution of go.sum conflicts (takes upstream's version for all):

```bash
python3 -c "
with open('go.sum', 'r') as f:
    lines = f.read().split('\n')
result, skip = [], False
for line in lines:
    if line.startswith('<<<<<<< HEAD'): skip = True; continue
    elif line.startswith('=======') and skip: skip = False; continue
    elif line.startswith('>>>>>>> upstream/'): continue
    elif not skip: result.append(line)
with open('go.sum', 'w') as f:
    f.write('\n'.join(result))
)"
```

If **vendor/** is present in your branch, after resolving modules run:

```bash
go mod vendor
```

### Step 5: Verify downstream-only files

After resolving conflicts, verify these files still exist and are correct:

```bash
ls .tekton/
cat renovate.json
cat sonar-project.properties
test -f Dockerfile.rhtap && echo Dockerfile.rhtap OK
cat COMPONENT_NAME
test -f rpms.in.yaml && test -f rpms.lock.yaml && echo RPM lockfiles OK
cat OWNERS
```

### Step 6: Commit and push

```bash
git commit -m "Sync with upstream open-cluster-management-io/cluster-permission $(date +%Y-%m-%d)"
git push origin sync-upstream-$(date +%Y%m%d)
```

Then create a PR on GitHub targeting `main`.

## Handling Manual/Urgent Cherry-Picks

If a specific upstream PR is needed before the next bulk sync:

1. Identify the upstream PR number and commit SHA
2. Create a branch: `git checkout -b sync-upstream-pr-NNNN`
3. Cherry-pick: `git cherry-pick <upstream-sha>`
4. PR title convention: `:seedling: [sync] <original commit message>`
5. PR body should reference: `Synced from upstream PR: open-cluster-management-io/cluster-permission#NNNN`

**Warning:** Cherry-picks create different SHAs. The next bulk sync merge will likely have
conflicts in go.mod/go.sum where both versions appear. Resolve by taking upstream's version.

## Diagnosing Sync State

To check how far behind downstream is from upstream:

```bash
# Fetch latest upstream
git fetch upstream main

# See commits in upstream that aren't in downstream
git log --oneline upstream/main --not origin/main

# See downstream-only commits
git log --oneline origin/main --not upstream/main
```

If your `origin` is a personal fork and `stolostron/cluster-permission` is another remote, substitute that remote name for `origin` in the `git log` examples so you compare against the downstream integration branch you care about.
