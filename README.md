# wasp-dev

**WASP — Workspace-Aware Spec-based Publisher.** A Claude Code marketplace forked from [bee-dev](https://github.com/george-popescu/bee-dev) that ships the upstream `bee` and `nectar` plugins alongside a complementary `wasp` plugin layering an audit-to-publish lifecycle on top — with multi-package and cross-repository cascade publishing.

## What's in this marketplace

Three plugins ship together. They install independently; you can use any one alone or all three.

### `bee` plugin (vendored from upstream)

The upstream [bee plugin](https://github.com/BEE-CODED/bee-dev) at version `4.8.2`, **unmodified**. Every `/bee:*` command works exactly as documented upstream — the full spec-driven workflow (`/bee:init`, `/bee:new-spec`, `/bee:plan-phase`, `/bee:ship`, `/bee:audit`, `/bee:hive`, etc.), now including Bee Multi-Spec (parallel specs, per-spec memory, worktree promotion).

- Folder: `plugins/bee/`
- Upstream source: [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev)
- License: MIT (inherited)

### `nectar` plugin (vendored from upstream)

The upstream nectar plugin at version `1.0.0`, **unmodified**. A commandless, skills-based distillation of spec-driven development: 9 skills (shape, plan, build, review, honey, debug, audit, capture, orient) and 3 read-only enforcement agents. No state machine — plain artifacts in `docs/work/` are the only state, and "drop the honey" delivers a feature end-to-end from a single confirmation.

- Folder: `plugins/nectar/`
- Upstream source: [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev)
- License: MIT (inherited)

### `wasp` plugin (additive)

A separate plugin that adds four new lifecycle commands and never modifies bee:

| Command | Purpose |
|---|---|
| `/wasp:audit-prep` | Pre-audit safety check — verifies `.bee/audit-specs/` is clean before the next `/bee:audit` cycle |
| `/wasp:bundle-audit-specs` | Group loose audit-specs into per-theme `bundles/<name>/_bundle.md` folders |
| `/wasp:unify-audit-specs` | Consolidate loose files + bundles into a single mega-bundle ready for `/bee:new-spec --from-discussion` |
| `/wasp:pollinate` | Post-ship publishing pipeline: push, tag, wait for CI, verify on npm, create + backfill GitHub Releases. Multi-package monorepo aware (v1.1.0+) — auto-detects packages, queues only the ones whose code changed, polls each package live with ✅ visuals through publish. |
| `/wasp:cross-pollinate` | Cross-repository cascade publisher (v1.2.0+) — orchestrates `/wasp:pollinate` across a workspace of linked repos in dep-graph order, with downstream dep-pin updates between hops. |

- Folder: `plugins/wasp/`
- See [`plugins/wasp/README.md`](plugins/wasp/README.md) for command-level docs

## Why the split

Two plugins, not one renamed fork. The goals are:

1. **Easy upstream absorption.** When `bee-dev` ships a new version, we pull it into `plugins/bee/` unchanged. Zero merge conflicts because we never touched a file in there.
2. **Clear ownership.** `/bee:*` = upstream behavior, exactly. `/wasp:*` = our work, fully on us.
3. **Coexistence.** Workspace docs and CLAUDE.md files that reference `/bee:audit` keep working. Our extensions plug in as new commands rather than overrides.

## State directories

Wasp commands respect bee's domain conventions:

| Folder | Owner | Used for |
|---|---|---|
| `.bee/` | bee | All spec/audit lifecycle data (`STATE.md`, `specs/`, `audits/`, `audit-specs/`, `events/`, `archive/`, `config.json` lifecycle block). The four `/wasp:*` lifecycle commands read and write here because they extend bee's workflow. |
| `.wasp/` | wasp | Wasp-native artifacts that bee doesn't know about (planned: `.wasp/mass-pollinate.yml` cross-repo dep graph in a future version). |

Both folders should be gitignored by default. (bee already does this for `.bee/`; add `.wasp/` to the same boilerplate when wasp introduces files there.)

## Install

```bash
# Add this marketplace
claude plugin marketplace add https://github.com/AncientHodler-Demiurg/wasp-dev

# Install bee (upstream), nectar (upstream), wasp (additions), or any mix
claude plugin install bee
claude plugin install nectar
claude plugin install wasp
```

If you previously had `bee-dev` registered as a marketplace, you can remove it (this fork ships an equivalent `bee` plugin), or keep both registered side-by-side — Claude Code's plugin system namespaces them independently.

## Fork lineage

- **Forked from:** [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev) marketplace `1.9.1`, which contained `bee` plugin `4.5.1`
- **Initial wasp-dev release:** `1.0.0` (2026-05-14)
- **Last upstream sync:** `1.5.0` (2026-07-13) — absorbed bee-dev marketplace `1.11.0`, updating `bee` to `4.8.2` and adding the new vendored `nectar` plugin `1.0.0`.
- **Tracking upstream:** `git remote add upstream https://github.com/george-popescu/bee-dev.git` — `git fetch upstream && git merge upstream/main` to absorb future releases. Conflicts should be limited to `plugins/bee/`, `plugins/nectar/`, and the marketplace/README/CHANGELOG metadata, and resolved by taking upstream verbatim for the vendored plugin trees.

## Versions

| wasp-dev | bee plugin | wasp plugin | Notes |
|---|---|---|---|
| `1.0.0` | `4.5.1` (vendored unmodified) | `1.0.0` | Foundation: split plugins, 4 wasp commands from the audit-to-publish lifecycle, fork hygiene |
| `1.1.0` | `4.5.1` | `1.1.0` | Multi-package detection in `/wasp:pollinate` (npm workspaces, `packages/*`, custom dirs); per-package readiness sweep + live ✅ polling |
| `1.2.0` | `4.5.1` | `1.2.0` | `/wasp:cross-pollinate` cross-repository cascade orchestration with dep-graph traversal, topo-sorted serial execution, and downstream dep-pin updates |
| `1.5.0` | `4.8.2` (vendored unmodified) | `1.4.4` | Upstream sync: `bee` `4.5.1`→`4.8.2` (Multi-Spec, perf passes) + new vendored `nectar` plugin `1.0.0` (skills-based autonomous delivery). `wasp` unchanged. |

## License

MIT. Upstream `bee` and `nectar` plugin source under `plugins/bee/` and `plugins/nectar/` carries upstream's MIT license unchanged. Wasp plugin source under `plugins/wasp/` is original work authored by AncientHodler-Demiurg under the same MIT license.
