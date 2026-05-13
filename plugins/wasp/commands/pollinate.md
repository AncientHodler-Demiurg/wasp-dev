---
description: Post-ship publishing pipeline for GitHub repos — works for npm-backed packages (publishes to npmjs.com) AND plain repos (versioning + GitHub Releases only). First run = interactive bootstrap wizard. Subsequent runs = full publish pipeline (push, tag, optional npm publish, GitHub Release, verify, backfill).
argument-hint: "[--reinit] [--dry-run] [--skip-backfill] [--skip-npm]"
---

## Current State (load before proceeding)

Read these files using the Read tool:
- `.bee/STATE.md` — if not found: NOT_INITIALIZED
- `.bee/config.json` — if not found: use `{}`

## Git Status (load before proceeding)

Run these via Bash tool:
- `git status --short` — if fails: NO_GIT
- `git rev-parse --abbrev-ref HEAD` — capture as `$CURRENT_BRANCH`
- `git log -1 --format='%H %s'` — capture as `$HEAD_COMMIT_INFO`

## Instructions

You are running `/wasp:pollinate` -- the post-ship publishing pipeline for npm-backed GitHub repositories. After `/bee:ship` and `/bee:commit` have produced a clean commit on the spec branch, this command takes that commit through the full ceremony: push to origin → tag → trigger CI publish → verify npm registry → create GitHub Release → backfill any missing prior Releases. Follow these steps in order.

This command is **idempotent**: if any step has already happened (tag exists, Release exists, npm version published), it detects that and skips, so the command is safe to re-run after partial failures.

This command **respects the bug-detection-by-design** philosophy: it stops at the first ambiguous condition and asks the user via AskUserQuestion rather than guessing. Pushing to remote, creating tags, and publishing to npm are visible/destructive actions; deliberation matters.

### Step 0: First-Run Bootstrap (or Load Existing Credentials)

Before running the publish pipeline, ensure the project is initialized for pollinate. On **first contact** with a repo, this runs a 4-stage wizard that gathers + verifies everything pollinate needs (GitHub repo URL, PAT, repo secrets, npm package URL). On **subsequent runs**, it detects existing initialization and validates it's still good — fast path, no prompts.

If `--reinit` is in `$ARGUMENTS`, force re-running the wizard even if credentials exist. The previous credentials file is backed up to `.bee/pollinate-credentials/.archive/pollinate-credentials-{ISO8601-timestamp}.md`.

#### Step 0.1: Detect existing initialization

Check `.bee/pollinate-credentials/pollinate-credentials.md`:

- **If file exists AND `--reinit` is NOT passed:**
  1. Parse the YAML-like fields (`Github repository`, `Github owner`, `Github repo`, `npm package`, `npm URL`, `Local PAT path`, `Repo secret RELEASE_TOKEN`, `Repo secret NPMPUSHER`).
  2. Verify each field is still consistent with the project state:
     - `Github repository` matches `git remote -v` origin URL
     - `npm package` matches `{lifecycle.npm_dir}/package.json` `name`
     - `.secrets/PAT.txt` exists and is non-empty
  3. If all checks pass: load values into runtime variables (`$REPO_OWNER`, `$REPO_NAME`, `$PKG_NAME`, `$PKG_VERSION`, `$NPM_URL`, `$LOCAL_PAT`) and **SKIP to Step 1**. Display:
     ```
     ✓ Pollinate already initialized for {$REPO_OWNER}/{$REPO_NAME} (verified {timestamp from file}).
     ```
  4. If any check fails: display "Pollinate credentials drifted: {specific mismatch}. Re-running wizard." and continue to 0.2.

- **If file does NOT exist:** continue to 0.2 (first run).

- **If `--reinit` is passed:** archive the existing file, then continue to 0.2:
  ```bash
  TS=$(date +%Y-%m-%dT%H%M%S)
  mkdir -p .bee/pollinate-credentials/.archive
  mv .bee/pollinate-credentials/pollinate-credentials.md \
     .bee/pollinate-credentials/.archive/pollinate-credentials-${TS}.md
  ```

#### Step 0.2: Stage A — Auto-detect (no user input)

1. **A1 — GitHub repo from git remote.** Run `git remote -v` and parse the `origin` URL. Handle both forms:
   - `https://github.com/{owner}/{repo}.git` (or without `.git`)
   - `git@github.com:{owner}/{repo}.git`
   - URLs with embedded tokens like `https://x-access-token:ghp_xxx@github.com/{owner}/{repo}.git` — strip the auth before parsing.

   Extract `$REPO_OWNER` and `$REPO_NAME`. If origin is not a github.com URL, halt: "Pollinate currently supports github.com origins only. Detected origin: {url}."

2. **A2 — npm package from package.json.** Read `{$CFG.npm_dir}/package.json`. Extract `name`, `version`, `repository`, `publishConfig`. Store as `$PKG_NAME`, `$PKG_VERSION`, `$PKG_REPOSITORY`, `$PKG_PUBLISHCONFIG`.

   If `name` is scoped (`@scope/pkg`), extract `$NPM_SCOPE` (e.g. `@stoachain`).

3. **A3 — Compute npm URL.** `$NPM_URL = "https://www.npmjs.com/package/" + $PKG_NAME`.

4. **A4 — Already handled in 0.1.**

5. **A5 — Pre-existing PAT detection.** Check `.secrets/PAT.txt`:
   - If file exists AND has non-empty content: trim whitespace, store as `$LOCAL_PAT`. Set `$PAT_PRE_VERIFIED = false` for now (will be set true after 0.3 B4 validation).
   - Otherwise: `$LOCAL_PAT = ""`.

6. **A6 — Auto-suggest repo type from package.json state.**

   Determine the proposed `$REPO_TYPE` based on what was found in A2:

   | package.json state | Auto-suggested `$REPO_TYPE` | Confidence |
   |---|---|---|
   | File missing entirely | `plain` | high — clearly not a JS package |
   | `private: true` | `plain` | high — explicitly opted out of publishing |
   | Has `name + version + publishConfig` | `npm-package` | high — fully wired for publishing |
   | Has `name + version` but no `publishConfig` | `npm-package` | medium — could still publish, just defaults |
   | Has `name + version` with `publishConfig.registry` matching `npm.pkg.github.com` | `multi-registry` | medium — explicitly wired for GitHub Packages |

   Store as `$AUTO_REPO_TYPE`. The user confirms or overrides at B0 (next stage). If `$AUTO_REPO_TYPE = "plain"` and no package.json exists, set `$PKG_NAME`, `$PKG_VERSION`, `$NPM_URL`, `$NPM_SCOPE` all to empty strings — Stage B will skip npm-related substeps entirely.

#### Step 0.3: Stage B — Confirmation pass (user reviews + confirms each)

Each substep displays the relevant context, then asks via AskUserQuestion. The flow is **deliberately conversational** — every user-action step gets explicit confirmation.

##### B0: Confirm repo type

Display the auto-detected suggestion + reasoning:

```
Pollinate works for two kinds of repos:

  npm-package      Publishes to npmjs.com on tag (full pipeline: tag → CI →
                   npm publish → GitHub Release → registry verify).

  plain            GitHub Release only — versioning + tagging + Release page.
                   No npm token, no npmjs publish. Perfect for documentation
                   repos, internal tooling, research notes, etc.

  multi-registry   npmjs + GitHub Packages mirror (npm-package flow plus a
                   second publish to npm.pkg.github.com for the GitHub
                   sidebar Packages widget).

Auto-detected based on package.json: {$AUTO_REPO_TYPE}
  Reason: {one of:}
    - "package.json missing — definitely a plain repo"
    - "package.json has `private: true` — opted out of publishing"
    - "package.json has full publish config — looks like npm-package"
    - "package.json has minimal config — could be npm-package, defaulting yes"
    - "publishConfig.registry points to npm.pkg.github.com — multi-registry"
```

AskUserQuestion(
  question: "Confirm repo type. Auto-detected: {$AUTO_REPO_TYPE}.",
  options: [
    "{$AUTO_REPO_TYPE} (Recommended — auto-detected)",
    "npm-package",
    "plain",
    "multi-registry"
  ]
)

Note: the recommended-option label substitutes the actual auto-detected value, but the other 3 options are always present so the user can override.

Store the chosen value as `$REPO_TYPE`. Derive runtime flags:

