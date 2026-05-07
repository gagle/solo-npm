---
description: Hotfix a previous stable major (e.g., v1.5.0 → v1.5.1) while main develops the next major. Triggers from prompts like "fix bug in v1", "patch v1.5.0", "backport this fix to v1", "hotfix the v1 rate limiter", "v1 users are reporting a parsing bug", "patch the previous major". Agent creates/checks-out the maintenance branch (`<major>.x`), applies the fix per your description, releases with the correct dist-tag (`@latest` if still current stable, `@v<major>` legacy if newer major has shipped). AI-driven; no manual git checkout or package.json edits.
---

# Hotfix

Backward maintenance hotfix on a `<major>.x` branch. AI-driven end-to-end: the user describes the fix in chat (or in the invoking prompt); the agent creates/checks-out the maintenance branch, makes the fix, runs `/verify`, releases the patch with the correct dist-tag, returns to main.

## When to use

- `main` is on a newer major (`2.0.0-beta.x` or already-promoted `2.0.0`).
- A bug surfaces in the previous major's line (e.g., `1.5.0`) that needs an immediate fix.
- You don't want to interrupt the v2 work to ship the v1 patch.

This skill creates a `<major>.x` (e.g., `1.x`) maintenance branch checked out from the most recent `v<major>.*` tag. Fixes land on this branch and tag as `v1.5.1`. Dist-tag handling is dynamic based on whether the maintenance major is still "current stable" or "legacy".

## Phase 0 — Read prompt context

Scan the user's invoking prompt (and recent chat turns) for hints:

| Prompt mentions | Pre-fill |
|---|---|
| `v1`, `v1.5`, `1.x`, `the previous major`, `v1 line` | `TARGET_MAJOR` |
| Any descriptive phrase about the bug or component (`rate limiter`, `crashes on 429`, `parsing bug`, `auth header missing`) | `FIX_DESCRIPTION` to the user's verbatim phrasing |
| `--cherry-pick <sha>` flag, *"backport this commit"*, *"port commit abc123"*, *"cherry-pick abc1234 to v1"* | `CHERRY_PICK_SHA` (resolve from prompt or recent chat context); skip both `FIX_DESCRIPTION` prompt **and** the "describe the fix" message in Phase D |

Examples:
- *"Hotfix the v1 rate limiter — it crashes on 429"* → `TARGET_MAJOR=1`, `FIX_DESCRIPTION="rate limiter crashes on 429"`. Skip "Which major?" question; skip "describe the fix" prompt.
- *"Patch v1.5"* → `TARGET_MAJOR=1`. Still ask for fix description.
- *"Backport this fix to v1"* (where "this fix" refers to a recent fix on main) → `TARGET_MAJOR=1`, extract `FIX_DESCRIPTION` from recent chat context. Skip both prompts.
- *"v1 users are reporting a parsing bug"* → `TARGET_MAJOR=1`, `FIX_DESCRIPTION="parsing bug reported by users"`. Skip both.
- *"Cherry-pick abc1234 to v1"* → `TARGET_MAJOR=1`, `CHERRY_PICK_SHA="abc1234"`. Backport mode (Phase D applies via `git cherry-pick` instead of agent code edits).
- *"--cherry-pick abc1234"* (explicit flag) → `CHERRY_PICK_SHA="abc1234"`. Same backport mode; `TARGET_MAJOR` still resolved via Phase A if not pre-filled.
- Bare *"hotfix"* → ask both questions; cherry-pick mode off.

If extraction is ambiguous, pre-fill what's clear and ask the rest. Don't over-extract.

## Phase 0.5 — Prompt-extraction validation (E1 from v0.11.0)

After Phase 0 pre-fills slots, validate against the canonical regex framework in `/unpublish` Phase 0.5. Slots specific to `/hotfix`:

- **`TARGET_MAJOR`** — must be a positive integer (`^[0-9]+$`). Rejects "v1" (with v-prefix), "1.x" (range syntax), "first" (English).
- **`CHERRY_PICK_SHA`** — git SHA regex `^[a-f0-9]{7,40}$`. Rejects partial typed SHAs that wouldn't match a real commit.
- **`NEXT_VERSION`** (computed) — re-validate as semver after the patch-bump computation in Phase E.2.

