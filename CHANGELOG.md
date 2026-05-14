# Changelog — wasp-dev marketplace

All notable changes to the **wasp-dev marketplace** are documented here. This file tracks marketplace-level versions. The vendored bee plugin's own history lives at [`plugins/bee/CHANGELOG.md`](plugins/bee/CHANGELOG.md). The wasp plugin's own history lives at [`plugins/wasp/CHANGELOG.md`](plugins/wasp/CHANGELOG.md).

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [1.4.4] - 2026-05-14 — SessionStart banner + `/wasp:cross-pollinate --add-member` wizard

`wasp` plugin bumped to `1.4.4`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Added (via `wasp` plugin)

Two quality-of-life additions:

- **SessionStart hook** — a new `plugins/wasp/hooks/hooks.json` registers a `SessionStart` hook that runs `plugins/wasp/scripts/session-start.sh`. The script auto-detects whether CWD is a wasp-managed workspace (walks up looking for `.wasp/cross-pollinate.yml`) and/or a wasp-managed repo (`.wasp/config.json` in CWD), then prints a banner showing wasp version, workspace name + repo/edge counts, active repo context (name, lifecycle type, current branch), and any in-flight state.md alerts. Silent exit when CWD has no wasp data. No new dependencies — pure bash + plugin.json read.

- **`/wasp:cross-pollinate --add-member`** — interactive wizard for adding a new repo to an existing workspace without re-inferring the full dep graph. 7-stage flow: (M-A) identify repo by path or git clone URL; (M-B) detect role (consumer / publisher / both); (M-C) multi-select workspace packages to depend on with per-edge dep-type choice and optional auto-add to package.json; (M-D) generate per-repo `.wasp/config.json` + `pollinate-credentials.md` + scaffolded `.secrets/pat.txt` + `.gitignore` adds; (M-E) atomic update of `cross-pollinate.yml` (append repo + edges); (M-F) re-render `dep-graph.md` via the v1.4.3 template; (M-G) summary with next-steps checklist. Uses `.wasp/state.md` for crash recovery. Fully reversible if user cancels.

### Use case

Both features serve the multi-consumer workspace pattern (one shared core, N consumer apps). The SessionStart banner makes the wasp version + workspace shape visible at every session — no more "is this still wasp-managed? what version?" friction. The --add-member wizard makes bringing a new consumer (or publisher) into the workspace a guided 30-second operation instead of a hand-edited 5-step checklist.

### Unchanged
- `bee` plugin pristine at 4.5.1.

---

## [1.4.3] - 2026-05-14 — Auto-generated `.wasp/dep-graph.md` + cascade subgraph visualization

`wasp` plugin bumped to `1.4.3`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Added (via `wasp` plugin)

Three small additions that surface the workspace dependency graph as a human-readable artifact, ending the historical state where the graph lived only as a flat YAML edge list:

- **`/wasp:cross-pollinate --init/--reinit`** now generates `.wasp/dep-graph.md` alongside `cross-pollinate.yml` at the workspace root. The file contains an ASCII layer model, a Mermaid diagram (renders natively on GitHub), an edge table, per-package cascade scenarios ("what happens when X publishes?"), and a version state snapshot (local vs npm). Regenerated on every `--init/--reinit` from current package.json scans. Step 0.6.5 in the command.
- **`/wasp:cross-pollinate --dry-run`** Step 6 now displays an "Affected dep subgraph" preview before the "Will publish" list — shows only the edges this cascade will traverse, with old→new version transitions highlighted. Edges not participating in the cascade are summarized as a count with a pointer to `.wasp/dep-graph.md` for the full view.
- **`/wasp:health`** Check W4.5 catches drift between `cross-pollinate.yml` and `dep-graph.md` via mtime comparison. WARN-level only (drift is cosmetic, not functional). Auto-fix under `--fix` re-renders dep-graph.md from current cross-pollinate.yml without re-inferring edges.

### Notes
- `dep-graph.md` is a one-way derivation from `cross-pollinate.yml` — wasp commands never read it back. If you want to change edges, edit `cross-pollinate.yml` (or run `--reinit`), not dep-graph.md.
- The 16th health check (W4.5) brings the total to 5 workspace checks + 12 per-repo checks.

### Unchanged
- `bee` plugin pristine at 4.5.1.

---

## [1.4.2] - 2026-05-14 — `/wasp:health`, `/wasp:forensics`, `/wasp:debug`

`wasp` plugin bumped to `1.4.2`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Added (via `wasp` plugin)
Three new diagnostic commands that parallel bee's `/bee:health` + `/bee:forensics` + `/bee:debug` trio, adapted for wasp's data shape (state.md, cross-pollinate.yml, .wasp/.archive/):

- `/wasp:health` — read-only setup validation. 5 workspace checks + 12 per-repo checks (configs, credentials, PAT scopes, GitHub secrets, dep graph integrity, no orphaned state.md, etc.). PASS/WARN/FAIL per check with recovery hints. Optional `--fix` flag auto-remediates WARN-level issues. Appends to `.wasp/health-history.md`.
- `/wasp:forensics` — post-mortem analysis of a specific failed/stuck run. Targets active or archived state.md, reconstructs timeline, cross-checks recorded gates against external reality (git/npm/GitHub), diagnoses root cause (clean failure / divergence / silent stall), suggests recovery actions ranked.
- `/wasp:debug` — open-ended investigation entry point. Auto-discovers context, ranks hypotheses, walks user through targeted drill-down via AskUserQuestion. May dispatch to /wasp:health or /wasp:forensics for deeper analysis.