```
$REPO_TYPE = "npm-package"     → $PUBLISHES_TO_NPM = true,  $GH_PACKAGES = false
$REPO_TYPE = "plain"           → $PUBLISHES_TO_NPM = false, $GH_PACKAGES = false
$REPO_TYPE = "multi-registry"  → $PUBLISHES_TO_NPM = true,  $GH_PACKAGES = true
```

`$PUBLISHES_TO_NPM` gates B7+B8+B9 (npm-token setup) and Steps 4c, 8, 9a/9b/9c/9d.

##### B1: Confirm GitHub repo

Display:
```
Detected GitHub repository:
  Owner: {$REPO_OWNER}
  Repo:  {$REPO_NAME}
  URL:   https://github.com/{$REPO_OWNER}/{$REPO_NAME}
```

AskUserQuestion(
  question: "Use this as the GitHub repository for pollinate?",
  options: ["Yes, correct", "Override (different repo)", "Cancel"]
)

If "Override": ask for full URL via free-text. Re-parse owner/name. Re-loop B1 for confirm.
If "Cancel": display "Bootstrap cancelled. Re-run /wasp:pollinate when ready." and stop.

##### B2: Validate or request a classic GitHub PAT

If `$LOCAL_PAT` is non-empty (A5 found existing token):
  Display: `Found existing PAT at .secrets/PAT.txt. Validating...`
  Skip ahead to B4 directly (skip the create-and-paste prompts).

Otherwise:
  Display:
  ```
  Pollinate needs a GitHub Personal Access Token with these EXACT scopes:
    [x] repo
    [x] workflow
    [x] write:packages

  IMPORTANT: This must be a CLASSIC token (not fine-grained).
    - Fine-grained tokens do not have the write:packages scope.
    - Pollinate will refuse fine-grained tokens (prefix github_pat_).

  Create one at:
    https://github.com/settings/tokens
    → "Generate new token" → "Generate new token (classic)"
    → Tick all 3 scopes: repo, workflow, write:packages
    → Set expiration (90 days minimum recommended)
    → "Generate token"
    → Copy the token immediately (you cannot view it again)
  ```

  AskUserQuestion(
    question: "GitHub PAT created and copied to clipboard?",
    options: ["Yes, ready to paste", "Cancel"]
  )

##### B3: Paste PAT into .secrets/PAT.txt

Skip if B2 jumped ahead.

Pollinate creates the file structure:
```bash
mkdir -p .secrets
[ -f .secrets/PAT.txt ] || touch .secrets/PAT.txt

# Update .gitignore proactively (D1 handles the formal save)
if ! grep -q "^\.secrets/$\|^\.secrets$\|^/\.secrets/$" .gitignore 2>/dev/null; then
  echo ".secrets/" >> .gitignore
  echo "✓ Added .secrets/ to .gitignore"
fi
```

Display:
```
File ready: .secrets/PAT.txt (currently empty)

Paste the token:
  - Open .secrets/PAT.txt in your editor
  - Paste the token (overwrite anything else)
  - Save the file

Pollinate will read it next.
```

AskUserQuestion(
  question: "PAT pasted into .secrets/PAT.txt and saved?",
  options: ["Yes, saved", "Cancel"]
)

After confirmation, read the file: `$LOCAL_PAT = $(cat .secrets/PAT.txt | tr -d '\n\r' | xargs)` (trim whitespace).

If `$LOCAL_PAT` is empty: re-loop B3 with "File still empty. Did you save?".

##### B4: Validate the PAT

Run three validation checks:

1. **Format check.**
   - If `$LOCAL_PAT` starts with `github_pat_`: halt with:
     ```
     ✗ That's a fine-grained token. Pollinate requires a CLASSIC token because
       the write:packages scope only exists on classic tokens.

       Generate a classic token at:
       https://github.com/settings/tokens → Generate new token (classic)
     ```
     Loop back to B2.
   - If `$LOCAL_PAT` does not start with `ghp_`: warn that this isn't a recognized token format and ask user to confirm.

2. **API call.** Call `GET https://api.github.com/user` with `Authorization: Bearer $LOCAL_PAT`. Expect HTTP 200.
   - On 401: token is invalid or revoked. Loop back to B2.
   - On other errors: surface the error and ask user.

3. **Scope check.** Parse the `X-OAuth-Scopes` response header from the same call. Required scopes: `repo`, `workflow`, `write:packages`. All three must be present. Other scopes are fine.
   - On missing scopes: list which are missing and tell the user to either regenerate the token (recommended) or add scopes via the existing token's edit page. Loop back to B2.

On success, display:
```
✓ PAT validates as user: {github-username from /user response}
✓ Scopes confirmed: repo, workflow, write:packages
```

Set `$PAT_PRE_VERIFIED = true`.

##### B5: Add PAT as RELEASE_TOKEN repo secret

Display:
```
Now register the SAME PAT as a repository secret on GitHub.

Open in your browser:
  https://github.com/{$REPO_OWNER}/{$REPO_NAME}/settings/secrets/actions

Click "New repository secret" (or "Update" if RELEASE_TOKEN already exists):
  Name:  RELEASE_TOKEN
  Value: (paste the same PAT you just put in .secrets/PAT.txt)

Click "Add secret".
```

AskUserQuestion(
  question: "RELEASE_TOKEN secret added to the repo?",
  options: ["Yes, added", "Already there from before", "Cancel"]
)

##### B6: Verify RELEASE_TOKEN exists

Call `GET https://api.github.com/repos/{$REPO_OWNER}/{$REPO_NAME}/actions/secrets` with `Authorization: Bearer $LOCAL_PAT`. Parse response — it returns `{ "secrets": [{ "name": "...", ... }, ...] }` (names only, never values).

Confirm `RELEASE_TOKEN` is in the list.