On validation failure, STOP with diagnostic.

## Phase 0.5b — Shell-safety hardening (Tier-4 #4 from v0.13.0)

Apply the shell-safety check from `/unpublish` Phase 0.5b (canonical) to `TARGET_MAJOR`, `CHERRY_PICK_SHA`, and `NEXT_VERSION` slots. Same metacharacter blacklist + double-quoted interpolation convention.

## Phase A — Pre-flight + state detection

1. Working tree clean (`git status --porcelain`); else STOP with the standard "commit or stash first" message.
2. Verify `release.yml` honours `publishConfig.tag` (grep for `EXPLICIT_TAG=` in the workflow). If missing, **auto-chain to `/solo-npm:init --refresh-yml` after a single approval gate**:

   ```
   Header:   "Refresh release.yml"
   Question: "Your release.yml doesn't honour publishConfig.tag. A hotfix to a legacy major would publish to @latest and clobber stable users. Refresh now?"
   Options:
     - Refresh release.yml and continue (Recommended) — runs /solo-npm:init --refresh-yml
     - Abort
   ```

   On `Refresh and continue`: invoke `/solo-npm:init --refresh-yml` (it performs targeted insert + commits + pushes a single `chore: refresh release.yml for dist-tag detection` commit). Then re-enter Phase A from step 1.

   On `Abort`: STOP gracefully. Print: *"Aborted. Re-run `/solo-npm:hotfix` when ready."*

3. List published majors from `git tag --list 'v*' --sort=-v:refname | head -20`. Group by major number. Identify candidate maintenance lines:
   - `currentMain` = LATEST_TAG's major number
   - `candidates` = all distinct majors with at least one published tag, **excluding** `currentMain` (because patching the current major is just `/release`, not a hotfix)

4. If `candidates` is empty, STOP:

   > No older majors to hotfix. The current line is `v<currentMain>.*` — use `/release` to ship a patch.

5. If `candidates` has exactly one entry AND Phase 0 didn't pre-fill `TARGET_MAJOR`: auto-pick that single candidate (no prompt needed; only one option).

6. If `candidates` has multiple entries AND `TARGET_MAJOR` not pre-filled, AskUserQuestion:

   ```
   Header:   "Hotfix major"
   Question: "Which major to hotfix?"
   Options:  one entry per candidate (label: "v<major>.x line (latest tag: v<major>.<minor>.<patch>)") + Abort
   ```

## Phase B — Branch

Branch name: `<TARGET_MAJOR>.x` (e.g., `1.x`). Source tag: most recent `v<TARGET_MAJOR>.*` tag.

**Shallow-clone detection (B4, v0.11.0)**: if the repo was cloned with `--depth=N` (common in CI), older tags + their commit history may be missing. `git tag --list` returns local tags only; missing source tag = silent failure when `git checkout -b` later can't find the ref.

```bash
if [ "$(git rev-parse --is-shallow-repository 2>/dev/null)" = "true" ]; then
  echo "ERROR: this is a shallow clone — older release tags + their commit history may be missing."
  echo "       /solo-npm:hotfix needs the full history to branch from v${TARGET_MAJOR}.* tags."
  echo
  echo "Fix:   git fetch --unshallow origin"
  echo "       Then re-invoke /solo-npm:hotfix."
  exit 1
fi

timeout 60 git fetch --tags origin
SOURCE_TAG=$(git tag --list "v${TARGET_MAJOR}.*" --sort=-v:refname | head -1)
if [ -z "$SOURCE_TAG" ]; then
  echo "ERROR: no v${TARGET_MAJOR}.* tags found locally even after 'git fetch --tags'."
  echo "       Either v${TARGET_MAJOR} was never released, or the fetch failed silently."
  echo "       Verify: git ls-remote --tags origin | grep 'v${TARGET_MAJOR}\\.'"
  exit 1
fi

if git ls-remote --heads --exit-code origin "${TARGET_MAJOR}.x" >/dev/null 2>&1; then
  # Branch exists on remote — continuing maintenance
  git checkout "${TARGET_MAJOR}.x"
  git pull --ff-only
else
  # First hotfix on this line — create from source tag
  git checkout -b "${TARGET_MAJOR}.x" "${SOURCE_TAG}"
  git push -u origin "${TARGET_MAJOR}.x"
fi
```

