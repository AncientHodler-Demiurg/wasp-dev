# Changelog — wasp plugin

All notable changes to the **wasp plugin** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [1.0.0] - 2026-05-14 — Initial release

The wasp plugin's first version. Ships the four audit-to-publish lifecycle commands originally drafted as a drop-in upgrade for upstream `bee-dev` v4.3.0 in 2026-05-02. Repackaged here as a standalone plugin in the wasp-dev marketplace alongside the vendored upstream `bee` plugin.

### Added
- `/wasp:audit-prep` — pre-audit safety check (397 lines). Classifies `.bee/audit-specs/` leftovers as archived / in-flight / unprocessed / mis-placed; offers cleanup paths so the next `/bee:audit` cycle starts clean. Flags: `--auto` for non-interactive batch mode.
- `/wasp:bundle-audit-specs` — explicit per-theme bundling (520 lines). Groups loose audit-specs into `bundles/<theme>/_bundle.md` folders. Flags: `--dry-run`.
- `/wasp:unify-audit-specs` — mega-bundle unification (535 lines). Consolidates loose files + bundles into one `_unified.md` with milestone structure; empties `.bee/audit-specs/`. Flags: `--slug <name>`, `--dry-run`.
- `/wasp:pollinate` — post-ship publishing pipeline (1331 lines). Detects npm-backed vs plain repos via first-run wizard; configures `.bee/config.json` lifecycle block; pushes, tags, waits for CI, verifies npm registry, creates + backfills GitHub Releases. Handles the broken `gh release create --notes-from-tag --repo` flag combination via REST API fallback. Includes Step 3.5 spec-target reconciliation (auto-bumps `package.json` to the version cited in the active spec's `requirements.md`). Flags: `--reinit`, `--dry-run`, `--skip-backfill`, `--skip-npm`.

### Provenance
- Tested against `StoaChain/DALOS_Crypto` v3.1.0 high-additive-bundle ship cycle (2026-05-02).
- Verified end-to-end against the public npm registry (`@stoachain/dalos-crypto@3.1.0`).
- 9 GitHub Releases created/backfilled (v1.0.0 through v3.1.0).

### Known limitations addressed in future versions
- **No multi-package support yet.** `/wasp:pollinate` v1.0.0 handles one package per repo. v1.1.0 will add npm-workspaces detection, `packages/*` enumeration, and per-package change detection so monorepos publish only the packages whose code changed since the last release tag.
- **No cross-repo cascade orchestration.** v1.2.0+ will add `/wasp:mass-pollinate` that walks a dependency graph across a workspace of linked repositories, bumping the right `package.json` files in topological order.

---

## Provenance

The four commands originated as a PR-ready upgrade for upstream [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev) authored on 2026-05-02. Rather than waiting for upstream absorption, they were repackaged as the standalone `wasp` plugin so they could be installed alongside upstream bee without modifying it.

License: MIT (matches upstream bee).