All three are read-only by default. None auto-fixes (recovery remains user's call).

Wasp now ships 8 commands total. See [`plugins/wasp/CHANGELOG.md`](plugins/wasp/CHANGELOG.md) v1.4.2 for the full feature list.

### Unchanged
- `bee` plugin pristine at 4.5.1.

---

## [1.4.1] - 2026-05-14 — `.wasp/state.md` extended to all wasp commands

`wasp` plugin bumped to `1.4.1`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Added (via `wasp` plugin)
- `/wasp:audit-prep`, `/wasp:bundle-audit-specs`, `/wasp:unify-audit-specs` now write `.wasp/state.md` per-repo (init + finalize lifecycle, archived to `.wasp/.archive/state-{run_id}.md` on success).
- `/wasp:cross-pollinate` state file migrated from JSON (`cross-pollinate-state.json`) to markdown (`state.md`) for consistency with the per-repo format.
- Unified state.md header across all 5 wasp commands: `# Wasp state — {name}` + `**Command:** {command_name}` field.
- In-flight collision detection — every wasp command checks for an existing in-flight state.md and warns before overwriting.

See [`plugins/wasp/CHANGELOG.md`](plugins/wasp/CHANGELOG.md) v1.4.1 for the full feature list.

### Deferred to v1.4.2
- `/wasp:debug`, `/wasp:forensics`, `/wasp:health` commands — wasp-side parallels of bee's debug commands.

### Unchanged
- `bee` plugin pristine at 4.5.1.

---

## [1.4.0] - 2026-05-14 — `/wasp:pollinate` state file + `--resume` support

`wasp` plugin bumped to `1.4.0`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Added (via `wasp` plugin)
- `.wasp/state.md` per-repo state file for `/wasp:pollinate`. Tracks queue + per-package gate progress + run history + failure context. Updated at every gate transition through the publish pipeline.
- `--resume` flag for `/wasp:pollinate`. Reads `.wasp/state.md`, drift-checks against current HEAD, and jumps to the first incomplete gate.
- State file is archived to `.wasp/.archive/state-{run_id}.md` on successful run completion — useful for forensics + retrospective analysis.

### Notes
- Cross-pollinate's existing workspace state (`.wasp/cross-pollinate-state.json`) is unchanged. When it invokes pollinate per repo, each pollinate sub-invocation writes its own per-repo state.md.
- Future versions may add equivalent state tracking to the audit-spec lifecycle commands and may add `/wasp:debug` / `/wasp:forensics` / `/wasp:health` commands to inspect wasp-side state (parallel to bee's debug commands which are bee-infrastructure-focused).

See [`plugins/wasp/CHANGELOG.md`](plugins/wasp/CHANGELOG.md) v1.4.0 for the full feature list.

### Unchanged
- `bee` plugin pristine at 4.5.1.

---

## [1.3.0] - 2026-05-14 — Wasp state moves to `.wasp/` namespace

`wasp` plugin bumped to `1.3.0`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Changed (via `wasp` plugin)
- **State directory cleanup**: pollinate's per-repo credentials + lifecycle config moved from `.bee/` to `.wasp/`. This fixes a cross-namespace leak where a wasp command was writing to bee's data folder (a hangover from the original `_BeeUpgrade` source material's assumption that pollinate would be upstreamed into bee).
- **Auto-migration on first run**: pollinate's Step 0.1 detects legacy `.bee/` layout and offers a non-destructive migration. Bee's other config fields (`stacks`, `implementation_mode`, etc.) in `.bee/config.json` are preserved untouched; only the `lifecycle:` key is moved into a new `.wasp/config.json` file.
- Cross-pollinate's per-member-repo init check updated to look at `.wasp/pollinate-credentials/` instead of `.bee/`.

See [`plugins/wasp/CHANGELOG.md`](plugins/wasp/CHANGELOG.md) v1.3.0 for the full migration details.

### Unchanged
- `bee` plugin pristine at 4.5.1.
- `/bee:*` commands keep using `.bee/` as before — they're bee's commands operating on bee's data.
- Wasp's audit-spec lifecycle commands (`/wasp:audit-prep`, `/wasp:bundle-audit-specs`, `/wasp:unify-audit-specs`) still operate on `.bee/audit-specs/` — that's bee-owned data; wasp commands manipulate it but don't own it.

---

## [1.2.1] - 2026-05-14 — `target_branch` config + non-main branch support

`wasp` plugin bumped to `1.2.1`. `bee` plugin unchanged at `4.5.1` (vendored upstream).

### Added (via `wasp` plugin)
- `/wasp:pollinate` lifecycle config gains a new `target_branch` field. Pollinate's push step now respects this instead of hardcoding `origin/main`. Auto-detected from the git remote's default branch during the wizard; overridable to any branch (e.g. `dev` for consumer apps like OuronetUI).
- Example in schema appendix: "Consumer app on a non-main branch" showing the `target_branch: "dev"` pattern with `version_file_path` pointing at `src/constants/version.ts`.

### Unchanged
- `bee` plugin pristine at 4.5.1.
- Existing configs without `target_branch` continue to work — defaults to `"main"` for backwards compatibility.

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