## Phase C — Compute target dist-tag

Determine whether the maintenance line is still "current stable" or "legacy":

```bash
# Most recent stable tag (excludes pre-releases)
LATEST_STABLE=$(git tag --list 'v*' --sort=-v:refname | grep -vE -- '-[a-z]+\.[0-9]+$' | head -1)
LATEST_STABLE_MAJOR=$(echo "$LATEST_STABLE" | sed -E 's/^v([0-9]+).*/\1/')
```

| Comparison | Target dist-tag | Reason |
|---|---|---|
| `TARGET_MAJOR >= LATEST_STABLE_MAJOR` | `latest` | The hotfix line IS the current stable line (even if main is in pre-release for the next major). |
| `TARGET_MAJOR < LATEST_STABLE_MAJOR` | `v<TARGET_MAJOR>` | A newer stable major already ships from main; this hotfix is legacy. |

Surface the target dist-tag to the user as part of Phase D's plan (so they understand where the patch will land).

## Phase D.0 — Error-handling patterns (H2, H4, H6 from `/unpublish` reference)

Before applying the fix and progressing through Phases D–G, apply the standard solo-npm error patterns. Canonical wording lives in `/unpublish` Phases C.0–D.2:

- **H2 — `.solo-npm/state.json` corruption guard**: state.json reads (e.g., the bundle-size cache write in Phase E.7, or any future hotfix-aware cache lookup) wrap in try/catch. On parse fail surface non-fatal warning, treat as empty cache, continue.
- **H4 — Registry propagation lag retry**: Phase E.5 verify (`npm view <pkg>@<NEXT_VERSION>` and `npm view <pkg> dist-tags.<target-tag>`) uses 3 attempts × 5s sleep before declaring inconsistency. Don't HARD STOP the hotfix on lag — the publish via CI succeeded, this is verification lag.
- **H6 — Chain-target failure recovery**: chains into `/init --refresh-yml` (Phase A if release.yml stale), `/release` (Phase F.5 forward-port). Capture child STOP messages verbatim and surface in `/hotfix` context with retry/abort options. For Phase F.5: if the forward-port chain fails after a successful cherry-pick, the cherry-pick is already on main — surface clearly so the user can resume manually.

## Phase D.0a — H5 concurrent-release lock (v0.11.0)

Same per-repo lock pattern as `/solo-npm:release` Phase C.0a (canonical reference). Acquire `.solo-npm/locks/<repo-root>.lock` with stale-PID auto-cleanup before applying the fix. A `/hotfix` race with `/release` on the same repo could otherwise produce duplicate tags or interleaved commits across maintenance branches.

## Phase D — Apply fix

Two sub-paths: **cherry-pick mode** (when `CHERRY_PICK_SHA` was extracted in Phase 0) or **describe-the-fix mode** (everything else).

### D.1 Cherry-pick mode (`CHERRY_PICK_SHA` set)

The fix already exists as a commit on another branch (typically main). Apply it via `git cherry-pick`:

```bash
git cherry-pick <CHERRY_PICK_SHA>
```

