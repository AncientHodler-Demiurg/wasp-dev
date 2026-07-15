# Retainer 01 — Nectar sync handoff

> **Purpose:** This file is a self-handoff written at the end of a Claude Code conversation so the *next* fresh session retains the context we built. Read this first if you're a new session opened in `D:\_Claude\_Tools\wasp-dev` and the user references "the nectar sync", "the retainer", or asks to continue that work. Retainers are numbered `retainer-01.md`, `retainer-02.md`, … — newer numbers supersede older ones for overlapping topics, but each documents its own episode. This is **01**.

**Written:** 2026-07-13
**Session model:** Claude Opus 4.8
**Repo:** `D:\_Claude\_Tools\wasp-dev` (marketplace `wasp-dev`), branch `main`
**HEAD at handoff:** `65c3929` (pushed to origin)

---

## 1. Why a fresh session is being started

The user installed a new plugin (**nectar**) during this conversation. Skills load at CLI **startup**, and this conversation was carrying **pre-restart context** — the running session's available-skills list still showed only `bee:*` and `wasp:*`, **no `nectar:*`**. Reopening the app but *resuming* the old conversation keeps the stale skill context. So the user is starting a **genuinely new conversation** to force a clean load of nectar's skills. This handoff exists so that new session doesn't lose what we did.

**First thing the new session should help verify:** that nectar's skills are actually loaded. Test by asking *"what nectar skills do you have available?"* — a loaded nectar exposes: **shape, plan, build, review, honey, debug, audit, capture, orient**. If they don't appear in a genuinely fresh session, that's a real problem to troubleshoot (scope/enable/marketplace-cache), because the install itself is confirmed enabled (see §3).

---

## 2. What we accomplished this session (the upstream sync)

George Popescu (upstream `github.com/george-popescu/bee-dev`, aka BEE-CODED) shipped **nectar v1.0.0** as a *second* plugin **alongside** bee (not a rename, not a replacement). We absorbed the whole upstream into this fork.

**Result — marketplace `wasp-dev` now ships THREE plugins:**

| Plugin | Before | After | Source of truth |
|---|---|---|---|
| `bee` | 4.5.1 | **4.8.2** | vendored, byte-identical to `upstream/main` |
| `nectar` | — | **1.0.0** (NEW) | vendored, byte-identical to `upstream/main` |
| `wasp` | 1.4.4 | 1.4.4 (unchanged) | our own layer |
| *marketplace* | 1.4.4 | **1.5.0** | ours |

