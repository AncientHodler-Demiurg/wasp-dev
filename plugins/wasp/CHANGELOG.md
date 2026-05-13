# Changelog — wasp plugin

All notable changes to the **wasp plugin** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [1.1.0] - 2026-05-14 — Multi-package `/wasp:pollinate`

`/wasp:pollinate` learns to handle monorepos: detects every publishable package in a repo, queues only the ones whose code actually changed since their last release tag, validates each package's full publish route end-to-end before saving config, and shows live ✅ polling per package through the publish flow.

### Added

- **Auto-detection of publishable packages** in Stage A (new A7 step). Three probes run in order: (1) npm workspaces field in root `package.json`, (2) `packages/*/package.json` enumeration, (3) single-subdir conventions (`ts/`, `js/`, `src/`). Falls back to repo-root `package.json` for the legacy single-package case.

- **Per-package readiness sweep** (Stage Bm — new substep). For every detected/declared package, runs 9 validation checks before saving config: package.json sanity → registry reachability → auth secret presence on GitHub → PAT scope check → package name claim status → access-mismatch detection → `npm publish --dry-run` → workflow file exists → last-tag readability. Each check renders as `⏳ → ✅/❌/⚠️` live. Wizard halts on any blocking failure.

- **Lifecycle config schema upgrade**: new `packages: [...]` array, each entry declaring `name`, `dir`, `tag_pattern`, `release_title_pattern`, `workflow`, `changelog_path`, `readme_path`, and a `registries: [...]` array (with `kind`, `url`, `access`, `provenance`, `auth_secret` per registry — supports multi-registry packages publishing to both npm and GitHub Packages).

- **Per-package change detection in Step 3** (new). For each declared package: `git diff --name-only <last-tag>..HEAD -- <pkg.dir>` and queue only packages with non-empty diff. Halt early if queue is empty ("no code changed since last release — nothing to publish"). Avoids unnecessary version bumps and churn-publishing in monorepos.

- **Conventional Commits auto-bump suggestion**. Scans `git log <last-tag>..HEAD -- <pkg.dir>` per package: `BREAKING CHANGE:` → suggest MAJOR; `feat:` → suggest MINOR; `fix:` → suggest PATCH; otherwise → PATCH. Pre-selects the suggestion in the per-package bump prompt; user can override.

- **Uniform version model** (Step 3.4 new). With `version_model: "uniform"` (default), all queued packages converge on a shared tag version — the LARGEST of the individually-picked bumps wins. Matches the existing stoa-js / DALOS_Crypto publish workflow design (one shared tag, smart per-package match).

- **Per-package spec target reconciliation** (Step 3.5 rewrite). Spec's `requirements.md` can now declare `**Version target for `@scope/pkg`:** vX.Y.Z` for per-package targets; falls back to global `**Version target:** vX.Y.Z` applied to all queued packages.

- **Per-package documentation gates** (Step 4 rewrite). Each queued package gets its CHANGELOG entry + README version + npm-tarball README checked. Per-package gate summary at the end.

- **Per-package tag annotation** (Step 5 rewrite). Multi-package monorepo tags get a structured body that lists every queued package with its individual CHANGELOG section, plus a "NOT published in this release" footer listing skipped packages.

- **Workflow-grouped CI wait** (Step 8 rewrite). Groups the queue by `pkg.workflow` to produce `$WORKFLOW_RUNS`. Polls each workflow run separately with live `⏳ → ✅` progress. Extracts per-package result from job logs (detects "X@y.z already on npm — skipping" for idempotent re-runs).

- **Per-package live registry polling with ✅ visuals** (Step 9 rewrite). Verifies each queued package SERIALLY through 6 gates: tag pushed → workflow green → npm registry live (5min budget, exponential backoff) → dist-tag = latest → provenance attestation present → GitHub Release created. Each gate renders ⏳ while polling, transitions to ✅ on pass, ❌ on fail. Failures halt the run (downstream packages need verified upstream).

- **Per-package final report** (Step 11 rewrite). Lists every queued package with `✅ published` + npm URL, every skipped package with `⏭️ skipped (no changes)`, every shared Release with link, every CI run with link.

- **`--batch-approve` flag** (new). Presents the full publish plan once and proceeds without per-step prompts after approval. Useful for trusted CI-driven runs; default remains per-step interactive.

- **Backwards compatibility** — legacy single-package config (top-level `npm_dir`, `tag_pattern`, etc.) still works without migration. Step 2 auto-synthesizes a one-entry `packages: [...]` array from legacy fields so downstream loops are uniform.

### Changed

- **Step 1 NO_LIFECYCLE_CONFIG guard**: now accepts EITHER a `packages: [...]` array OR a legacy `npm_dir` field (or `repo_type: "plain"`).

- **Step 2 config resolution**: introduces `$PACKAGES` array as the master per-package config + repo-level defaults table. Effective config display now shows the per-package summary.

- **Step 3 "Read Package + Compute Version"** renamed to **"Compute Publish Queue + Per-Package Versioning"** to reflect the new responsibility.

- **All single-package `$PKG_*` variables** are now per-package fields on `$pkg` records (e.g. `$pkg.next_version`, `$pkg.tag_name`, `$pkg.release_title`).

### Notes

- **State directory unchanged**: `/wasp:pollinate` v1.1.0 still operates on `.bee/` (config + credentials + state). The `.wasp/` directory is reserved for future wasp-native artifacts (e.g. `/wasp:cross-pollinate`'s workspace dep graph in v1.2.0+).
- **`version_model: "independent"`** is reserved for v1.2.0+. v1.1.0 halts with a clear message if requested. Uniform-version is the only model supported in this version.

### Known limitations addressed in future versions

- **Cross-repository cascade publishing** still requires manual coordination across repos. v1.2.0 ships `/wasp:cross-pollinate` to walk a multi-repo dep graph and orchestrate package republishes + downstream dep-pin updates.

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
