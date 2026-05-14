# CLAUDE.md

This file is your project briefing if you're a Claude Code session opened at `D:\_Claude\wasp-dev`. Read it before doing anything else here. It contains the design, conventions, and workflow specific to this repo.

For user-facing install/usage docs, see [README.md](README.md). For version history, see [CHANGELOG.md](CHANGELOG.md). For broader workspace context (the StoaOuronet workspace that consumes this plugin), look at `D:\_Claude\StoaOuronet\WORKSPACE.md`.

---

## What this repo is

This is a **Claude Code marketplace** named `wasp-dev`. Forked from upstream [BEE-CODED/bee-dev](https://github.com/BEE-CODED/bee-dev) on 2026-05-14. The fork hosts two plugins:

- **`bee`** — the upstream bee plugin (4.5.1), **vendored unchanged** under `plugins/bee/`. Never modify files in there.
- **`wasp`** — our additive layer, original work, under `plugins/wasp/`. This is where ALL development happens.

The marketplace.json at `.claude-plugin/marketplace.json` declares both plugins so a single `claude plugin marketplace add` install gives users both.

Remote: https://github.com/AncientHodler-Demiurg/wasp-dev (origin)
Upstream tracker: https://github.com/george-popescu/bee-dev (upstream)

---

## Critical rule: don't touch `plugins/bee/`

Every file under `plugins/bee/` is vendored from upstream. We keep them byte-identical so future upstream merges produce zero conflicts. Any improvement that affects bee's behavior:

- If it's a bee-internal fix: contribute upstream, then `git pull` here when it ships
- If it's our extension: add it to `plugins/wasp/` instead, even if it conceptually relates to bee

The one allowed exception: `plugins/bee/CHANGELOG.md` may be re-synced when we pull a new upstream version (since upstream maintains it).

---

## Repo layout

```
wasp-dev/                                       ← marketplace root
├── .claude-plugin/
│   └── marketplace.json                         declares BOTH plugins
├── .secrets/
│   └── pat.txt                                  GitHub PAT for pushing this fork (gitignored)
├── README.md                                    user-facing install docs
├── CHANGELOG.md                                 marketplace-level history (per-marketplace-version)
├── LICENSE                                      MIT
├── plugins/
│   ├── bee/                                     ← VENDORED — DO NOT MODIFY
│   │   ├── .claude-plugin/plugin.json           upstream version, byte-identical
│   │   ├── commands/                            51 /bee:* commands
│   │   ├── agents/, hooks/, skills/, dashboard/, scripts/
│   │   ├── README.md, CHANGELOG.md, LICENSE
│   │   └── ...
│   └── wasp/                                    ← OUR PLUGIN
│       ├── .claude-plugin/plugin.json           our version + description + repo URL
│       ├── commands/                            8 /wasp:* command files (.md)
│       ├── README.md                            plugin-level user docs
│       └── CHANGELOG.md                         plugin-level history (per-plugin-version)
```

There are no agents/hooks/skills/dashboard in `plugins/wasp/` (yet) — we ship only commands. If we ever add agents or skills, they'd live in equivalent subfolders here.

---

## How wasp commands are structured

Each command is a single Markdown file at `plugins/wasp/commands/<name>.md`. The file is a **prompt** that Claude executes when the user runs `/wasp:<name>`. Format:

```markdown
---
description: One-paragraph summary of what the command does. Surfaces in /help and autocomplete.
argument-hint: "[--flag1] [--flag2 <value>]"
---

## Current State (load before proceeding)

Read these files using the Read tool:
- `<path>` — what to capture or how to behave if missing

## Git Status (load before proceeding)

Run via Bash:
- ...

## Instructions

You are running `/wasp:<name>` — <one-line role>. Follow these steps in order.

### Step 0: ...
### Step 1: ...
...

## State file schema  (if applicable, v1.4.0+ commands)

```markdown
# Wasp state — {name}
...
```

## Design notes (do not display to user)
- ...
```

The `description` is critical — it's what surfaces in `/` autocomplete and `/help`. Write it as a rich one-paragraph that lists ALL the major capabilities (not a terse line). Look at any existing command's frontmatter for the style.

---

## State file protocol (`.wasp/state.md`)

As of v1.4.0+ (pollinate) and v1.4.1+ (all wasp commands), every wasp command writes `.wasp/state.md` with a unified schema:

```markdown
# Wasp state — {repo_name or workspace_name}

**Command:** {pollinate | audit-prep | bundle-audit-specs | unify-audit-specs | cross-pollinate}
**Run ID:** {ISO 8601 timestamp set at run start}
**Status:** {command-specific status enum}
**Started:** ...
**Last update:** ...
**Wasp plugin version:** 1.4.x
**HEAD at start:** {git SHA, where applicable}
**Mode:** {interactive | batch-approve | dry-run}

## {command-specific tables, gates, run history}

## Failure context
(empty on success; populated on failure with failing step + diagnostic + recovery hint)
```

Lifecycle: Created at the command's "plan approved" step → Updated at each gate transition → Finalized + archived on success (`mv .wasp/state.md .wasp/.archive/state-{run_id}.md`) → Stays in place on failure for `--resume`.

State directory ownership:
- Per-repo commands write to `<repo>/.wasp/state.md`
- Workspace-level commands (only `cross-pollinate` at present) write to `<workspace>/.wasp/state.md`

The state file schema reference lives in `plugins/wasp/commands/pollinate.md` under "## State file schema (`.wasp/state.md`, v1.4.0+)" — the canonical source. Other commands' schema appendix sections reference back to it.

---

## State directory policy (`.wasp/` vs `.bee/`)

| Folder | Owner | Wasp commands' role |
|---|---|---|
| `.wasp/` | wasp (us) | WRITE: `state.md`, `config.json`, `pollinate-credentials/`, `cross-pollinate.yml`, `cross-pollinate-history.md`, `health-history.md`, `forensics-{id}.md`, `debug-sessions/`, `.archive/` |
| `.bee/` | bee (upstream) | READ-ONLY (except for audit-spec commands that legitimately operate on `.bee/audit-specs/` as bee-data manipulators). Wasp NEVER writes to `.bee/STATE.md`, `.bee/specs/`, `.bee/audits/`, or `.bee/config.json`. |

This separation was established in wasp v1.3.0. Pre-1.3.0, pollinate's data lived in `.bee/` — a leak we fixed. Don't reintroduce.

---

## Versioning rules

Strict SemVer 2.0.0 with one wrinkle: marketplace version and wasp plugin version are bumped in lockstep. Same number, in both files.

- **PATCH** (X.Y.Z): backwards-compatible bug fixes or small tweaks. Examples: v1.2.1 (target_branch fix), v1.4.1 (state.md extended to all commands — actually arguably MINOR, but treated as PATCH because it was a follow-up).
- **MINOR** (X.Y+1.0): backwards-compatible feature additions. Examples: v1.1.0 (multi-package pollinate), v1.2.0 (cross-pollinate), v1.4.0 (state.md + --resume), v1.4.2 (debug/forensics/health trio).
- **MAJOR** (X+1.0.0): breaking changes. None yet.

For each release:

1. Edit `plugins/wasp/.claude-plugin/plugin.json` — bump `version`
2. Edit `.claude-plugin/marketplace.json` — bump TWO version fields (marketplace's own `version` AND the `wasp` plugin entry's `version` inside `plugins[]` array)
3. Add entries to BOTH `CHANGELOG.md` (marketplace-level, root) AND `plugins/wasp/CHANGELOG.md` (plugin-level). Format: Keep a Changelog. Use the existing patterns as templates.
4. Don't touch `plugins/bee/.claude-plugin/plugin.json` version — that mirrors upstream.

---

## Release flow (every push)

After making changes (e.g. editing a command file, adding a new one):

```bash
cd D:/_Claude/wasp-dev

# 1. Bump versions (see above)
# 2. Update CHANGELOGs

# 3. Stage + commit
git add .claude-plugin/marketplace.json CHANGELOG.md plugins/wasp/
git commit -m "feat(wasp): vX.Y.Z - <one-line summary>

<body — what changed, why, breaking? backwards-compat notes?>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"

# 4. Tag
git tag -a vX.Y.Z -m "wasp-dev vX.Y.Z — <one-line summary>

<body>

bee plugin unchanged at 4.5.1."

# 5. Push (uses the PAT in .secrets/pat.txt)
TOKEN=$(tr -d '\r\n ' < .secrets/pat.txt)
GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=Never \
  git push "https://x-access-token:${TOKEN}@github.com/AncientHodler-Demiurg/wasp-dev.git" main
GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=Never \
  git push "https://x-access-token:${TOKEN}@github.com/AncientHodler-Demiurg/wasp-dev.git" vX.Y.Z

# 6. Refresh the local install cache + update the installed plugin
claude plugin marketplace update wasp-dev
claude plugin update wasp@wasp-dev
```

After step 6: the user (when they restart Claude Code) will be running the new version. No browser, no marketplace UI involved — the GitHub repo IS the marketplace.

---

## Upstream bee tracking

The wasp-dev repo has an `upstream` remote pointing at `https://github.com/george-popescu/bee-dev.git`. To absorb a new bee version when one ships upstream:

```bash
cd D:/_Claude/wasp-dev
git fetch upstream
git merge upstream/main
```

Expected conflicts: only inside `plugins/bee/` (because we never modify those files manually). Resolve by taking upstream verbatim — `git checkout --theirs plugins/bee/`.

After merge:
- Bump `plugins/bee/.claude-plugin/plugin.json` version to match the new upstream version (e.g., 4.5.1 → 4.6.0)
- Don't bump the wasp plugin version unless we ALSO made wasp changes in this commit
- The marketplace.json's `bee` plugin entry's `version` field should mirror plugins/bee/'s actual version
- Add a brief entry to root `CHANGELOG.md` noting "Synced vendored bee plugin to 4.6.0"
- Tag the marketplace as needed (e.g., if we want to mark this re-sync as a marketplace version bump)

---

## PAT for pushing

`.secrets/pat.txt` contains a GitHub Personal Access Token (classic) for pushing to `AncientHodler-Demiurg/wasp-dev`. Scopes: `repo, workflow, write:packages`. The file is gitignored.

If the PAT expires or gets revoked, generate a new one at https://github.com/settings/tokens (classic, same scopes) and paste into `.secrets/pat.txt`.

---

## Current command roster (v1.4.2)

| File | Role |
|---|---|
| `plugins/wasp/commands/audit-prep.md` | Pre-audit safety check; classifies `.bee/audit-specs/` |
| `plugins/wasp/commands/bundle-audit-specs.md` | Group loose audit-specs into bundles |
| `plugins/wasp/commands/unify-audit-specs.md` | Consolidate everything into a mega-bundle |
| `plugins/wasp/commands/pollinate.md` | Per-repo publish pipeline (multi-package monorepo aware, state.md + resume) |
| `plugins/wasp/commands/cross-pollinate.md` | Workspace-level cascade across linked repos |
| `plugins/wasp/commands/health.md` | Read-only setup-validity diagnostic |
| `plugins/wasp/commands/forensics.md` | Post-mortem analysis of failed runs |
| `plugins/wasp/commands/debug.md` | Open-ended investigation entry point |

The pollinate.md is the largest (~2200 lines after all v1.4.x additions); cross-pollinate.md is the second largest. The audit-spec commands and the diagnostic trio are 250-500 lines each.

---

## Adding a new wasp command (cookbook)

1. Create `plugins/wasp/commands/<name>.md` with the standard front-matter + structure (see "How wasp commands are structured" above)
2. Decide: does this command produce per-repo state.md or workspace state.md? Follow the protocol in pollinate.md.
3. Document the new command's slot in:
   - `plugins/wasp/.claude-plugin/plugin.json` description field (extend if it broadens the plugin's reach)
   - `.claude-plugin/marketplace.json` wasp plugin entry's description
   - Root `README.md` table of commands
   - `plugins/wasp/README.md` per-command reference
   - `plugins/wasp/CHANGELOG.md` v-next entry
4. Bump versions (MINOR if new command, PATCH if just fixing existing)
5. Commit, tag, push per the release flow

---

## When to use which command (for users, for our own reference)

| Symptom / Goal | Command |
|---|---|
| "Is my setup OK?" | `/wasp:health` |
| "This run failed — what happened?" | `/wasp:forensics --active` |
| "I want to publish a single repo's packages" | `/wasp:pollinate` |
| "I want to publish a cascade across multiple repos" | `/wasp:cross-pollinate` |
| "I have an interrupted run, pick up where it left off" | `/wasp:pollinate --resume` or `/wasp:cross-pollinate --resume` |
| "Something is off, where do I start?" | `/wasp:debug` |
| "Audit produced N files, organize them into bundles" | `/wasp:bundle-audit-specs` |
| "Audit produced N files, consolidate into one mega-bundle" | `/wasp:unify-audit-specs` |
| "Run /bee:audit but clean up `.bee/audit-specs/` first" | `/wasp:audit-prep` then `/bee:audit` |

---

## Testing approach

There's no automated test harness for command files (they're prompts, not code). Validation paths:

1. **Static review**: read the modified command file end-to-end after each edit. Check internal consistency, cross-references between steps, no dangling variable references.
2. **Schema validation**: `claude plugin validate <path>` checks plugin.json + marketplace.json shapes.
3. **Dry-run invocation**: `claude plugin marketplace update wasp-dev && claude plugin update wasp@wasp-dev`, restart Claude Code, then invoke the command with `--dry-run` or `--reinit` in a test project to see how the wizard flows.
4. **Live test against the StoaOuronet workspace**: the user's primary test consumer. Pick a low-stakes change (typo fix in CHANGELOG) and run pollinate/cross-pollinate end-to-end. Cost: one disposable patch version.

---

## Memory & cross-session continuity

Project memory for sessions opened here lives at `C:\Users\bicam\.claude\projects\D---Claude-wasp-dev\memory\` (auto-created on first session). Write `<topic>.md` entries + an index in `MEMORY.md` as discoveries accumulate.

Related memory in the StoaOuronet project's memory directory (`C:\Users\bicam\.claude\projects\D---Claude-StoaOuronet\memory\`) has useful context:
- `wasp_dev_plugin.md` — high-level plugin overview
- `command_prefix_convention.md` — /wasp: vs /bee: convention
- `plugin_source_location.md` — repo URLs + paths

You can Read those for backstory, but don't depend on them — write your own memory entries for wasp-dev-specific learnings.

---

## User preferences (from prior sessions)

- **Always use `/wasp:` for our commands**, never `/bee:`. The bee plugin is vendored upstream; wasp is our additive layer. Don't confuse the two.
- **Never push to remote without explicit user request.** Exception: when committing changes that are clearly part of an authorized release (e.g., the user said "ship v1.X.Y" or similar), pushing is part of the release flow. For ambiguous cases, ask.
- **Lockstep versions** between marketplace.json and plugin.json. Never bump one without the other.
- **Don't modify `plugins/bee/`** ever, period. If a bee improvement is needed, file it upstream or implement equivalently in `plugins/wasp/`.
- **Conventional commits** for commit messages (feat / fix / refactor / chore / docs).
- **CRLF line endings** are normal on Windows — git's autocrlf handles conversion. Don't fight it.

---

## Quick reference: development workflow

```bash
# Edit a command file
$EDITOR plugins/wasp/commands/<name>.md

# Bump versions in 2 files (see Versioning rules section)
# Update 2 CHANGELOGs

# Commit + tag + push (use the Bash flow in the Release section above)

# Update local install
claude plugin marketplace update wasp-dev
claude plugin update wasp@wasp-dev

# (restart Claude Code session to surface new commands in /-autocomplete)
```

That's it. Ten files, one workflow, one PAT.

---

## Pointers

- Origin: https://github.com/AncientHodler-Demiurg/wasp-dev
- Upstream: https://github.com/george-popescu/bee-dev
- Consumer workspace (StoaOuronet): `D:\_Claude\StoaOuronet` — has `WORKSPACE.md` describing the linked repos + cross-pollinate config
- Claudstermind (cross-project knowledge base): `D:\_Claude\Claudstermind\` (mentioned in other repo CLAUDE.md files)

# BeeDev
Stack: claude-plugin-marketplace
Wasp's own development is informal (no spec → plan → execute loop). Edit command files, bump versions, commit + tag + push. Use the bee/wasp commands TO develop other projects, but not to develop wasp itself.