- On found: display `✓ Repo secret RELEASE_TOKEN present.`
- On not found: display `✗ Cannot find RELEASE_TOKEN secret on the repo.` and re-loop B5.
- On 403 from API: token lacks `repo` scope (shouldn't happen since B4 verified). Halt with diagnostic.
- On 404: repo doesn't exist or PAT lacks access. Halt with diagnostic.

##### B7: Add NPMPUSHER repo secret

**Skip B7 + B8 + B9 entirely if `$PUBLISHES_TO_NPM` is false** (plain repo). Display:

```
✓ Plain repo — skipping npm-token setup (no NPMPUSHER needed).
✓ No npm package URL to confirm.
```

Proceed to Stage C.

Otherwise (npm-package or multi-registry), continue:

Display:
```
Pollinate also needs an npm token to publish (separate from GitHub).

1. Generate an npm Granular Access Token:
   https://www.npmjs.com/settings/{your-npm-username}/tokens/granular-access-tokens/new

   Required:
     - Token name:           pollinate-{$REPO_NAME}
     - Expiration:           1 year (or your preference)
     - Permissions:          "Read and write"
     - Packages and scopes:
         → Selected packages and scopes
         → Add "{$NPM_SCOPE}" (e.g. @stoachain)
     - IP/CIDR restriction:  leave empty (GitHub Actions uses dynamic IPs)

   Click "Generate Token" and copy it (cannot be viewed again).

2. Add it as a repository secret on GitHub:
   https://github.com/{$REPO_OWNER}/{$REPO_NAME}/settings/secrets/actions

   Click "New repository secret":
     Name:  NPMPUSHER
     Value: (paste the npm token)

   Click "Add secret".
```

AskUserQuestion(
  question: "NPMPUSHER secret added to the repo?",
  options: ["Yes, added", "Already there from before", "Cancel"]
)

##### B8: Verify NPMPUSHER exists

Same API call as B6. Confirm `NPMPUSHER` is in the secrets list. Same handling.

Note for the user: pollinate cannot verify the npm token's value (only its presence on the repo). The first publish attempt will validate publish-access via the npm registry. If npm returns 403, the token's package-scope permissions are wrong — pollinate detects this in the workflow log and explains the fix.

##### B9: Confirm npm package URL

Display:
```
npm package detected from package.json:
  Name:            {$PKG_NAME}
  Current version: {$PKG_VERSION}
  npm scope:       {$NPM_SCOPE}
  npm URL:         {$NPM_URL}
```

AskUserQuestion(
  question: "Use this as the npm package?",
  options: ["Yes, correct", "Override (different package)", "Cancel"]
)

#### Step 0.4: Stage C — Repo health checks (auto, with warnings)

Each runs an API call. Failures are **non-blocking** (pollinate continues to D), but the user sees a warning + suggested fix path. Exception: C1 IS blocking (no Actions, no auto-publish — pipeline cannot proceed).

##### C1: Actions enabled (BLOCKING)

Call `GET /repos/{$REPO_OWNER}/{$REPO_NAME}/actions/permissions` with the local PAT.

Check `enabled: true`. If false:
```
✗ GitHub Actions are DISABLED on this repo. The publish workflow will never
  trigger on tag push.

  Enable at:
  https://github.com/{$REPO_OWNER}/{$REPO_NAME}/settings/actions

  → "Allow all actions and reusable workflows" (or your org's policy)

After enabling, re-run /wasp:pollinate.
```

Halt — this is blocking.

If `enabled: true`: display `✓ GitHub Actions enabled.`

##### C2: Branch protection on main

Call `GET /repos/{$REPO_OWNER}/{$REPO_NAME}/branches/main/protection` with the local PAT.

- **HTTP 404**: no branch protection on main. Pollinate can fast-forward push freely. Display `✓ No branch protection on main — fast-forward push enabled.`
- **HTTP 200**: parse response.
  - If `required_pull_request_reviews` is null/absent: ✓ no PR requirement.
  - If present: warn:
    ```
    ⚠ Branch protection on main requires {N} PR review(s).
      Pollinate will use the PR-required publish flow (slower, but compatible).

      Setting lifecycle.branch_protection = "pr-required" in .bee/config.json.

      Alternatively, to allow direct fast-forward pushes:
        - Add yourself as a bypass actor at:
          https://github.com/{owner}/{repo}/settings/branches
        - Then set lifecycle.branch_protection = "fast-forward".
    ```
    Update the lifecycle config in `.bee/config.json` in-place: set `branch_protection: "pr-required"`.
- **HTTP 403**: PAT lacks repo admin scope; protection rules are not visible. Display warning, default to `branch_protection: "fast-forward"` and let the actual push attempt determine if PR-required is needed.

##### C3: Workflow file exists

Check `.github/workflows/{$CFG.ci_workflow_filename}` exists in the working tree.

- If exists: display `✓ Publish workflow present at .github/workflows/{filename}.`
- If missing:
  ```
  ⚠ No CI publish workflow at .github/workflows/{$CFG.ci_workflow_filename}.

  Without it, npm publish + GitHub Release auto-creation won't trigger on
  tag push. Pollinate would need to publish locally instead (slower, less
  audit trail).

  Reference template: https://github.com/StoaChain/DALOS_Crypto/blob/main/.github/workflows/ts-publish.yml
  ```
  AskUserQuestion(
    question: "Continue without CI workflow? Pollinate will fall back to local publish.",
    options: ["Yes, continue (local publish)", "Halt — I'll add the workflow first", "Custom"]
  )

  If "Halt": stop, user adds workflow, re-runs `/wasp:pollinate`.
  If "Yes, continue": set `lifecycle.ci_workflow_skip = true` in config and proceed.

##### C4: CHANGELOG.md exists

Check `{$CFG.changelog_path}` exists.

- If exists: display `✓ CHANGELOG present.`
- If missing:
  ```
  ⚠ No CHANGELOG.md at {$CFG.changelog_path}.

  Pollinate uses CHANGELOG sections for tag annotations + Release bodies.
  Without it, tag bodies fall back to git-log output (less curated).
  ```
  AskUserQuestion(
    question: "Scaffold an empty Keep-a-Changelog template?",
    options: ["Yes, scaffold", "Skip — I'll add manually", "Custom"]
  )

##### C5: package.json fields

Skip this substep entirely if `$REPO_TYPE = "plain"` AND no package.json exists at `{$CFG.npm_dir}/package.json`. (A pure plain repo with no JS-package metadata at all.)

Otherwise, the field-set checked depends on `$REPO_TYPE`:

**If `$REPO_TYPE` is `npm-package` or `multi-registry`:**

- `name` — must be present (already used in A2).
- `version` — must be present.
- `repository.url` — required for the npm-↔-GitHub link. Should match `https://github.com/{$REPO_OWNER}/{$REPO_NAME}` (with or without `.git`).
- `publishConfig.access: "public"` — required for scoped public packages (`@scope/name`).

**If `$REPO_TYPE` is `plain` AND a package.json exists** (e.g. a frontend app with package.json but no publish intent):

- `version` — should be present and match `$PKG_VERSION`.
- `private: true` — should be set to make the no-publish intent explicit (npm refuses to publish a `private: true` package, providing a safety net even if the user accidentally runs `npm publish`).
- `repository.url` — still useful for the GitHub link, but optional.

For each missing/wrong field:
```
⚠ package.json {field}: {issue}
    Expected: {expected}
    Actual:   {actual or "(missing)"}
```

If any issues found, AskUserQuestion(
  question: "Auto-fix package.json via Edit?",
  options: ["Yes, apply all fixes", "Show fixes per-field", "Skip", "Custom"]
)

If "Yes": apply all Edits. If "Show per-field": ask one-by-one.

#### Step 0.5: Stage D — Save state

##### D1: .gitignore

Already added `.secrets/` proactively in B3. Verify here:
- `.gitignore` exists in working tree.
- Contains `.secrets/` (or `.secrets`) line.
- If `.bee/` is not gitignored: warn that the lifecycle config (`config.json`) and `pollinate-credentials/` won't be ignored. Add `.bee/` to `.gitignore` if user agrees:
  ```
  ⚠ .bee/ is not in .gitignore. Pollinate stores credentials there.

  AskUserQuestion(
    question: "Add .bee/ to .gitignore?",
    options: ["Yes, add", "No, I have a custom rule", "Custom"]
  )
  ```

##### D2: Lifecycle config in .bee/config.json

Read `.bee/config.json`. If `lifecycle` block is missing, build it based on `$REPO_TYPE`.

**Plain-repo only — extra question first:** if `$REPO_TYPE = "plain"`, ask where the version comes from:

```
AskUserQuestion(
  question: "Where does the version number live for this plain repo?",
  options: [
    "VERSION file at repo root (Recommended for plain repos)",
    "Latest git tag (e.g. tag v1.2.3 → version 1.2.3)",
    "Specified manually each time I run /wasp:pollinate",
    "Custom path / Custom"
  ]
)
```

Maps to:

| Choice | `lifecycle.version_source` | `lifecycle.version_file_path` |
|---|---|---|
| VERSION file | `"version_file"` | `"VERSION"` |
| Latest git tag | `"git_tags"` | (omitted) |
| Manual | `"manual"` | (omitted) |
| Custom path | `"version_file"` | (user-provided path) |

If "VERSION file" and the file does not yet exist, offer to create it with an initial value (default `0.1.0` or ask).

**For npm-package and multi-registry**, `version_source` is implicitly `"package_json"` and the path is implied by `npm_dir`.

**Build the proposed lifecycle config:**

```json
{
  "lifecycle": {
    "repo_type": "{$REPO_TYPE}",
    "publishes_to_npm": {true if npm-package or multi-registry, else false},
    "version_source": "{package_json|version_file|git_tags|manual}",

    // Only present when version_source = "version_file":
    "version_file_path": "{path}",

    // Only present when publishes_to_npm = true:
    "npm_dir": "{$CFG.npm_dir}",
    "npm_registry": "https://registry.npmjs.org",
    "use_provenance": true,
    "github_packages_mirror": {true if multi-registry, else false},

    // Always present:
    "tag_pattern": "{user-chosen, default v{version}}",
    "release_title_pattern": "{user-chosen, default same as tag_pattern}",
    "tag_message_source": "changelog",
    "changelog_path": "{$CFG.changelog_path}",
    "backfill_on_publish": true,

    // CI knobs — defaults shift by repo_type:
    "ci_workflow_filename": "{filename}",
    "ci_workflow_skip": {boolean from C3 — defaults true for plain repos with no .github/workflows/* file},
    "wait_for_ci_seconds": 600,

    "branch_protection": "{from C2}"
  }
}
```

The npm-only fields are simply OMITTED for plain repos — pollinate's Stage A reader uses `$REPO_TYPE` as the gate, so missing fields default to safe values.

For `tag_pattern` specifically, ask:
```
AskUserQuestion(
  question: "Tag pattern? Determines the git tag name format.",
  options: [
    "v{version}  (Recommended for single-stack repos)",
    "ts-v{version}  (For dual-stack repos with TS port — disambiguates against Go-side v* tags)",
    "{prefix}-v{version}  (Custom prefix)",
    "Custom"
  ]
)
```

For `release_title_pattern`, ask similarly. If `tag_pattern` has a prefix, suggest dropping it for the title (e.g. tag `ts-v3.1.0` displays as Release `v3.1.0`).

Display the proposed config in full, then:
```
AskUserQuestion(
  question: "Save this lifecycle config to .bee/config.json?",
  options: ["Yes, save", "Edit a field first", "Cancel"]
)
```

##### D3: Write pollinate-credentials.md

Create `.bee/pollinate-credentials/` directory if missing. Write `pollinate-credentials.md`:

```markdown
# Pollinate credentials — initialized {ISO 8601 timestamp}

**Github repository:** https://github.com/{$REPO_OWNER}/{$REPO_NAME}
**Github owner:** {$REPO_OWNER}
**Github repo:** {$REPO_NAME}
**npm package:** {$PKG_NAME}
**npm scope:** {$NPM_SCOPE}
**npm URL:** {$NPM_URL}
**Local PAT path:** .secrets/PAT.txt
**PAT scopes verified:** repo, workflow, write:packages
**PAT verified at:** {ISO timestamp from B4}
**PAT user:** {github username from B4}
**Repo secret RELEASE_TOKEN:** present (verified {timestamp from B6})
**Repo secret NPMPUSHER:** present (verified {timestamp from B8})

## Health-check results

- C1 Actions enabled: ✓
- C2 Branch protection: {✓ no protection | ⚠ PR-required | ⚠ unable to verify}
- C3 Workflow file `.github/workflows/{filename}`: {✓ present | ⚠ missing — local publish}
- C4 CHANGELOG: {✓ present | ⚠ missing — git-log fallback}
- C5 package.json fields: {✓ all present | ⚠ {N} fixed during bootstrap}

## Lifecycle config

Saved to `.bee/config.json` lifecycle block. See command file for the full schema.

---

**Reset:** delete `.bee/pollinate-credentials/` to force the wizard to re-run.
**Re-init:** run `/wasp:pollinate --reinit` to re-run the wizard without deleting (existing file is archived to `.bee/pollinate-credentials/.archive/`).
```

##### D4: Final summary

Display:
```
🐝 Pollinate initialized for {$REPO_OWNER}/{$REPO_NAME}

Saved files:
  ✓ .secrets/PAT.txt (gitignored)
  ✓ .gitignore updated (.secrets/ {+ .bee/ if added})
  ✓ .bee/config.json lifecycle block
  ✓ .bee/pollinate-credentials/pollinate-credentials.md

Verified:
  ✓ GitHub PAT (user: {github-username}, scopes: repo+workflow+write:packages)
  ✓ Repo secret RELEASE_TOKEN
  ✓ Repo secret NPMPUSHER
  {✓|⚠} Health checks (see pollinate-credentials.md)

Next: continuing to publish pipeline...
```

Then continue to Step 1 (publish pipeline starts).

---

### Step 1: Validation Guards

Check these guards in order. Stop immediately if any fails:

1. **NOT_INITIALIZED guard:** If the dynamic context above contains "NOT_INITIALIZED" (meaning `.bee/STATE.md` does not exist), tell the user:
   "BeeDev is not initialized. Run `/bee:init` first."
   Do NOT proceed.

2. **NO_GIT guard:** If git status check failed, tell the user:
   "No git repository detected. `/wasp:pollinate` operates on git remotes."
   Do NOT proceed.

3. **NO_LIFECYCLE_CONFIG guard:** Read `.bee/config.json`. If `lifecycle` block is missing OR `lifecycle.publishes_to_npm` is not `true`, tell the user:
   "This project is not configured for npm publishing. Add a `lifecycle` block to `.bee/config.json`:

   ```json
   \"lifecycle\": {
     \"publishes_to_npm\": true,
     \"npm_dir\": \".\",
     \"tag_pattern\": \"v{version}\",
     \"release_title_pattern\": \"v{version}\"
   }
   ```

   See `commands/pollinate.md` for the full lifecycle config schema."
   Do NOT proceed.

4. **CLEAN_TREE guard:** Parse the git status output. If any uncommitted changes exist, tell the user:
   "Uncommitted changes detected:
   {git status output}

   `/wasp:pollinate` requires a clean tree. Either commit (via `/bee:commit`) or stash, then re-invoke."
   Do NOT proceed.

5. **COMMITTED_PHASES guard (soft):** Read STATE.md Phases table. If the spec is still active (Status not `NO_SPEC`) AND any phase has a Status that is NOT `COMMITTED`, warn:
   "Some phases are not COMMITTED:
   - Phase {N}: {name} (Status: {status})

   Pollinating publishes the current commit regardless. Confirm:"

   AskUserQuestion(
     question: "Some phases not committed. Pollinate anyway?",
     options: ["Pollinate", "Cancel", "Custom"]
   )
   If "Cancel", display "Pollinate cancelled." and stop.
   This is a SOFT guard — sometimes you want to publish a hotfix without going through the full spec lifecycle.

### Step 2: Load Lifecycle Config

Read `.bee/config.json` `lifecycle` block. Resolve effective config with these defaults:

| Key | Default | Purpose |
|---|---|---|
| `publishes_to_npm` | `true` (already gated above) | Master switch for npm steps |
| `npm_dir` | `"."` | Directory containing `package.json` |
| `npm_registry` | `"https://registry.npmjs.org"` | Target npm registry |
| `tag_pattern` | `"v{version}"` | Git tag format. `{version}` is replaced with `package.json` version |
| `release_title_pattern` | `"v{version}"` | GitHub Release display title. Can differ from `tag_pattern` (e.g. drop a `ts-` prefix) |
| `tag_message_source` | `"changelog"` | One of `"changelog"` (extract section from CHANGELOG.md), `"manual"` (prompt user), `"both"` |
| `changelog_path` | `"CHANGELOG.md"` | Path to CHANGELOG, relative to repo root |
| `github_packages_mirror` | `false` | If true, also publish to `https://npm.pkg.github.com` |
| `use_provenance` | `true` | Add `--provenance` to `npm publish` (npm 9.5+, GitHub Actions only) |
| `backfill_on_publish` | `true` | Compute missing prior Releases and create them via REST |
| `wait_for_ci_seconds` | `600` | Max wait for CI publish workflow |
| `ci_workflow_filename` | `"publish.yml"` | Workflow file name (relative to `.github/workflows/`) |
| `ci_workflow_skip` | `false` | If true, skip CI monitoring (for repos without CI auto-publish) |
| `branch_protection` | `"fast-forward"` | One of `"fast-forward"` (push directly), `"pr-required"` (open PR and wait for merge) |

Store these as `$CFG.<key>`. Display the resolved config to the user:

```
Pollinate config (effective):
  npm_dir:               {npm_dir}
  tag_pattern:           {tag_pattern}
  release_title:         {release_title_pattern}
  github_packages:       {github_packages_mirror}
  provenance:            {use_provenance}
  backfill:              {backfill_on_publish}
  CI workflow:           {ci_workflow_filename} (skip={ci_workflow_skip})
  branch_protection:     {branch_protection}
```

### Step 3: Read Package + Compute Version

Version source dispatch — driven by `$CFG.lifecycle.version_source`:

| `version_source` | How `$PKG_VERSION` is computed |
|---|---|
| `"package_json"` (default for npm-package / multi-registry) | Read `{npm_dir}/package.json` `version` field |
| `"version_file"` | Read `cat {version_file_path} \| tr -d '\n\r' \| xargs` |
| `"git_tags"` | Run `git describe --tags --abbrev=0`, then strip the `tag_pattern` prefix |
| `"manual"` | AskUserQuestion: "Version for this run?" — free-text, must match `^[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?$` (semver) |

1. Run the dispatch above to populate `$PKG_VERSION`.

2. **For npm-package / multi-registry**: also read `name` from `{npm_dir}/package.json`. Store as `$PKG_NAME`. Halt if missing.

   **For plain**: `$PKG_NAME` is empty (no npm package name). Computing it from `$REPO_NAME` is a fallback for display purposes only.

3. **Validation**: if `$PKG_VERSION` is empty or non-semver, halt with the source-specific diagnostic:
   - `version_source: "package_json"` → "package.json not found at {path} or `version` field missing/invalid."
   - `version_source: "version_file"` → "VERSION file not found at {path} or empty/invalid."
   - `version_source: "git_tags"` → "No git tags found matching `tag_pattern`. Initial release? Pollinate cannot infer the first version — set `version_source: \"manual\"` for the first run."
   - `version_source: "manual"` → "Version input failed semver validation: {input}."

4. Compute the tag name by substituting `{version}` in `$CFG.tag_pattern`:
   - `tag_pattern: "v{version}"` + `version: "3.1.0"` → `$TAG_NAME = "v3.1.0"`
   - `tag_pattern: "ts-v{version}"` + `version: "3.1.0"` → `$TAG_NAME = "ts-v3.1.0"`

5. Compute the Release title similarly from `$CFG.release_title_pattern`. Store as `$RELEASE_TITLE`.

6. Display:

   ```
   Repo type:      {$REPO_TYPE}
   Version source: {version_source} → {$PKG_VERSION}
   {Package:        {$PKG_NAME}     ← only if not plain}
   Git tag:        {$TAG_NAME}
   Release title:  {$RELEASE_TITLE}
   HEAD commit:    {first 8 chars of HEAD SHA} {commit subject}
   ```

### Step 3.5: Spec Target Reconciliation

After computing `$PKG_VERSION`, cross-check it against the **active spec's** declared version target. This catches forgotten version bumps before they ship.

This step runs only when STATE.md has an active spec (Status NOT `NO_SPEC`). If `NO_SPEC`, skip — there's no spec to reconcile against (e.g., hotfix outside the Bee lifecycle).

#### 3.5a. Locate active spec's requirements.md

1. Read STATE.md `Current Spec Path`. Store as `$SPEC_PATH`.
2. Read `$SPEC_PATH/requirements.md`. If missing, skip 3.5 silently.

#### 3.5b. Extract version target citation

Use Grep on `$SPEC_PATH/requirements.md` for the version-target citation. Patterns to match (case-insensitive, in priority order):

1. `\*\*Version target:\*\*\s+v?(\d+\.\d+\.\d+)` (Bee bundle template default)
2. `\*\*Target version:\*\*\s+v?(\d+\.\d+\.\d+)` (alternative wording)
3. `Version target:\s+v?(\d+\.\d+\.\d+)` (no bold)
4. `Target version:\s+v?(\d+\.\d+\.\d+)` (no bold)

Capture the FIRST matched semver as `$SPEC_TARGET_VERSION`. If none of the patterns match, skip 3.5 silently — the spec was created without a documented version target (e.g., a manually-written spec).

#### 3.5c. Compare and reconcile

Compare `$PKG_VERSION` (from Step 3) against `$SPEC_TARGET_VERSION`:

| Comparison | Action |
|---|---|
| `equal` | ✓ Display `Spec target matches package.json: {$SPEC_TARGET_VERSION}.` Continue to Step 4. |
| `target > current` | The user forgot to bump. Offer reconciliation (3.5d). |
| `target < current` | ⚠ ⚠ Anomaly — package.json is ahead of the spec target. May indicate the spec was already partially shipped, or the user pre-bumped manually. Display a warning + AskUserQuestion before proceeding. |
| Versions differ in non-comparable ways (pre-release tags, etc.) | Display both, ask user to confirm which to use. |

Display the comparison:

```
Spec target reconciliation:
  package.json:  {$PKG_VERSION}
  Spec target:   {$SPEC_TARGET_VERSION}  (from {$SPEC_PATH}/requirements.md)
  Comparison:    {equal | target-ahead | target-behind | non-comparable}
```

#### 3.5d. Apply the bump (only if `target > current`)

```
AskUserQuestion(
  question: "Apply version bump {$PKG_VERSION} → {$SPEC_TARGET_VERSION}?",
  options: [
    "Yes — add a new chore(version) commit (Recommended)",
    "Yes — amend the most recent commit (single-commit-per-release)",
    "Override version manually",
    "Skip the bump (use current package.json version)"
  ]
)
```

##### Option 1: chore(version) commit (Recommended)

Edit `$CFG.npm_dir/package.json` (or `$CFG.version_file_path` if `version_source = "version_file"`) to set `version` to `$SPEC_TARGET_VERSION`:

```bash
# For package.json:
node -e "const p=require('./{$CFG.npm_dir}/package.json'); p.version='{$SPEC_TARGET_VERSION}'; require('fs').writeFileSync('./{$CFG.npm_dir}/package.json', JSON.stringify(p, null, 2)+'\n');"
```

(For `version_file`: `echo "{$SPEC_TARGET_VERSION}" > {$CFG.version_file_path}`.)

If `package-lock.json` exists in the same directory, also bump it:

```bash
cd {$CFG.npm_dir} && npm install --package-lock-only --silent  # regenerates lock file with new version
```

Stage + commit:

```bash
git -C {$CFG.npm_dir} add package.json package-lock.json 2>/dev/null
git commit -m "chore(version): bump to {$SPEC_TARGET_VERSION}

Aligned package.json with spec target documented in
{$SPEC_PATH}/requirements.md.

Co-Authored-By: Claude (via /wasp:pollinate Step 3.5)
"
```

Use the same git author env vars as the most recent commit (so the new commit's author/committer matches). Read these via:
- `GIT_AUTHOR_NAME = git log -1 --format='%an'`
- `GIT_AUTHOR_EMAIL = git log -1 --format='%ae'`
- (and `GIT_COMMITTER_NAME` / `GIT_COMMITTER_EMAIL` similarly)

After commit, set `$PKG_VERSION = $SPEC_TARGET_VERSION` and recompute `$TAG_NAME` and `$RELEASE_TITLE` from the new value. Re-display the Step 3 summary block with the updated values.

##### Option 2: amend the most recent commit

```
⚠ Amending will rewrite the most recent commit. Only safe if the commit
  has not been pushed to a shared remote yet.
```

Verify the most recent commit hasn't been pushed:
```bash
LOCAL_HEAD=$(git rev-parse HEAD)
REMOTE_HEAD=$(git rev-parse @{upstream} 2>/dev/null || echo "")
if [ "$LOCAL_HEAD" = "$REMOTE_HEAD" ]; then
  echo "⚠ Most recent commit is already pushed to upstream. Amend would require force-push."
fi
```

If already pushed, refuse with a warning + suggest Option 1 instead. (Force-push to main is in the safety-rule prohibitions; refuse hard.)

If safe to amend, edit package.json + lock file, then:

```bash
git -C {$CFG.npm_dir} add package.json package-lock.json 2>/dev/null
git commit --amend --no-edit
```

`--no-edit` preserves the original commit message + author. The version-bump becomes part of the spec's commit, single commit per release.

##### Option 3: Override manually

```
AskUserQuestion(
  question: "What version to use? (semver only)",
  type: free-text
)
```

Validate the input is semver. Use as `$PKG_VERSION` going forward (no commit changes — user takes responsibility).

##### Option 4: Skip

Keep `$PKG_VERSION` as the current package.json value. Note in `$WARNINGS` for the final report: "Spec target {$SPEC_TARGET_VERSION} not applied — published as {$PKG_VERSION}."

#### 3.5e. Post-reconciliation summary

After the chosen action, display the final state:

```
✓ Spec target reconciliation complete:
  Final $PKG_VERSION:  {value}
  Action taken:         {chore-commit | amended | manual-override | skipped}
  {if commit added: Commit added: {short SHA} chore(version): bump to X.Y.Z}
```

### Step 4: Pre-publish Documentation Gates

Check that the documentation surfaces are consistent with the version about to be published. For each gate, if it fails, surface the issue and ask the user how to handle.

#### 4a. CHANGELOG entry exists

Use Grep to look for `^## \[{$PKG_VERSION}\]` in `$CFG.changelog_path`. If no match:

AskUserQuestion(
  question: "No CHANGELOG.md entry for {$PKG_VERSION}. Halt and write one first?",
  options: ["Halt — I'll add the entry", "Pollinate without CHANGELOG entry", "Custom"]
)

If "Halt", display the path and stop. If "Pollinate without", note in `$WARNINGS` and continue (the tag annotation will be a generic auto-generated body).

#### 4b. Version match: package.json ↔ root README badge (if applicable)

If a root `README.md` exists, grep for version badges (typical patterns: `Version-{x.y.z}`, `version: {x.y.z}`). Report any badge that is NOT at `$PKG_VERSION`:

```
Version-mismatch warning:
  README.md:5  badge shows v{stale_version}, package.json is v{$PKG_VERSION}
```

AskUserQuestion(
  question: "Version badges out of date. Update before pollinating?",
  options: ["Update via Edit then continue", "Pollinate as-is", "Cancel", "Custom"]
)

If "Update via Edit", offer specific Edits to the user, apply them, re-amend the commit (if it was the most recent and the tree still clean), and re-verify before proceeding. NEVER force-push to update — instead, recommend an additional commit.

If "Pollinate as-is", note in `$WARNINGS` and continue.

#### 4c. npm-tarball README exists (if `npm_dir` ≠ `.`)

**Skip this gate entirely if `$PUBLISHES_TO_NPM` is false** (plain repo). For plain repos, `npm_dir` is irrelevant — there's no npm tarball.

Otherwise:

If `$CFG.npm_dir` is not `.`, check `{npm_dir}/README.md` exists. If missing, warn:

```
{$CFG.npm_dir}/README.md is missing. The published npm package will have no README.
Consumers landing on npmjs.com will see a default placeholder.
```

AskUserQuestion(
  question: "Pollinate without npm-tarball README?",
  options: ["Pollinate as-is (Not recommended)", "Halt — I'll add a README", "Custom"]
)

#### 4d. Display gate summary

If all gates pass: `"Documentation gates: PASS"`.
Otherwise: list each warning with the user's resolution decision.

### Step 5: Tag Annotation Body

Assemble the annotated-tag body (which becomes the GitHub Release body).

Per `$CFG.tag_message_source`:

#### `"changelog"` (default)

Extract the CHANGELOG section for `$PKG_VERSION` using sed:

```bash
sed -n "/^## \[${PKG_VERSION}\]/,/^## \[/{/^## \[/!p; /^## \[${PKG_VERSION}\]/p}" {changelog_path} | sed '$d'
```

Save the section to `/tmp/pollinate-{$PKG_VERSION}-body.md`. If the section is empty (CHANGELOG entry was just `## [{version}]` with no body), fall through to `"manual"`.

#### `"manual"`

AskUserQuestion(
  question: "Tag annotation body? (Becomes the GitHub Release body)",
  options: ["Auto-generate from package.json + git log", "Open editor — let me write it", "Custom"]
)

If "Auto-generate", write a minimal body:

```
{$PKG_NAME}@{$PKG_VERSION}

Commits since previous tag:
{git log --oneline {previous_tag}..HEAD}
```

#### `"both"`

CHANGELOG section + a "## Commits since previous tag" appendix.

#### Display the body before tagging:

```
Tag annotation body ({lines} lines, {bytes} bytes):
{first 20 lines, truncated with "..." if longer}

[Full body saved at /tmp/pollinate-{$PKG_VERSION}-body.md]
```

AskUserQuestion(
  question: "Use this annotation body?",
  options: ["Yes, proceed", "Edit it", "Cancel", "Custom"]
)

If "Edit it", offer to open the file via the Edit tool and re-display.
If "Cancel", display "Pollinate cancelled." and stop.

### Step 6: Push to origin/main

Per `$CFG.branch_protection`:

#### `"fast-forward"` (default)

1. Run `git fetch origin` to refresh remote refs.

2. Determine if the current branch can fast-forward to `origin/main`:
   - If `$CURRENT_BRANCH` is `main`: simply `git push origin main`.
   - If `$CURRENT_BRANCH` is a feature branch (e.g. `claude/foo-a04cb2`): check whether HEAD is a fast-forward of `origin/main` via `git merge-base origin/main HEAD == origin/main` (i.e. main has not advanced past the branch's base). If yes, push HEAD to main: `git push origin {HEAD_SHA}:refs/heads/main`. If no, the branch has diverged — fall back to `"pr-required"` flow.

3. Also push the feature branch itself (so the audit trail shows both): `git push origin {$CURRENT_BRANCH}`.

4. On push failure (e.g. branch protection rejects, force-push not allowed), surface the error and switch to `"pr-required"` flow.

#### `"pr-required"`

1. Push the feature branch: `git push origin {$CURRENT_BRANCH}`.

2. Display the PR URL hint: `https://github.com/{repo}/pull/new/{$CURRENT_BRANCH}`.

3. AskUserQuestion(
     question: "Branch pushed. Create PR + merge before continuing?",
     options: ["I'll merge then return", "Continue without merging (tag from feature branch)", "Cancel", "Custom"]
   )

   If "I'll merge then return": halt and tell the user to invoke `/wasp:pollinate` again after the PR is merged. The command is idempotent — safe to re-run.

   If "Continue without merging": warn that the GitHub Release will point at a non-main commit; if `$CFG.tag_pattern` is `"v{version}"` (no prefix), this can be confusing. Get explicit confirm.

#### Display result

```
Pushed:
  feature branch: {$CURRENT_BRANCH} → origin/{$CURRENT_BRANCH}
  main:           {HEAD_SHA[:8]} → origin/main (fast-forward)
```

### Step 7: Create + Push Annotated Tag

1. Check tag idempotency: `git rev-parse {$TAG_NAME}` (locally) and `git ls-remote --tags origin {$TAG_NAME}`.
   - If tag exists locally AND points at `$HEAD_COMMIT`: skip creation, proceed to push.
   - If tag exists locally AND points at a DIFFERENT commit: halt with "Tag {$TAG_NAME} already exists at {old_commit}, but HEAD is {new_commit}. Resolve manually."
   - If tag exists on origin BUT not locally: fetch it (`git fetch origin tag {$TAG_NAME}`) and re-check.
   - If tag does not exist anywhere: create it now.

2. Create the annotated tag, sourcing the body from `/tmp/pollinate-{$PKG_VERSION}-body.md`:

   ```bash
   git tag -a {$TAG_NAME} {$HEAD_COMMIT} -F /tmp/pollinate-{$PKG_VERSION}-body.md
   ```

   Use the same git author env vars as the most recent commit (so the tagger matches the committer). Read these via:
   - `GIT_AUTHOR_NAME = git log -1 --format='%an'`
   - `GIT_AUTHOR_EMAIL = git log -1 --format='%ae'`
   - `GIT_COMMITTER_NAME` and `GIT_COMMITTER_EMAIL` similarly

3. Push the tag: `git push origin {$TAG_NAME}`.

4. Display:

   ```
   Tag created and pushed: {$TAG_NAME} → {$HEAD_COMMIT[:8]}
   ```

### Step 8: Wait for CI Publish Workflow

Skip this step if `$CFG.ci_workflow_skip` is `true` — proceed directly to Step 9.

**Common case for plain repos:** `ci_workflow_skip` defaults to `true`. Pollinate skips this entire step and proceeds to Step 9 (which for plain repos creates only the GitHub Release). If the plain repo does have a CI workflow that runs on tag (e.g. for tests, docs build), it can still be monitored by setting `ci_workflow_skip: false` and pointing `ci_workflow_filename` at the right file.

The tag push triggers `.github/workflows/{$CFG.ci_workflow_filename}`. Monitor it:

1. Poll the GitHub API for workflow runs filtered by tag name. Identify the run triggered by this push (matches `head_sha = $HEAD_COMMIT` AND `head_branch = $TAG_NAME`).

2. Poll status every 15 seconds, up to `$CFG.wait_for_ci_seconds`. Display progress:

   ```
   CI publish workflow: in_progress (15s elapsed, max {$CFG.wait_for_ci_seconds}s)
   ```

3. On completion, parse the per-step status:
   - If `conclusion == "success"`: continue to Step 9 with celebration.
   - If `conclusion == "failure"`: fetch the failed step name and the last 100 lines of its log via the GitHub API. Classify the failure:

#### Failure classification

| Failed step | Likely cause | Recovery |
|---|---|---|
| `Lint` / `Typecheck` / `Test` / `Build` | Real bug in code | Halt. Display log excerpt. User must fix and re-tag (likely needs a new patch version). |
| `Verify {package_name} version matches tag` | Tag/package mismatch | Halt. Reconcile and re-tag. |
| `Verify {NPMPUSHER|GH_TOKEN} secret is configured` | Repo missing secret | Halt with link to repo settings. |
| `Publish to npmjs.org` | npm 2FA / scope permission / network | Halt. Display log. May need npm token rotation. |
| `Create GitHub Release for the pushed tag` | Known gh CLI flag bug (see below) | **Auto-fallback to REST API** (see Step 9c). |
| `Backfill GitHub Releases for prior tags (idempotent)` | Same gh CLI flag bug, or stale backfill list | Continue (npm publish already succeeded); flag for follow-up. |
| Anything else | Unknown | Halt. Display log. Ask user. |

#### Known gh CLI flag bug (auto-handled)

The flag combination `gh release create --notes-from-tag --repo X` stopped being supported on GitHub-hosted runners around 2026-04-30 (gh CLI image update). When this exact failure mode is detected ("using `--notes-from-tag` with `--repo` is not supported"), pollinate continues to Step 9 — the npm publish has already succeeded by this point in the workflow, and Step 9c will create the GitHub Release manually via REST API.

### Step 9: Verify + Create Artifacts

#### 9a. Verify npm registry

**Skip 9a entirely if `$PUBLISHES_TO_NPM` is false** (plain repo). Plain repos have no npm artifact to verify — skip directly to 9b (GitHub Release verification).

Otherwise:

GET `{$CFG.npm_registry}/{$PKG_NAME}/{$PKG_VERSION}`. Expect HTTP 200 with the package metadata.

If 404, the npm publish never landed — halt with diagnostic.

If 200 but `version` mismatches: halt with diagnostic (registry caching issue).

If 200 and matches: also fetch `{$CFG.npm_registry}/{$PKG_NAME}` and check `dist-tags.latest`. If `latest` is not `$PKG_VERSION`, ask:

AskUserQuestion(
  question: "{$PKG_NAME}@latest is {actual}, expected {$PKG_VERSION}. Did you intend to publish a non-latest (e.g. patch on old major)?",
  options: ["Yes, intended", "No — should be latest", "Custom"]
)

If "No", explain that `npm publish` requires `--tag latest` to override; the user can `npm dist-tag add {$PKG_NAME}@{$PKG_VERSION} latest` manually.

If `$CFG.use_provenance` is true: GET `{$CFG.npm_registry}/-/npm/v1/attestations/{$PKG_NAME}@{$PKG_VERSION}`. Expect 200. If 404, warn that provenance attestation is missing (npm publish step may have lacked `--provenance` or missing `id-token: write` permission).

Display:

```
npm registry: ✓ {$PKG_NAME}@{$PKG_VERSION} live (latest, provenance: present)
              {npm package URL}
```

#### 9b. Verify GitHub Release exists

GET `https://api.github.com/repos/{owner}/{repo}/releases/tags/{$TAG_NAME}`.

If 200: the workflow's gh-release step succeeded (lucky!). Skip 9c.
If 404: proceed to 9c.

#### 9c. Create GitHub Release via REST API (fallback)

POST to `https://api.github.com/repos/{owner}/{repo}/releases` with:

```json
{
  "tag_name": "{$TAG_NAME}",
  "name": "{$RELEASE_TITLE}",
  "body": "{contents of /tmp/pollinate-{$PKG_VERSION}-body.md}",
  "draft": false,
  "prerelease": false,
  "make_latest": "true"
}
```

Use `GH_TOKEN` from `$GITHUB_TOKEN` env var, `~/.github_token` file, or one of `$GITHUB_PERSONAL_TOKEN`, the project's own `.secrets/` directory (read but never log). If no token available, halt with link to https://github.com/settings/tokens.

Note for Windows + PowerShell users: the JSON body must be encoded with explicit `[string]` cast and UTF-8 no-BOM file write to avoid PSObject-as-payload bugs. See `commands/pollinate.md` "PowerShell encoding" appendix below.

#### 9d. (Optional) Mirror to GitHub Packages

Skip if `$CFG.github_packages_mirror` is `false` (the default for `$REPO_TYPE = "npm-package"` and always for `$REPO_TYPE = "plain"`).

For `$REPO_TYPE = "multi-registry"` this step always runs.

Run a second `npm publish` with the GitHub Packages registry:

```bash
cd {$CFG.npm_dir}
echo "//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}" > .npmrc.gh
echo "@{$ORG}:registry=https://npm.pkg.github.com" >> .npmrc.gh
npm publish --userconfig .npmrc.gh --access public
rm .npmrc.gh
```

This makes the package appear in the GitHub repo's "Packages" sidebar widget. (Provenance attestation alone is not always sufficient — the GitHub Packages registry entry is what triggers the widget for older repos.)

### Step 10: Backfill Missing Prior Releases (Idempotent)

Skip if `$CFG.backfill_on_publish` is `false`.

1. List all tags matching `$CFG.tag_pattern` (turning `{version}` into `*`):
   - `tag_pattern = "v{version}"` → glob `v*`
   - `tag_pattern = "ts-v{version}"` → glob `ts-v*`

   `git ls-remote --tags origin '{glob}' | awk '{print $2}' | sed 's|refs/tags/||;s|\^{}$||' | sort -u`

2. List all existing GitHub Releases via API: `GET /repos/{owner}/{repo}/releases?per_page=100`.

3. Compute the set of tags WITHOUT Releases. This is the backfill list.

4. If the backfill list is empty: display "Backfill: nothing to do."

5. Otherwise:

   ```
   Found {N} prior tags missing GitHub Releases:
     - {tag1}  ({when})
     - {tag2}  ({when})
     ...
   ```

   AskUserQuestion(
     question: "Backfill {N} missing Releases now?",
     options: ["Yes, backfill all", "No, skip", "Pick specific tags", "Custom"]
   )

   If "Yes": for each missing tag, extract its annotation body (`git tag -l --format='%(contents)' {tag}`) plus matching CHANGELOG section if extractable, POST a Release via REST API with `make_latest: "false"` (so historical releases don't override the current latest). Display per-tag result.

   If "Pick specific": present the list as a multi-select AskUserQuestion (limit 4 per call; if more, batch).

   If "No": note for follow-up and continue.

### Step 11: Final Report

Display a comprehensive summary:

```
🐝 Pollinate complete!

Package:        {$PKG_NAME}@{$PKG_VERSION}
Tag:            {$TAG_NAME}
Release:        {$RELEASE_TITLE}

✓ Pushed to origin/main: {HEAD_SHA[:8]}
✓ Tag pushed: {$TAG_NAME}
✓ CI workflow: {success | partial-failure-handled | skipped}
✓ npm registry: {$PKG_NAME}@{$PKG_VERSION} (latest, provenance: {present|absent})
✓ GitHub Release: {URL}
{✓ GitHub Packages: published if mirrored}
{✓ Backfill: {N} prior Releases created}

Verification URLs:
  npm:    https://www.npmjs.com/package/{$PKG_NAME}/v/{$PKG_VERSION}
  github: https://github.com/{owner}/{repo}/releases/tag/{$TAG_NAME}
  CI run: {workflow_run_url}
```

If `$WARNINGS` is non-empty, also display a "Follow-ups" block listing each:

```
Follow-ups:
  - {warning 1 with suggested fix}
  - {warning 2 with suggested fix}
```

Then offer the exit menu:

AskUserQuestion(
  question: "Pollinate complete. Next?",
  options: [
    "Archive spec",
    "Start a new spec",
    "Stay here — more work in this spec",
    "Custom"
  ]
)

- **Archive spec**: invoke `/bee:archive-spec`
- **Start a new spec**: invoke `/bee:new-spec`
- **Stay here**: end command, return to user

---

## PowerShell encoding appendix (Windows-specific)

When running on Windows with PowerShell 5.1, GitHub REST API calls require careful JSON encoding. PowerShell 5.1's `ConvertTo-Json` wraps file-content strings in PSObject metadata, which the GitHub API rejects with HTTP 422. Use this pattern:

```powershell
$body = [string](Get-Content $path -Raw -Encoding UTF8 | Out-String).TrimEnd();
$payload = New-Object -TypeName PSObject -Property @{
  tag_name = $tag;
  name = $title;
  body = $body;       # explicit [string] cast above is REQUIRED
  draft = $false;
  prerelease = $false;
  make_latest = 'true'
};
$json = $payload | ConvertTo-Json -Compress -Depth 10;
[System.IO.File]::WriteAllText($payloadFile, $json,
  (New-Object System.Text.UTF8Encoding $false));   # no-BOM UTF-8
Invoke-RestMethod -Method Post -Uri $url -Headers $headers -InFile $payloadFile;
```

PowerShell 7+ handles this without the `[string]` cast, but pollinate targets the lowest common denominator.

---

## Lifecycle config schema (full reference)

Add to `.bee/config.json`:

```json
{
  "lifecycle": {
    "repo_type": "npm-package",
    "publishes_to_npm": true,
    "version_source": "package_json",
    "version_file_path": "VERSION",
    "npm_dir": ".",
    "npm_registry": "https://registry.npmjs.org",
    "tag_pattern": "v{version}",
    "release_title_pattern": "v{version}",
    "tag_message_source": "changelog",
    "changelog_path": "CHANGELOG.md",
    "github_packages_mirror": false,
    "use_provenance": true,
    "backfill_on_publish": true,
    "wait_for_ci_seconds": 600,
    "ci_workflow_filename": "publish.yml",
    "ci_workflow_skip": false,
    "branch_protection": "fast-forward"
  }
}
```

### Field reference

| Field | Type / Values | Used when |
|---|---|---|
| `repo_type` | `"npm-package"` \| `"plain"` \| `"multi-registry"` | Always — top-level discriminator |
| `publishes_to_npm` | bool — derived from `repo_type` | Gates npm-related steps |
| `version_source` | `"package_json"` \| `"version_file"` \| `"git_tags"` \| `"manual"` | Step 3 dispatch |
| `version_file_path` | string | Only when `version_source = "version_file"` |
| `npm_dir` | string (relative path) | Only when `publishes_to_npm = true` |
| `npm_registry` | URL | Only when `publishes_to_npm = true` |
| `use_provenance` | bool | Only when `publishes_to_npm = true` |
| `github_packages_mirror` | bool | Only when `repo_type = "multi-registry"` |
| `tag_pattern` | string with `{version}` placeholder | Always |
| `release_title_pattern` | string with `{version}` placeholder | Always |
| `tag_message_source` | `"changelog"` \| `"manual"` \| `"both"` | Always |
| `changelog_path` | string | Always (warning if missing) |
| `backfill_on_publish` | bool | Always |
| `ci_workflow_filename` | string | Only when `ci_workflow_skip = false` |
| `ci_workflow_skip` | bool | Always |
| `wait_for_ci_seconds` | number | Only when `ci_workflow_skip = false` |
| `branch_protection` | `"fast-forward"` \| `"pr-required"` | Always |

### Per-project examples

**Single-stack TS package** (e.g. OuronetCore):

```json
"lifecycle": {
  "publishes_to_npm": true,
  "tag_pattern": "v{version}",
  "release_title_pattern": "v{version}"
}
```

All other fields use defaults.

**Dual-stack repo with TS port in subdirectory** (e.g. DALOS_Crypto):

```json
"lifecycle": {
  "publishes_to_npm": true,
  "npm_dir": "ts",
  "tag_pattern": "ts-v{version}",
  "release_title_pattern": "v{version}",
  "ci_workflow_filename": "ts-publish.yml"
}
```

The `tag_pattern` keeps the `ts-` prefix for git-side disambiguation (the Go reference uses `v*`); the `release_title_pattern` drops it for clean display on GitHub Releases page (matching what npm shows).

**Internal/private package** (no GitHub Release, just npm):

```json
"lifecycle": {
  "repo_type": "npm-package",
  "publishes_to_npm": true,
  "ci_workflow_skip": true,
  "backfill_on_publish": false
}
```

The user runs `npm publish` outside of pollinate; pollinate just handles git operations + verification.

**Plain GitHub repo** (versioning + Releases only, no package registry):

```json
"lifecycle": {
  "repo_type": "plain",
  "publishes_to_npm": false,
  "version_source": "version_file",
  "version_file_path": "VERSION",
  "tag_pattern": "v{version}",
  "release_title_pattern": "v{version}",
  "tag_message_source": "changelog",
  "changelog_path": "CHANGELOG.md",
  "backfill_on_publish": true,
  "ci_workflow_skip": true,
  "branch_protection": "fast-forward"
}
```

Documentation repos, research notes, internal tooling, anything that wants version + tag + GitHub Release ceremony without any package registry. Pollinate skips B7+B8+B9 (npm-token setup), Step 4c (npm-tarball README), Step 8 (CI publish), Step 9a (npm registry verify), and Step 9d (GitHub Packages mirror). Only Steps 6 (push), 7 (tag), 9b/9c (GitHub Release) and 10 (backfill) run from the publish pipeline.

**Plain repo with git-tag-based versioning** (no VERSION file):

```json
"lifecycle": {
  "repo_type": "plain",
  "publishes_to_npm": false,
  "version_source": "git_tags",
  "tag_pattern": "v{version}",
  "release_title_pattern": "v{version}",
  "ci_workflow_skip": true
}
```

Pollinate runs `git describe --tags --abbrev=0`, strips the `v` prefix, and uses that as the current version. Asks the user for the next version at run-time.

**Multi-registry package** (npmjs + GitHub Packages mirror):

```json
"lifecycle": {
  "repo_type": "multi-registry",
  "publishes_to_npm": true,
  "github_packages_mirror": true,
  "use_provenance": true,
  "tag_pattern": "v{version}",
  "release_title_pattern": "v{version}"
}
```

Same flow as `npm-package` but Step 9d also runs (publishes a copy to `https://npm.pkg.github.com` so the GitHub repo's Packages sidebar widget appears).

---

## Design Notes (do not display to user)

- **Repo-type taxonomy keeps the command general.** The `npm-package` / `plain` / `multi-registry` split lets pollinate work for documentation repos, internal tooling, and research notes (plain) just as well as for published packages (npm-package) and dual-target packages (multi-registry). The wizard's B0 substep auto-detects from package.json state and lets the user override. All npm-specific steps (B7-B9, 4c, 9a, 9d) are gated on `$PUBLISHES_TO_NPM` so plain repos take a fast path through the pipeline. Future registry types (PyPI, crates.io, ghcr container images) can be added as new `repo_type` values without disturbing the existing flow.
- **Version source for plain repos.** A `VERSION` file at repo root is the recommended pattern — single source of truth, simple to bump, easy to grep. Git-tag-based versioning (`version_source: "git_tags"`) is supported as a fallback for repos that don't want any in-tree version file. Manual entry (`version_source: "manual"`) is the escape hatch for ad-hoc workflows.
- **Idempotency is load-bearing.** Each step checks for existing state before acting. Re-running pollinate after a partial failure resumes from the first incomplete step.
- **The gh-CLI flag-incompatibility fallback (Step 9c)** is the single most important feature. Both the bee-dev publish workflow template and the OuronetCore/DALOS forks of it emit a failed `gh release create --notes-from-tag --repo X` on every publish since 2026-04-30 (when the GitHub-hosted runner gh CLI version updated). Pollinate detects this exact failure mode and falls back to REST API creation transparently. The workflow itself should be patched separately to remove the `--repo` flag (or use `--notes-file` instead) — pollinate does NOT patch the workflow because it does not own the workflow file.
- **No force-push, ever.** Step 6 only does fast-forward pushes; if the branch has diverged, it falls back to PR-required flow.
- **GitHub token resolution order:** environment variable → `.secrets/` directory in repo → user's `~/.github_token` → halt. Tokens are never logged.
- **Per-project config drives behavior.** Pollinate is one command; the lifecycle block lets it serve dual-stack monorepos and single-stack packages alike without branching code.
- **Backfill is opt-in per project (default true).** For repos that explicitly don't want historical Releases backfilled (e.g. legacy alpha tags that should stay invisible), set `backfill_on_publish: false`.
- **AskUserQuestion appears at every visible/destructive decision point.** This makes pollinate slower than a fully-automated pipeline, but the publish ceremony is high-blast-radius — deliberation is the feature, not a bug.
- **Windows-PowerShell-friendly.** REST API JSON encoding handles the PowerShell 5.1 PSObject-wrapping bug explicitly. macOS/Linux paths are simpler (jq + curl) and pollinate auto-detects platform.
- **No agents are spawned.** Pollinate runs entirely in main context — every step is short and deterministic enough to execute inline. This also keeps the audit trail clean (every action visible to the user as it happens).