- On success → continue to `/verify`, then Phase E.
- On conflict → invoke the [Cherry-pick conflict handoff](#cherry-pick-conflict-handoff) helper. The skill exits cleanly; the user resolves manually and continues with `/release` separately.

This path skips agent-skills delegation entirely — there's no fix to author, only one to apply.

### D.2 Describe-the-fix mode (default)

This path does *user code work* — not release infrastructure. solo-npm delegates to `agent-skills:debugging-and-error-recovery` when available.

If Phase 0's prompt-context extraction already captured `FIX_DESCRIPTION`, proceed directly. Otherwise surface this verbatim:

> ### Ready to hotfix v<TARGET_MAJOR>.x
>
> I've checked out the `<TARGET_MAJOR>.x` branch from `<SOURCE_TAG>`.
> Target dist-tag for this hotfix: **`<latest|v<major>>`** (because <reason>).
>
> **Describe the fix** in chat — I'll make the changes here, run `/verify`, and ship the patch.

When the fix description is in hand:

1. **Check if agent-skills is installed.** Look for `agent-skills@*` in Claude Code's enabled plugins (e.g., via `/plugin marketplace list` output or by checking `~/.claude/plugins/cache/*/agent-skills/`).
2. **If installed**: invoke `/agent-skills:debugging-and-error-recovery` with the fix description as context. That skill brings reproduce → localise → reduce → fix → guard methodology, suited to a backward-maintenance bug fix where the v1 codebase may differ from main's. After it returns, run `/verify`.
3. **If not installed**: the agent applies the fix directly using its general code-editing tools (Edit, Write, Bash) per the `FIX_DESCRIPTION`. This is the fallback — solo-npm works standalone. Recommended setup is solo-npm + agent-skills together (see README "Scope and partners").

### After D.1 or D.2

Run `/verify` (composes with the verify skill, scoped to whatever subset the fix touched).

If verify fails:
- If agent-skills was used in D.2, the debugging-and-error-recovery skill will surface diagnostic context. Allow user to direct next step.
- Otherwise, surface the failure and let user direct iteration.

## Phase E — Release

After verify passes:

### E.1 Set `publishConfig.tag` if needed

If target dist-tag is `latest`: leave `publishConfig.tag` unset (or remove it if present) — `latest` is the registry default.

If target dist-tag is `v<TARGET_MAJOR>`: set `publishConfig.tag = "v<TARGET_MAJOR>"` in `package.json` so `release.yml`'s detection step uses it.

```bash
# For legacy line (only):
node -e "const fs=require('fs'); const p=require('./package.json'); p.publishConfig=p.publishConfig||{}; p.publishConfig.tag='v${TARGET_MAJOR}'; fs.writeFileSync('./package.json.tmp', JSON.stringify(p,null,2)+'\n'); fs.renameSync('./package.json.tmp', './package.json');"
# Atomic write (H7 pattern): write to .tmp + rename. A killed process never leaves
# a corrupted package.json — the rename is a single atomic inode swap on POSIX.
```

### E.2 Compute next version

`NEXT_VERSION` = patch bump from `SOURCE_TAG` (e.g., `v1.5.0` → `1.5.1`).

```bash
npm version <NEXT_VERSION> --no-git-tag-version
```

### E.3 Render plan + approval gate

Render summary + changelog draft (single Bug Fixes entry from the fix's commit message) to chat.

```
Header:   "Hotfix"
Question: "Approve the hotfix release plan above?"
Options:  Proceed with v<NEXT_VERSION> (publishes to @<target-tag>) / Abort
```

### E.4 Commit, tag, publish

After approval, apply the canonical git push rejection categorization from `/solo-npm:release` C.3 and the tag-collision pre-flight from `/solo-npm:release` C.5:

```bash
git add CHANGELOG.md package.json
git commit -m "chore: release v${NEXT_VERSION}"
# Pre-commit hook failure rollback (B3, v0.11.0): if `git commit` fails AND
# stderr matches "pre-commit hook"/"hook .* failed", surface case-by-case
# recovery options per /solo-npm:release C.2 canonical (--no-verify bypass,
# git reset HEAD, fix-and-retry). Working tree is staged-but-uncommitted;
# never silently continue past hook failure.
HOTFIX_SHA=$(git rev-parse HEAD)   # captured for Phase F.5 forward-port

# Push the maintenance-branch commit with rejection categorization
# (non-FF / pre-receive hook / pre-push hook (Tier-3 L) / branch-protection / auth)
# + SSL/TLS error remediation per /unpublish Phase −1.10.
# See /release C.3 canonical for full case-by-case wording.
PUSH_OUT=$(timeout 60 git push origin "${TARGET_MAJOR}.x" 2>&1)
PUSH_RC=$?
if [ $PUSH_RC -ne 0 ]; then
  echo "ERROR: git push origin ${TARGET_MAJOR}.x failed (exit $PUSH_RC):"
  echo "$PUSH_OUT" | sed 's/^/  /'
  ssl_remediation_if_applicable "$PUSH_OUT" || true
  exit 1
fi

# Tag-collision pre-flight (per /release C.5 canonical pattern)
if timeout 30 git ls-remote --tags origin "refs/tags/v${NEXT_VERSION}" 2>/dev/null | grep -q .; then
  echo "ERROR: tag v${NEXT_VERSION} already exists on origin."
  echo "       Investigate: git fetch --tags && git show v${NEXT_VERSION}"
  echo "       If safe to overwrite: git push origin :refs/tags/v${NEXT_VERSION}, then re-run /solo-npm:hotfix."
  exit 1
fi

git tag "v${NEXT_VERSION}"

# Push tag with rejection categorization; clean up local tag on failure
TAG_PUSH_OUT=$(timeout 60 git push --tags 2>&1)
if [ $? -ne 0 ]; then
  git tag -d "v${NEXT_VERSION}" 2>/dev/null   # don't leave a local-only tag dangling
  echo "ERROR: git push --tags failed:"
  echo "$TAG_PUSH_OUT" | sed 's/^/  /'
  exit 1
fi

timeout 1800 gh run watch --exit-status   # 30min cap; fail-fast on long CI
```

`HOTFIX_SHA` holds the release commit on `<TARGET_MAJOR>.x` — used by Phase F.5 if the user opts to forward-port to main.

### E.5 Verify on registry

```bash
npm view "<PACKAGE_NAME>@${NEXT_VERSION}" version
npm view "<PACKAGE_NAME>" dist-tags."<target-tag>"
```

The second should equal `NEXT_VERSION` — confirms the publish landed on the correct dist-tag.

If target was `v<TARGET_MAJOR>` (legacy), also verify `dist-tags.latest` did NOT change (still points at the newer major's most recent stable).

### E.6 Create GitHub Release with notes

After registry verify, create a GitHub Release page on the `<TARGET_MAJOR>.x` branch's tag.

**Critical**: when the target dist-tag is `v<TARGET_MAJOR>` (legacy line — newer stable already shipped from main), pass `--latest=false` to `gh release create` so the GitHub Releases UI does NOT mark this hotfix as the latest release. The latest release tag should stay on the current main-line stable.

```bash
# Pre-flight (same pattern as /release C.7.5 — distinguish not-installed vs not-authenticated):
if ! command -v gh >/dev/null 2>&1; then
  echo "⚠ gh not installed; skipping GitHub Release creation. Install from https://cli.github.com"
elif ! gh auth status >/dev/null 2>&1; then
  echo "⚠ gh installed but not authenticated; skipping. Run: gh auth login"
else
  NOTES=$(awk -v v="${NEXT_VERSION}" '
    $0 ~ "^## v" v "( |$)" { capture=1; next }
    capture && /^## v/ { exit }
    capture { print }
  ' CHANGELOG.md)

  # Determine the latest flag based on whether this is the current stable major:
  if [ "${TARGET_DIST_TAG}" = "v${TARGET_MAJOR}" ]; then
    # Legacy line — don't override "latest" badge on GitHub
    LATEST_FLAG="--latest=false"
  else
    # Current stable major — this IS the new latest
    LATEST_FLAG="--latest"
  fi

  gh release create "v${NEXT_VERSION}" \
    --title "v${NEXT_VERSION}" \
    --notes "${NOTES}" \
    --target "${TARGET_MAJOR}.x" \
    ${LATEST_FLAG}
fi
```

The `--target "${TARGET_MAJOR}.x"` flag explicitly attaches the GitHub Release to the maintenance branch (not main).

### E.7 Update bundle-size baseline cache

Same pattern as `/release` Phase C.7.6 — write the new patch's `unpackedSize` to `.solo-npm/state.json#pkgCheck.lastSize`, then trim to keep at most the **last 3 versions per package** (semver-sorted descending). The cache is per-package-per-version, so the maintenance line's sizes accumulate alongside main's; the per-package trim treats them as a single version stream and keeps the 3 highest-semver entries regardless of branch.

## Phase F — Return to main

```bash
git checkout main
```

## Phase F.5 — Forward-port to main (gated)

Skipped automatically when Phase D was cherry-pick mode (`CHERRY_PICK_SHA` was set) — the fix already came *from* main, no forward-port needed.

Otherwise, surface the option:

```
Header:   "Forward-port"
Question: "Forward-port the v<NEXT_VERSION> fix to main?"
Options:
  - Yes — cherry-pick and ship as next release on main (Recommended if main has the same bug)
  - Yes — cherry-pick only, no release (I'll review first)
  - No — skip (I'll handle manually if needed)
```

`HOTFIX_SHA` is the commit sha of the hotfix's release commit on the `<TARGET_MAJOR>.x` branch (captured in Phase E.4 before `git checkout main`).

### Yes — cherry-pick and ship

```bash
git cherry-pick <HOTFIX_SHA>
```

- On success → chain to `/release` (which auto-detects state and routes correctly: stable patch / pre-release bump / promote / etc.).
- On conflict → invoke [Cherry-pick conflict handoff](#cherry-pick-conflict-handoff). Skill exits; user resolves manually.

### Yes — cherry-pick only

```bash
git cherry-pick <HOTFIX_SHA>
```

- On success → commit lands on main; the user reviews and runs `/release` separately on their own time.
- On conflict → conflict handoff (same as above).

### No — skip

Print final summary with footer:

> To forward-port later: `git checkout main && git cherry-pick <HOTFIX_SHA>` then `/release`.

## Phase G — Final summary

After Phase F.5 (whatever its outcome):

```
Hotfix v<NEXT_VERSION> published to dist-tag `<latest|v<major>>`.
  Branch:       <TARGET_MAJOR>.x  (preserved on origin for future hotfixes)
  Tarball:      https://www.npmjs.com/package/<PACKAGE_NAME>/v/<NEXT_VERSION>
  CI:           <gh run url>
Forward-port:   <released as v<X> on main | cherry-picked, ready for /release | skipped | conflict — manual resolution required>
You're back on main.
```

## Cherry-pick conflict handoff

Shared helper used by Phase D's cherry-pick mode and Phase F.5's forward-port. When `git cherry-pick` exits non-zero:

1. Read `git status --porcelain` to identify conflicted files.
2. Surface to the user verbatim:

   ```
   Cherry-pick conflict on these files:
     - <file-1> (both modified)
     - <file-2> (deleted by us, modified by them)

   The cherry-pick is paused (`.git/CHERRY_PICK_HEAD` exists).

   Options:
     - Resolve manually: edit the file(s), `git add`, then `git cherry-pick --continue`
     - Abort: `git cherry-pick --abort` (skip this port)
     - Or describe how the conflict should be resolved and I'll re-attempt
   ```

3. Exit the skill (or the cherry-pick step) cleanly. Do **not** auto-resolve. Do **not** auto-abort.
4. The final summary still prints, with a footer noting the unresolved cherry-pick state.

The user owns conflict resolution because the right resolution depends on intent (e.g., which side of a refactor wins) — auto-merging here is more dangerous than asking.

## Subsequent hotfixes on the same line

The `<major>.x` branch persists on origin. Future invocations of `/solo-npm:hotfix` for the same major:
- Phase B short path: branch exists, `git checkout` + `git pull --ff-only`.
- Phase 0 extraction works the same — user describes the new bug (or supplies `--cherry-pick <sha>` for backport).
- Phase D applies the fix (D.1 cherry-pick or D.2 describe-the-fix).
- Phase E publishes the next patch (e.g., `v1.5.2`).

Once the line is established, hotfixes are just "describe the next bug (or cherry-pick a sha), agent ships."

## What this skill does NOT do

- Auto-resolve cherry-pick conflicts — the user owns conflict resolution because intent matters.
- Maintain multiple `<major>.x` lines in one invocation (call again per major).
- Auto-detect "this fix should go to main too" without asking — Phase F.5 always gates with an `AskUserQuestion`.
- Drive the actual code-editing methodology in describe-the-fix mode — that's `agent-skills:debugging-and-error-recovery` when installed; general agent tools as fallback.
