# hen-ui-bump-sandbox

Sandbox repo for testing the **Hen UI auto-bump at merge queue time** (Option A).

This repo is a minimal reproduction of the relevant parts of `HigherEducation/site-publishing`:

```
packages/hen-ui/package.json   ← only the name + version field
package-lock.json              ← only the packages/hen-ui entry
.github/workflows/
  hen_ui_version_check.yml     ← the experimental workflow (see below)
```

---

## What Option A does

Instead of failing when a dev forgets to bump the `hen-ui` version, the workflow auto-bumps at **merge queue time** — the moment GitHub serializes PR merges into `main`.

| Event | Behavior |
|---|---|
| `pull_request` | Soft warning if version not bumped (no fail). Hard fail on version regression or lockfile drift. |
| `merge_group` | Auto-bumps minor if version unchanged. Patches lockfile. Commits + pushes to the queue branch. Fail only on regression. |
| `workflow_dispatch` | Manual trigger — simulates either event for testing without a real PR. |

### Why this is collision-safe

GitHub processes merge queue entries **sequentially**. By the time PR-B's queue entry runs, PR-A has already merged and `main` is at the new version. PR-B bumps from there.

---

## How to test

### 1. Simulate merge_group logic with `workflow_dispatch`

Go to **Actions → Hen UI Version Check → Run workflow**, pick `merge_group`.

This runs the auto-bump script exactly as it would in a real queue entry. You'll see the version bump and lockfile patch in the job output. *(Note: the commit+push step will fail here since `workflow_dispatch` on main doesn't target a queue branch — that's expected. The bump logic itself is what you're validating.)*

### 2. End-to-end merge queue test

> Requires: "Require merge queue" enabled on `main` in Settings → Branches.

1. Create a branch off `main` (e.g., `test/no-bump-pr`)
2. Make any change to `packages/hen-ui/package.json` **without** bumping the version
3. Open a PR targeting `main`
4. The `pull_request` check runs → you should see a **warning** (not a failure) about the missing bump
5. Click **Merge when ready** (or "Add to merge queue") on the PR
6. Watch the **merge_group** check run in Actions — it should:
   - Log `Auto-bumped: 0.17.46 → 0.17.47`
   - Push a `chore(hen-ui): auto-bump to 0.17.47 [merge-queue]` commit
   - Exit green
7. The PR merges. `main` now has version `0.17.47`.

### 3. Test a version regression (should always hard-fail)

1. Create a branch where `packages/hen-ui/package.json` has a **lower** version than `main`
2. Open a PR — the `pull_request` check should immediately fail with `::error::version went backwards`

---

## Enabling the merge queue (one-time setup)

1. Go to repo **Settings → Branches**
2. Edit or add a branch protection rule for `main`
3. Enable **"Require merge queue"**
4. Optionally add `Hen UI Version Check` as a required status check so the queue won't merge unless the auto-bump passes

---

## Promoting to `HigherEducation/site-publishing`

When you're satisfied with the behavior:

1. Copy `.github/workflows/hen_ui_version_check.yml` from this repo into `site-publishing`
2. Ask an org admin to enable merge queue on `main` in `site-publishing`
3. The `GITHUB_TOKEN` has write access on `merge_group` events (unlike fork PRs), so no PAT is needed

The one thing this sandbox **cannot** fully test: the PAT requirement — since `main` here is not protected by branch protection rules (or you have admin access), `GITHUB_TOKEN` pushes work fine. In `site-publishing` the queue branch itself is also not protected (only `main` is), so `GITHUB_TOKEN` should work there too.