**How it was done:** `git merge upstream/main` (upstream was at bee-dev marketplace `1.11.0`). We were 32 commits behind / 12 ahead. Only **4 metadata files conflicted** — `.claude-plugin/marketplace.json`, `CHANGELOG.md`, `README.md`, `.gitignore`. `plugins/bee/**` auto-advanced to 4.8.2 (we'd never touched it), `plugins/nectar/**` was a clean add, `plugins/wasp/**` was untouched.

**Conflict-resolution decisions (important — keep consistent in future syncs):**
- Kept the `wasp-dev` marketplace identity, owner, and the `wasp` plugin entry.
- Took upstream **verbatim** for `plugins/bee/**` and `plugins/nectar/**`.
- **Did NOT** let upstream's `bee-dev` marketplace CHANGELOG bleed into *our* `CHANGELOG.md` — that history lives in `plugins/bee/CHANGELOG.md`. Our `CHANGELOG.md` tracks only wasp-dev marketplace versions.
- `.gitignore` = union of both sides.
- Extended the "don't touch vendored plugins" rule in `CLAUDE.md` to cover `plugins/nectar/` too.

**Verification done:** staged `plugins/bee` and `plugins/nectar` tree hashes were confirmed **git-hash-identical** to `upstream/main` — zero local drift.

**Shipped:**
- Commit `65c3929` — `feat(marketplace): v1.5.0 — sync upstream bee 4.5.1→4.8.2 + vendor new nectar 1.0.0` — **pushed to `origin/main`**.
- Annotated tag **`v1.5.0`** — **pushed**.
- Safety branch **`backup/pre-nectar-sync-main`** at old HEAD `f8147ac` — still exists, not yet deleted. (Open question: user hasn't said whether to delete it.)

---

## 3. Local install state (confirmed via `claude plugin list`)

All three at **user scope** (available in every project, not just this repo), all **enabled**:

- `bee@wasp-dev` — **4.8.2** ✅ enabled
- `nectar@wasp-dev` — **1.0.0** ✅ enabled
- `wasp@wasp-dev` — **1.4.4** ✅ enabled

Install commands already run this session: `claude plugin marketplace update wasp-dev`, `claude plugin update bee@wasp-dev`, `claude plugin install nectar@wasp-dev`. **Install is done — the only remaining gate was the restart**, which the user is now doing.

---

## 4. How nectar works (this is what the user was learning — retain it)

Nectar is a **different paradigm** from bee. Do not describe it as "layered on bee."

- **No init. No `/nectar:*` commands. No per-repo activation.** There is no on-switch and no marker file. Because it's installed at user scope, nectar is loaded in **every** session/repo automatically. The right question is never "is nectar active on this repo?" but "is nectar loaded in this session?" (yes, if installed + fresh session).
- **Triggered by natural language.** Each skill's *description* is its trigger. Claude auto-fires the matching skill when the user's wording matches. The user drives it by stating intent, not typing commands.
- **State = plain files in `docs/work/`**, created on demand — not before. No `.bee/`-style state machine, no registry.
- **Never auto-commits.** Suggests a commit message; user approves.

**Skill → trigger → output:**

| Skill | Fires when user says… | Produces |
|---|---|---|
| shape | *"let's build X"*, *"add Y"*, *"change how Z works"* | `docs/work/<topic>/design.md` (only at feature/project scale; trivial/quick writes nothing). **Hard gate:** no code before scale detected; at feature/project scale, no code before design approved. |
| plan | *"plan it"* | `docs/work/<topic>/plan.md` — tasks in dependency waves, checkboxes = progress |
| build | *"implement it"*, *"start building"* | TDD implementation; ticks plan.md checkboxes |
| review | *"review this"* | multi-lens review + fixes; `review.md` |
| **honey** | **"drop the honey"**, *"build X autonomously, don't stop to ask"* | Runs shape→plan→build→review end-to-end with **exactly one up-front confirmation**, then final report. This is the headline feature. |
| debug | *"there's a bug where…"*, a failing test | root cause + fix + regression test |
| audit | *"audit this codebase"* | `docs/work/audit-YYYY-MM-DD.md`, severity-grouped |
| capture | *"remember to also handle X"* (mid-work) | one line in `docs/work/backlog.md` |
| orient | *"where were we?"*, *"what's next?"*, new session on existing `docs/work/` | reads artifacts+git, proposes ONE next action; creates/edits nothing |

**Three read-only enforcement agents** (discipline via structure, not willpower): `nectar:implementer` (TDD, sees only the plan line + design + code, never the conversation), `nectar:lens` (reviews one lens, read-only tools so "no fixes" is physics), `nectar:validator` (adversarially validates findings from a fresh context). Each dispatching skill has a spelled-out general-purpose fallback if the agent type is unavailable.

**⚠️ Critical caveat to keep repeating to the user:** Because nectar is global/always-on, it's *also* loaded in bee projects. **Do not mix bee and nectar on the same feature** — bee uses `.bee/` + `/bee:*` + state machine; nectar uses `docs/work/` + skills. Pick ONE paradigm per project/task or the two state models tangle.

---

## 5. Repo conventions this session had to respect (from CLAUDE.md)

- **Never modify `plugins/bee/` or `plugins/nectar/`** — both vendored, kept byte-identical for zero-conflict upstream merges. All original work goes in `plugins/wasp/`.
- **Lockstep versions** between `.claude-plugin/marketplace.json` and `plugins/wasp/.claude-plugin/plugin.json` — EXCEPT a pure upstream re-sync may move the marketplace version without bumping wasp (which is exactly what v1.5.0 did: marketplace 1.4.4→1.5.0, wasp stayed 1.4.4).
- **Conventional commits.** **Never push without explicit user request** (this session's push WAS explicitly requested).
- **Push flow** uses the PAT at `.secrets/pat.txt` (gitignored): `TOKEN=$(tr -d '\r\n ' < .secrets/pat.txt)` then push to `https://x-access-token:${TOKEN}@github.com/AncientHodler-Demiurg/wasp-dev.git`.
- **Upstream tracking:** `git fetch upstream && git merge upstream/main` (upstream remote = `github.com/george-popescu/bee-dev`). Conflicts expected only in vendored trees + the 4 metadata files.
- CRLF line endings are normal on Windows; don't fight git autocrlf.

---

## 6. Open threads / what the new session might pick up

1. **Verify nectar loaded** in the fresh session (see §1). This is the immediate next action.
2. **Delete the backup branch?** `backup/pre-nectar-sync-main` (`f8147ac`) still exists — user hasn't decided. Offer to delete once they're confident v1.5.0 is good.
3. Possibly do a **live "drop the honey" test run** on a throwaway repo so the user sees nectar's `docs/work/` artifacts appear — was offered, not yet done.
4. The user is still ramping on the nectar mental model — expect more "how does X work in nectar" questions.

---

## 7. Retainer protocol (for future sessions)

- Retainers live in `docs/` as `retainer-NN.md`, zero-padded, incrementing (`01`, `02`, `03`, …).
- Each retainer documents **one handoff episode**. When starting a new session that continues prior work, **read the highest-numbered retainer first**, then earlier ones as needed for backstory.
- When the user asks to "create a handoff / retainer", write the **next** number. Don't overwrite an existing one.
- Note: `docs/work/` is nectar's territory (§4). Keep retainers directly in `docs/` (not `docs/work/`) so they don't get confused with nectar feature artifacts.
