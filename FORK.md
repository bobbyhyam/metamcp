# Fork notes — bobbyhyam/metamcp

This is a personal fork of [metatool-ai/metamcp](https://github.com/metatool-ai/metamcp), maintained for self-hosting MetaMCP as an aggregator that exposes local MCP servers to claude.ai.

Upstream entered low-bandwidth maintenance mode in 2026-02 (see upstream `recent-updates.md`); this fork tracks upstream merges and adds local customizations.

## Branch layout

- **`main`** — clean mirror of `metatool-ai/metamcp:main`. Never edit directly. Auto-synced by `.github/workflows/sync-upstream.yml`.
- **`custom`** — default branch on this fork. All fork-specific changes live here, including everything in this file's "Fork-only files" list below.

```
upstream/main ──► main (auto, ff-only, weekly)
                   │
                   └──► custom (manual: git merge main, resolve conflicts, push)
```

### Updating `custom` with upstream changes

```bash
git checkout main && git pull
git checkout custom
git merge main
# resolve any conflicts, commit, push
git push origin custom
```

## Releases (Docker images to GHCR)

Tags matching `fork-vX.Y.Z` (or pre-release variants like `fork-vX.Y.Z-rc1`) on the `custom` branch trigger `.github/workflows/fork-docker-publish.yml`, which publishes multi-arch images to:

- `ghcr.io/bobbyhyam/metamcp:X.Y.Z`
- `ghcr.io/bobbyhyam/metamcp:fork-latest`

```bash
git checkout custom
git tag fork-v1.0.0
git push origin fork-v1.0.0
```

The `fork-v` prefix is intentional: it does not match upstream's tag filter (`vX.Y.Z`, `vX.Y.Z-test`, `vX.Y.Z-docker-per-mcp`), so upstream's `docker-publish.yml` will not fire on our tags. They live in parallel without colliding.

## Sending PRs back to upstream

**Rule: branch upstream-bound work from `main`, never from `custom`.** `main` is a clean mirror of upstream, so a branch off it has none of the fork-specific files and produces a clean diff for upstream.

```bash
git checkout main && git pull
git checkout -b feature/cool-thing
# work, commit, push
gh pr create --repo metatool-ai/metamcp \
  --base main --head bobbyhyam:feature/cool-thing
```

If a useful change already exists on `custom`, cherry-pick it onto a fresh `main`-based branch instead of reworking it:

```bash
git checkout main && git pull
git checkout -b feature/cool-thing
git cherry-pick <sha-from-custom>
```

## Fork-only files (do not include in upstream PRs)

These exist only on `custom`. They are filename-prefixed `fork-` (or named `FORK.md`) so they are visually obvious if accidentally staged into an upstream-bound branch.

- `FORK.md` — this file
- `.github/workflows/sync-upstream.yml` — weekly upstream sync
- `.github/workflows/fork-docker-publish.yml` — fork's GHCR publish

## Customizations log

Append entries here as the fork diverges. Each entry: date, brief change description, and motivation. This helps future-you remember why the diff exists when an upstream merge conflicts.

- *(none yet)*
