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

Examples:
- *"Hotfix the v1 rate limiter — it crashes on 429"* → `TARGET_MAJOR=1`, `FIX_DESCRIPTION="rate limiter crashes on 429"`. Skip "Which major?" question; skip "describe the fix" prompt.
- *"Patch v1.5"* → `TARGET_MAJOR=1`. Still ask for fix description.
- *"Backport this fix to v1"* (where "this fix" refers to a recent fix on main) → `TARGET_MAJOR=1`, extract `FIX_DESCRIPTION` from recent chat context. Skip both prompts.
- *"v1 users are reporting a parsing bug"* → `TARGET_MAJOR=1`, `FIX_DESCRIPTION="parsing bug reported by users"`. Skip both.
- Bare *"hotfix"* → ask both questions.

If extraction is ambiguous, pre-fill what's clear and ask the rest. Don't over-extract.

## Phase A — Pre-flight + state detection

1. Working tree clean (`git status --porcelain`); else STOP with the standard "commit or stash first" message.
2. Verify `release.yml` honours `publishConfig.tag` (grep for `EXPLICIT_TAG=` in the workflow). If missing, STOP with:

   > Your `release.yml` doesn't honour `publishConfig.tag`. A hotfix to a legacy major would publish to `@latest` and clobber stable users. Run `/solo-npm:init` to refresh the workflow template.

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

```bash
git fetch origin
SOURCE_TAG=$(git tag --list "v${TARGET_MAJOR}.*" --sort=-v:refname | head -1)

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

## Phase D — Apply fix (composes with agent-skills)

This phase does *user code work* — not release infrastructure. solo-npm delegates to `agent-skills:debugging-and-error-recovery` when available.

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

After the fix lands, run `/verify` (composes with the verify skill, scoped to whatever subset the fix touched).

If verify fails:
- If agent-skills was used in Phase D, the debugging-and-error-recovery skill will surface diagnostic context. Allow user to direct next step.
- Otherwise, surface the failure and let user direct iteration.

## Phase E — Release

After verify passes:

### E.1 Set `publishConfig.tag` if needed

If target dist-tag is `latest`: leave `publishConfig.tag` unset (or remove it if present) — `latest` is the registry default.

If target dist-tag is `v<TARGET_MAJOR>`: set `publishConfig.tag = "v<TARGET_MAJOR>"` in `package.json` so `release.yml`'s detection step uses it.

```bash
# For legacy line (only):
node -e "const p=require('./package.json'); p.publishConfig=p.publishConfig||{}; p.publishConfig.tag='v${TARGET_MAJOR}'; require('fs').writeFileSync('./package.json', JSON.stringify(p,null,2)+'\n');"
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

After approval:

```bash
git add CHANGELOG.md package.json
git commit -m "chore: release v${NEXT_VERSION}"
git push origin "${TARGET_MAJOR}.x"

git tag "v${NEXT_VERSION}"
git push --tags

gh run watch --exit-status
```

### E.5 Verify on registry

```bash
npm view "<PACKAGE_NAME>@${NEXT_VERSION}" version
npm view "<PACKAGE_NAME>" dist-tags."<target-tag>"
```

The second should equal `NEXT_VERSION` — confirms the publish landed on the correct dist-tag.

If target was `v<TARGET_MAJOR>` (legacy), also verify `dist-tags.latest` did NOT change (still points at the newer major's most recent stable).

## Phase F — Return to main

```bash
git checkout main
```

Print final summary:

```
Hotfix v<NEXT_VERSION> published to dist-tag `<latest|v<major>>`.
  Branch:       <TARGET_MAJOR>.x  (preserved on origin for future hotfixes)
  Tarball:      https://www.npmjs.com/package/<PACKAGE_NAME>/v/<NEXT_VERSION>
  CI:           <gh run url>
You're back on main.
```

## Subsequent hotfixes on the same line

The `<major>.x` branch persists on origin. Future invocations of `/solo-npm:hotfix` for the same major:
- Phase B short path: branch exists, `git checkout` + `git pull --ff-only`.
- Phase 0 extraction works the same — user describes the new bug.
- Phase D applies the fix.
- Phase E publishes the next patch (e.g., `v1.5.2`).

Once the line is established, hotfixes are just "describe the next bug, agent ships."

## Forward-porting fixes to main

This skill does NOT auto-port the fix back to main. After a hotfix ships, the fix exists on `<TARGET_MAJOR>.x` but main may still have the bug (or its v2-rewrite equivalent).

Options:

- **If the fix applies cleanly to main's code**: `git checkout main && git cherry-pick <hotfix-commit-sha>`. Then `/release` to ship the next patch.
- **If main has a different API**: manually re-apply the fix with main's API, then `/release`.

Auto-porting is out of scope for v0.5.4.

## What this skill does NOT do

- Cherry-pick or merge the hotfix back to main (manual; see above).
- Maintain multiple `<major>.x` lines in one invocation (call again per major).
- Auto-detect "this fix should go to main too" — the user decides.
- Drive the actual code-editing methodology — that's `agent-skills:debugging-and-error-recovery` when installed; general agent tools as fallback.
