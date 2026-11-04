# Maintaining the content policy branch

Companion to [Per user music content policy](per-user-content-policy.md). The initial deliverable is documentation only; no server behavior, database, client settings, or household playback changes.

## Repository layout

Use a real fork of `music-assistant/server`, retaining history for direct rebases.

| Ref | Purpose |
| --- | --- |
| `upstream/dev` | Upstream development baseline |
| `origin/dev` | Unmodified fork baseline |
| `design/per-user-content-policy` | Design and maintenance documents |
| `feature/per-user-content-policy` | Future implementation branch, created when coding starts |

Inspected base: `c2501db1f6197189d981baf367e43c902569fc20`, 2026-11-04. Preserve this research provenance when recording later validation bases.

## Clone and configure

```sh
git clone https://github.com/kidhasmoxy/server.git music-assistant-content-policy
cd music-assistant-content-policy
git remote add upstream https://github.com/music-assistant/server.git
git fetch upstream dev
git switch design/per-user-content-policy
```

The original workspace checkout may already have both remotes; inspect `git remote -v` first. Never embed credentials in remote URLs.

## Rebase the design

Start with a clean working tree and coordinate with collaborators. Use a unique backup branch name. Save the old remote SHA so a concurrent push cannot be overwritten.

```sh
git status --short
git fetch upstream dev
git fetch origin design/per-user-content-policy
git switch design/per-user-content-policy
old_remote=$(git rev-parse origin/design/per-user-content-policy)
git branch backup/content-policy-before-rebase
git rebase upstream/dev
# Resolve deliberately, git add resolved files, then git rebase --continue.
# To abandon: git rebase --abort
git diff --check upstream/dev...HEAD
git diff --stat upstream/dev...HEAD
git log --oneline upstream/dev..HEAD
git push --force-with-lease=refs/heads/design/per-user-content-policy:$old_remote origin HEAD:design/per-user-content-policy
```

These commands use a POSIX shell. Rebasing rewrites commit IDs: use the explicit lease only on the coordinated feature/design branch, never upstream or a shared default branch. Choose a new backup name on subsequent runs. Preserve unrelated work; do not use an automatic destructive reset.

## Start implementation

```sh
git fetch upstream dev
git switch -c feature/per-user-content-policy upstream/dev
```

Keep the design branch as the specification. If it should accompany a future upstream PR, explicitly cherry-pick its documentation commit instead of merging a stale full branch. Follow the small implementation sequence; do not combine unrelated source cleanup.

If needed, use matching branches in forks of `music-assistant/models` and `music-assistant/frontend`. Record companion SHAs/package versions; do not vendor those projects into this fork. Private server state should not require shared model changes.

## Rebase implementation

1. Record old upstream base, feature head and old remote feature SHA. Create a unique backup branch and fetch upstream.
2. Rebase onto `upstream/dev`; resolve according to behavior rather than accepting either file wholesale.
3. Review `git range-diff OLD_BASE..BACKUP_BRANCH upstream/dev..feature/per-user-content-policy` for dropped or altered patches.
4. Re-audit identity, permissions, persistence, candidate fallback, preload/buffer reuse, routes, grouping and pagination. New upstream paths need coverage even when Git reports no conflicts.
5. Run acceptance tests, isolated migration fixtures, relevant upstream tests and required lint/type/pre-commit checks. Assert denial before audio acquisition/output.
6. Record new upstream SHA, companion SHAs, patch head, test commands/results, client/provider versions and limitations. Push with an explicit lease after review.

Never equate a clean merge with validated enforcement. Reusable audio buffers and fallback selection are particularly sensitive to misplaced filters.

## Upstream contribution

Target `music-assistant/server:dev` under its current contribution instructions. Explain behavior, authority, unrated semantics, queue reset and tests. Keep household identities, hostnames, tokens, speaker IDs and deployment scripts out of public commits.

Submit standalone normalization fixes separately where useful. Document models/frontend dependencies. Do not expose settings before enforcement dependencies land. Publishing this fork does not open an upstream PR or contact maintainers.

## Release and rollback

Until upstream accepts the feature, build a custom image from a reviewed feature SHA using the upstream process. Distinguish it from official images, pin deployment to a digest, and record upstream base and patch head.

Back up auth and durable queue-policy state before upgrade. Test with isolated data and a test player. A stock/older binary ignores this feature: stop protected playback and prevent restricted clients from reconnecting before rollback. Preserve backups; do not assume a downgraded schema understands policy. Deployment and household playback tests belong to a separately authorized task.
