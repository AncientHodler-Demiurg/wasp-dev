# wasp-dev

**WASP — Workspace-Aware Spec-based Publisher.** A Claude Code marketplace forked from [bee-dev](https://github.com/george-popescu/bee-dev) that ships the upstream `bee` plugin alongside a complementary `wasp` plugin layering an audit-to-publish lifecycle on top — and, in subsequent versions, multi-package and cross-repository cascade publishing.

## What's in this marketplace

Two plugins ship together. They install independently; you can use bee alone, wasp alone, or both.

### `bee` plugin (vendored from upstream)

The upstream [bee plugin](https://github.com/BEE-CODED/bee-dev) at version `4.5.1`, **unmodified**. Every `/bee:*` command works exactly as documented upstream. The 51 upstream commands (`/bee:init`, `/bee:new-spec`, `/bee:plan-phase`, `/bee:ship`, `/bee:audit`, `/bee:hive`, etc.) are all here.

- Folder: `plugins/bee/`
- Upstream source: [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev)
- License: MIT (inherited)

### `wasp` plugin (additive)

A separate plugin that adds four new lifecycle commands and never modifies bee:

| Command | Purpose |
|---|---|
| `/wasp:audit-prep` | Pre-audit safety check — verifies `.bee/audit-specs/` is clean before the next `/bee:audit` cycle |
| `/wasp:bundle-audit-specs` | Group loose audit-specs into per-theme `bundles/<name>/_bundle.md` folders |
| `/wasp:unify-audit-specs` | Consolidate loose files + bundles into a single mega-bundle ready for `/bee:new-spec --from-discussion` |
| `/wasp:pollinate` | Post-ship publishing pipeline: push, tag, wait for CI, verify on npm, create + backfill GitHub Releases |

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

# Install bee (upstream), wasp (additions), or both
claude plugin install bee
claude plugin install wasp
```

If you previously had `bee-dev` registered as a marketplace, you can remove it (this fork ships an equivalent `bee` plugin), or keep both registered side-by-side — Claude Code's plugin system namespaces them independently.

## Fork lineage

- **Forked from:** [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev) marketplace `1.9.1`, which contained `bee` plugin `4.5.1`
- **Initial wasp-dev release:** `1.0.0` (2026-05-14)
- **Tracking upstream:** `git remote add upstream https://github.com/george-popescu/bee-dev.git` — `git fetch upstream && git merge upstream/main` to absorb future bee releases. Conflicts should be limited to `plugins/bee/` and resolved by taking upstream verbatim.

## Versions

| wasp-dev | bee plugin | wasp plugin | Notes |
|---|---|---|---|
| `1.0.0` | `4.5.1` (vendored unmodified) | `1.0.0` | Foundation: split plugins, 4 wasp commands from the audit-to-publish lifecycle, fork hygiene |
| `1.1.0` (planned) | `4.5.1` | `1.1.0` | Multi-package detection in `/wasp:pollinate` (npm workspaces, `packages/*`, custom dirs) |
| `1.2.0+` (planned) | `4.5.1` | `1.2.0+` | `/wasp:mass-pollinate` cross-repo cascade publishing |

## License

MIT. Upstream bee plugin source under `plugins/bee/` carries upstream's MIT license unchanged. Wasp plugin source under `plugins/wasp/` is original work authored by AncientHodler-Demiurg under the same MIT license.
