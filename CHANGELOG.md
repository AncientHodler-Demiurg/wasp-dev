# Changelog — wasp-dev marketplace

All notable changes to the **wasp-dev marketplace** are documented here. This file tracks marketplace-level versions. The vendored bee plugin's own history lives at [`plugins/bee/CHANGELOG.md`](plugins/bee/CHANGELOG.md). The wasp plugin's own history lives at [`plugins/wasp/CHANGELOG.md`](plugins/wasp/CHANGELOG.md).

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [1.2.0] - 2026-05-14 — `/wasp:cross-pollinate` cross-repository cascade

`wasp` plugin bumped to `1.2.0`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Added (via `wasp` plugin)
- `/wasp:cross-pollinate` — new top-level command that orchestrates `/wasp:pollinate` across a workspace of linked repositories. Auto-infers the dependency graph from package.json scans, runs SCAN → CLOSE → TOPO-SORT to compute the cascade order, then executes serially with live ✅ polling per package and downstream dep-pin updates between hops. Includes `--init` bootstrap wizard, `--dry-run` (mandatory for first runs), `--execute` opt-out, `--batch-approve` for ergonomic flow, and `--resume` for post-failure continuation.
- Workspace state directory `.wasp/` is introduced. Holds `cross-pollinate.yml` (config), `cross-pollinate-state.json` (transient resume state), `cross-pollinate-history.md` (persistent audit log).

See [`plugins/wasp/CHANGELOG.md`](plugins/wasp/CHANGELOG.md) v1.2.0 for the full feature list.

### Unchanged
- `bee` plugin under `plugins/bee/` remains byte-identical to upstream bee-dev 1.9.1.
- `/wasp:pollinate` v1.1.0 multi-package behavior unchanged — cross-pollinate delegates per-package work to it.

---

## [1.1.0] - 2026-05-14 — Multi-package `/wasp:pollinate`

`wasp` plugin bumped to `1.1.0`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Added (via `wasp` plugin)
- `/wasp:pollinate` is now multi-package aware. Auto-detects packages (npm workspaces / `packages/*` / custom dirs), runs a 9-check readiness sweep per package on first run, queues only packages with code changes since their last tag, and shows live ✅ polling through publish — registry → dist-tag → provenance → GitHub Release. See [`plugins/wasp/CHANGELOG.md`](plugins/wasp/CHANGELOG.md) v1.1.0 entry for the full feature list.
- `--batch-approve` flag for ergonomic single-confirmation of the full publish plan.
- Lifecycle config schema upgrade: new `packages: [...]` array. Legacy single-package config still works backwards-compatibly.
- SemVer-correct per-package bumping with Conventional Commits auto-suggest.

### Unchanged
- `bee` plugin under `plugins/bee/` remains byte-identical to upstream bee-dev 1.9.1.
- All 51 `/bee:*` commands work exactly as before.

---

## [1.0.0] - 2026-05-14 — Fork foundation

Initial wasp-dev release. Forks [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev) marketplace `1.9.1` (containing bee plugin `4.5.1`) and adds a complementary `wasp` plugin layering an audit-to-publish lifecycle on top.

### Added
- **`wasp` plugin v1.0.0** — new additive plugin alongside the vendored upstream `bee`. Adds four `/wasp:*` commands:
  - `/wasp:audit-prep` — pre-audit safety check; verifies `.bee/audit-specs/` is clean before the next audit cycle.
  - `/wasp:bundle-audit-specs` — explicitly groups loose audit-specs into per-theme `bundles/<name>/_bundle.md`.
  - `/wasp:unify-audit-specs` — consolidates loose files + bundles into a single mega-bundle `_unified.md`.
  - `/wasp:pollinate` — post-ship publishing pipeline (push, tag, CI publish, GitHub Release, npm registry verify, backfill).
- **Marketplace dual-plugin layout** — `bee` (vendored upstream, unmodified) + `wasp` (our additive layer). Both plugins install independently.
- **Fork lineage documentation** — top-level README.md documents fork origin, upstream tracking workflow, and version compatibility matrix.

### Renamed
- Marketplace name: `bee-dev` → `wasp-dev`
- Marketplace version reset: `1.9.1` → `1.0.0` (this is a new project, not a bee-dev patch level)

### Unchanged (vendored from upstream)
- All 51 `/bee:*` commands work exactly as documented in upstream bee `4.5.1`.
- Bee plugin source under `plugins/bee/` is byte-identical to upstream bee-dev `1.9.1` `plugins/bee/`.
- All hooks, skills, agents, dashboard, scripts under `plugins/bee/` are unchanged.

### Notes
- **Upstream tracking:** `git remote add upstream https://github.com/george-popescu/bee-dev.git` was added. Future bee releases will be absorbed via `git fetch upstream && git merge upstream/main` with conflicts expected only inside `plugins/bee/` (resolve by taking upstream verbatim).
- **State directory policy:** `/wasp:*` commands in this version operate on bee's existing `.bee/` state directory (they extend bee's lifecycle, they don't replace it). Future versions introducing wasp-native artifacts will use a separate `.wasp/` folder.
- **`/bee:` references in `_BeeUpgrade` source material** were rewritten to `/wasp:` only for the four commands wasp owns. All other `/bee:*` references (calling `/bee:audit`, `/bee:ship`, `/bee:commit`, etc.) remain unchanged because those commands still live in the bee plugin.

---

## Fork history

This marketplace was forked from [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev) on 2026-05-14 at upstream version `1.9.1`. The upstream marketplace's pre-fork changelog is preserved at [`plugins/bee/CHANGELOG.md`](plugins/bee/CHANGELOG.md) — that file tracks the bee plugin's version history (currently at `4.5.1`) and is updated only when the bee plugin is re-synced from upstream.
