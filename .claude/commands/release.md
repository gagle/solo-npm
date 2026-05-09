---
description: Universal release entry point — tag-triggered npm release with OIDC provenance and one approval gate. Triggers from prompts like "ship it", "release this", "cut a release", "time to release v0.5.4", "push to npm", "get on npm". Auto-chains to /solo-npm:prerelease on pre-release version. Three phases (pre-flight, plan, execute). Wrap via .claude/skills/release/SKILL.md for repo-specific narrative.
---

# Release

> **This is the opinionated baseline.** Consumer repos typically wrap it
> via `.claude/skills/release/SKILL.md` to add repo-specific narrative
> (workspace shape, prepare-dist usage, custom verification commands).
> Invoke `/solo-npm:release` directly only when you want the unmodified
> baseline.

## Table of contents

- [Principle: skills are operators, not advisors](#principle-skills-are-operators-not-advisors)
- [Manual overrides (power-user escape hatches)](#manual-overrides-power-user-escape-hatches)
- [Phase 0 — Read prompt context](#phase-0--read-prompt-context)
- [Who this is for](#who-this-is-for) / [How it works](#how-it-works) / [Auto-detection](#auto-detection)
- [Phase 0.5 — Prompt-extraction validation](#phase-05--prompt-extraction-validation-e1-e3-from-v0110)
- [Phase 0.5b — Shell-safety hardening](#phase-05b--shell-safety-hardening-tier-4-4-from-v0130)
- [Phase A — Pre-flight](#phase-a--pre-flight) — working tree, verify, trust cache, audit cache
- [Phase B — Plan and confirm](#phase-b--plan-and-confirm) — collect commits, derive version, render plan, AskUserQuestion
- [Phase C.0 — Error-handling patterns](#phase-c0--error-handling-patterns-h2-h3-h4-h6-from-unpublish-reference) — H2/H3/H4/H6 references
- [Phase C.0a — H5 concurrent-release lock](#phase-c0a--h5-concurrent-release-lock-v0110)
- [Phase C — Execute](#phase-c--execute) — CHANGELOG, commit, push (with rejection categorization), tag (with collision pre-flight), CI watch, registry verify, GitHub Release (warn-not-block), bundle-size cache update
- [Phase G — Post-major deprecation chain (gated)](#phase-g--post-major-deprecation-chain-gated)
- [Failure modes and recovery](#failure-modes-and-recovery)
- [What this skill does NOT do](#what-this-skill-does-not-do)

## Phase −0 — Help mode (per `/unpublish` canonical)

If the user's prompt contains `--help` / `-h` / `"how does /solo-npm:release work"` / similar, surface a help summary **INSTEAD** of running the skill.

Synthesize from the **Phases** outline (0 / 0.5 / 0.5b / A.1–A.5 / B / C.0 / C.0a / C.1–C.7.6 / G), and 2–3 trigger phrases (e.g., *"ship it"*, *"release this"*, *"time to release v0.5.4"*). Note the auto-chain to `/prerelease` on pre-release version + post-major deprecate chain Phase G. See `/unpublish` Phase −0 for canonical format.

After surfacing, **STOP**. Re-invocation without help triggers runs normally.

## Principle: skills are operators, not advisors

When this skill detects a fixable precondition (missing trust, missing
release.yml, etc.), **automatically invoke the appropriate sibling
skill** without asking the user. The sibling skill has its own approval
gate before destructive operations — that's the only user-facing prompt
needed.

Do NOT pause to offer alternative invocation paths (e.g., the
`pnpm npm-trust:setup` shorthand vs. the `/solo-npm:trust` skill). Pick
one path (always the skill) and execute. The user invoked `/release`;
they want a release, not a menu of options.

Minimum-friction shape: **one `AskUserQuestion` per destructive
operation, max.** Everything fixable auto-chains.

## Manual overrides (power-user escape hatches)

| Scenario | Action |
|---|---|
| Want to start a pre-release line | Use `/solo-npm:prerelease`. `/release` from a stable version doesn't know to enter pre-release mode. |
| Want to start a hotfix on an older major | Use `/solo-npm:hotfix`. `/release` from main can't infer which older major to patch. |
| Want a different version than the auto-bump (e.g., `1.5.0` jump from `0.4.0`) | Mention the version in the prompt: *"Time to release v1.5.0"*. `/release` Phase 0 detects the named version and pre-fills `NEXT_VERSION`. |
| Want to bump differently than auto-derived | Improve commit messages (`feat!:` for breaking, `feat:` for minor, `fix:` for patch). Auto-bump reflects what your commits say. |
| Want custom changelog text | After the release commit lands, follow up with `docs(changelog): clarify v{X} notes` — auto-changelog regenerates correctly on the next release. |
| Want to continue an in-progress pre-release | Just type `/release`. It auto-detects pre-release version and chains into `/solo-npm:prerelease`. **No manual git/code work.** |

## Phase 0 — Read prompt context

Scan the user's invoking prompt for explicit version mentions (`v0.5.4`, `0.5.4`, `version 1.0.0`). If a specific version is named:

- Pre-fill `NEXT_VERSION` with the named version.
- Continue computing auto-bump from commits in B.4 as usual.
- If auto-bump matches user's named version: proceed silently to B.5 with `NEXT_VERSION` set.
- If auto-bump differs: render both in the plan summary ("auto-bump suggests v0.5.5; you said v0.5.4") and B.5 offers `Proceed with v{user-named} (your override) / Proceed with v{auto-bump} / Abort`.

If no version is named: continue as today (auto-bump derives, B.5 prompts Proceed/Abort).

## Who this is for

You're a solo developer — or running a small group of LLM agents —
shipping an npm package. **PRs are disabled** in your repo
(issue/discussion-only contribution model). There's no committee to
review code, no second pair of human eyes. You want releases that are:

- **Boring**: tested, typed, provenanced, every time, no exceptions.
- **One-touch**: not a 15-step runbook, not a commit-message lottery.
- **Provable**: SLSA attestations on every tarball, traceable back to
  git.

This skill drives the whole flow end-to-end with **one** human
checkpoint (a structured `AskUserQuestion` selector — unmissable,
unmistakable). Type `/release` (or `/solo-npm:release` for the bare
baseline), review the plan once, click `Proceed`, get a notification
when the tarball is on npm with provenance.

## How it works

Three-phase flow. Each phase has a single goal and a clear stop
condition.

- **Phase A** runs in silence if everything is green. Surface only
  failures.
- **Phase B** has the only routine human checkpoint: one
  `AskUserQuestion` selector.
- **Phase C** runs end-to-end after approval. Halt on first failure.

## Auto-detection

Everything repo-specific is auto-detected from the working tree. No
hand-editing required:

| Value | Source |
|---|---|
| Workspace shape | `npm-trust --doctor --json` (`workspace.shape`) |
| Repo slug | `git remote get-url origin` (parsed) OR doctor's `repo.inferredSlug` |
| Package name(s) | `package.json#name` (single) OR each `packages/*/package.json#name` (monorepo) |
| Main package dir (monorepo) | First entry in `pnpm-workspace.yaml#packages` OR doctor's `workspace.packages[0]` |
| Release workflow | `.github/workflows/release.yml` if it exists; doctor's `workflows[]` if ambiguous |
| Lint/test/build commands | From `/solo-npm:verify` skill (composes); fallback `pnpm run lint && pnpm test && pnpm run build` |

If a wrapper at `.claude/skills/release/SKILL.md` provides explicit
overrides (e.g., "the publish workflow is `release-monorepo.yml`"), use
those values instead of auto-detection.

## Phase 0.5 — Prompt-extraction validation (E1, E3 from v0.11.0)

After Phase 0 pre-fills slots, validate every extracted value against the canonical regex framework in `/unpublish` Phase 0.5 (canonical reference). For `/release`, the slots needing validation are:

- **`NEXT_VERSION` / version override** — semver regex `^\d+\.\d+\.\d+(-[A-Za-z0-9.-]+)?(\+[A-Za-z0-9.-]+)?$`. Reject pre-fills like `v999.999.999` (typos, non-semver) BEFORE Phase B's plan rendering.
- **Package name** in monorepo scope — npm name regex `^(@[a-z0-9][a-z0-9._-]*/)?[a-z0-9][a-z0-9._-]*$`. Reject if Phase 0 extracted a package name with shell metacharacters.

On validation failure, STOP with the diagnostic from `/unpublish` Phase 0.5 (slot + value + expected format).

## Phase 0.5b — Shell-safety hardening (Tier-4 #4 from v0.13.0)

After regex validation passes, apply the shell-safety check from `/unpublish` Phase 0.5b (canonical) to every extracted slot that flows into shell interpolation: `NEXT_VERSION` (passed to `git tag`, `git push`, `npm version`), package name (in monorepo subset). Reject high-risk shell metacharacters (`;`, `&`, `|`, backtick, `$`, parens, redirects, control chars). Skill body uses double-quoted interpolation at every command-construction site as the second line of defense.

## Phase A — Pre-flight

### A.0 Changed-only mode (monorepo, NEW v0.19.0)

If the user passed `--changed-only` (or `--since=<ref>`), this is a **monorepo-only filtered release**. Detect which workspace packages have source changes since the baseline ref, and operate only on that subset.

```bash
BASELINE_REF=${SINCE_REF:-$(git tag --list 'v*' --sort=-v:refname | head -1)}
CHANGED_PACKAGES=()
for pkg_dir in $(detect_workspace_packages); do
  if ! git diff --quiet "$BASELINE_REF" -- "$pkg_dir"; then
    PKG_NAME=$(node -p "require('./$pkg_dir/package.json').name")
    CHANGED_PACKAGES+=("$PKG_NAME")
  fi
done
```

If `CHANGED_PACKAGES` is empty, STOP with "No packages changed since $BASELINE_REF; nothing to release." This is an information signal, not an error — exit 0.

If non-empty, the rest of /release operates ONLY on `CHANGED_PACKAGES`:
- /verify Tier 1-8 runs against each changed package
- /public-api gate runs per changed package
- Tag scheme: per-package tags (`<pkg-name>-v<version>`) instead of root `v<version>`
- CHANGELOG: per-package CHANGELOG.md updates
- Plan rendering (Phase B): shows the change set explicitly so the user can confirm

If NOT in changed-only mode AND the repo is a monorepo, /release operates on ALL workspace packages (the existing behavior). Single-package repos ignore this flag.

### A.1 Working tree must be clean

```bash
git status --porcelain
```

If non-empty, **STOP**. Tell the user to commit or stash first. Do not
proceed.

### A.2 Run /solo-npm:verify

Invoke the `/solo-npm:verify` skill (or the project's `/verify` wrapper
if one exists). If neither is available, fall back to the project's
verification commands (detected from `package.json#scripts.lint`,
`package.json#scripts.test`, `package.json#scripts.build`).

As of v0.19.0, /verify runs Tiers 1-8 by default — including the new TS publish-gates: `/types` (attw), `/exports-check`, and `/smoke-test`. If any tier fails, **STOP** and surface the error.

### A.2b Public API breaking-change gate (NEW v0.19.0 — HARD GATE)

After /verify but BEFORE staging the version bump, run `/solo-npm:public-api`:

```bash
/solo-npm:public-api --strict
```

This:
1. Extracts the current dist/'s public API surface (function/class/interface/type/enum signatures)
2. Fetches the last released version's surface from the npm registry
3. Diffs and classifies: patch (no public-API change), minor (only added), major (any removed or signature-narrowed)
4. Compares against `NEXT_VERSION` (the user's requested bump)

If the classification > requested bump (e.g., user requested patch but diff is major), **STOP** with the precise version-bump suggestion. The user must either:
1. Bump version higher to match the diff (the recommended path)
2. Pass `--api-strict=false` to override (rare, e.g., when a "removed" symbol is provably unused by anyone)

The convention: **TS-library breaking changes always require a major bump.** /public-api enforces this by default.

For first releases (no baseline on npm), /public-api exits 0 with a "first release — no baseline" note. /release continues normally.

### A.3 Trust + environment check (cache-aware, content-aware as of v0.17.0)

For monorepos with many packages, running the full `--doctor` on every
release iterates ~3 npm calls per package — slow and mostly redundant
once trust is established. Phase A.3 uses a local cache at
`.solo-npm/state.json` to skip per-package work when nothing has
changed. As of v0.17.0 the cache is also **content-aware**: a change to
`.github/workflows/<RELEASE_WORKFLOW>` invalidates the cache even
within the time-TTL.

#### Step 1 — Read cache

Read `.solo-npm/state.json` if it exists. Compute:

- `filesystem` = current package list (from `pnpm-workspace.yaml`,
  `package.json#workspaces`, or root `package.json` for single packages)
- `newPackages = filesystem - cache.trust.configured` (set difference)
- `cacheAgeDays = now - cache.trust.lastFullCheck`
- `cacheStale = (cache missing) OR (cache.trust.lastFullCheck is null) OR (cacheAgeDays >= cache.trust.ttlDays)`

#### Step 2 — Branch on cache state

| Branch | Condition | Action |
|---|---|---|
| **Cold path** (full check) | `cacheStale` is true OR `cache.trust.workflowFileHash` is missing OR doesn't match the live workflow file's sha256 | Run `pnpm exec npm-trust --doctor --json --workflow <RELEASE_WORKFLOW>`. Apply the issue-code branching in Step 3 below. After success, **rewrite** `cache.trust.configured` with the current full list of `packages[]` where `trustConfigured == true OR provenancePresent == true`; refresh `lastFullCheck` to now; **store `workflowSnapshot.fileHash` to `cache.trust.workflowFileHash`** for content-aware invalidation. Save (H7 atomic). |
| **Targeted path** (delta only) | Cache fresh, hash matches, `newPackages` non-empty | Run `pnpm exec npm-trust --packages <newPackages...> --list --json`. Parse `ListReport.packages[]`; if any have `trustConfigured == false`, auto-invoke `/solo-npm:trust --packages <newPackages...>`. After trust returns success (exit 0 or 60-PARTIAL with handled failures), **append** `newPackages` to `cache.trust.configured`. Save. (Don't refresh `lastFullCheck` — the rest of the package set was not re-verified.) |
| **Hot path** (zero npm calls) | Cache fresh, hash matches, `newPackages` empty | Run a fast pre-flight that doesn't issue per-package npm calls: `pnpm exec npm-trust --validate-only --json --workflow <RELEASE_WORKFLOW>`. Returns a `ValidateReport`; if `ready === true` the env is good and we proceed to Phase B. If `ready === false`, surface `failures[]` and STOP per H6. |

**Forcing a full re-verify.** Delete `.solo-npm/state.json` (or set
`lastFullCheck` to null OR clear `workflowFileHash`) to force the cold
path on the next release — the explicit escape hatch for "I just
changed something on the npm registry side; re-verify everything."

**Content-aware invalidation rationale.** Time-TTL alone misses the case
where a maintainer edits `release.yml` (e.g., switches `release.yml` →
`release-monorepo.yml`, or changes `id-token: write` → token-auth).
The new workflow filename / permissions invalidate the trust binding
immediately on the npm registry side. Comparing the live file's sha256
against the cached hash catches that drift before the publish attempt
fails with a confusing 403.

#### Step 3 — Issue-code branching (cold path only)

When the doctor JSON is parsed, apply this branching. **Auto-fix when
the next action is unambiguous; do NOT pause to offer alternative
invocation paths.**

**Auto-fix (no user prompt):**

| Detected | Auto-action |
|---|---|
| `WORKFLOWS_NONE` | Auto-invoke `/solo-npm:init` to scaffold `release.yml`. After init completes, re-run A.3 (now with workflows in place). |
| Any package with `trustConfigured == false` AND `provenancePresent == false` | Auto-invoke `/solo-npm:trust`. After trust completes, re-run A.3. |

**Auto-chain with one approval gate:**

| Detected | Action |
|---|---|
| `PACKAGE_NOT_PUBLISHED` for any package | First publish must happen before tag-triggered release flow can run. AskUserQuestion: *"`<N>` package(s) need a first publish before `/release` can run. Bootstrap them now via `/solo-npm:init` (which has a guided publish flow)?"* — Options: `Yes — chain to /solo-npm:init` (Recommended) / `Abort`. On Yes: chain into `/solo-npm:init` (which routes through Phase 2 guided publish, then Phase 3 trust, then Phase 4 done). After init returns, re-run /release Phase A.3. |

**Hard stops (no auto-fix possible; surface to user and end):**

| Detected | Action |
|---|---|
| `summary.fail > 0` (any other failure) | STOP. Surface the failing issue verbatim. |
| `WORKSPACE_NOT_DETECTED`, `REPO_NO_REMOTE` | STOP. The user must fix the repo structure. |
| `REGISTRY_PROVENANCE_CONFLICT` | STOP. Surface the remedy: either remove `provenance: true` from `publishConfig` or change `registry` back to public npm. |

**Informational (continue silently):**

| Detected | Action |
|---|---|
| `AUTH_NOT_LOGGED_IN` | Ignore. Tag-triggered CI publishes via OIDC; local auth doesn't matter for release. |
| `PACKAGE_TRUST_DISCREPANCY` | Ignore. Registry has provenance even when `npm trust list` is empty. |
| Any other `warn`-severity issue | Surface as a one-line note, but proceed. |

Phase A passes silently for the typical case (everything green) or
after auto-fix completes. Move to Phase A.5.

### A.5 Cache-aware audit check

Read `.solo-npm/state.json#audit`. Branch:

| Condition | Action |
|---|---|
| Cache fresh (`now - audit.lastFullScan < audit.ttlDays`) AND `audit.tier1Count == 0` | Skip — proceed to Phase B. |
| Cache fresh AND `audit.tier1Count > 0` | **STOP**. Surface: *"Your last security audit (run X days ago) found <N> Tier-1 vulnerabilities. Run `/solo-npm:audit` to review, then `/solo-npm:deps` to fix before releasing. (To bypass: `rm .solo-npm/state.json` forces re-scan on next release; or run `/solo-npm:audit` to verify and re-cache.)"* |
| Cache stale OR missing | Skip — don't block release on no-info. The next `/solo-npm:audit` invocation populates the cache. |

This gate catches "we have a known critical CVE" before publishing without ever running a fresh audit. Zero-latency on the common path; explicit STOP only when there's an actionable gap.

Move to Phase B.

## Phase B — Plan and confirm

### B.1 Find the latest tag

```bash
git tag --list 'v*' --sort=-v:refname | head -1
```

Call this `LATEST_TAG`. If empty → **first release**: render the changelog draft + plan summary as visible chat output, then ask the user via `AskUserQuestion`:

- header: `"First release"`
- options: `0.1.0` / `1.0.0` / `Other (specify)` / `Abort`

If the user picks a version: set `NEXT_VERSION = picked`. Plan summary already rendered above. **Skip B.5 entirely** — the user already approved the version + saw the changelog. Proceed directly to Phase C.

### B.2 Detect version mode + auto-chain to /solo-npm:prerelease

Parse `package.json#version` (or the main package's `package.json#version` for monorepos). Two modes:

- **Stable** — current version matches `^\d+\.\d+\.\d+$`
- **Pre-release** — current version matches `^\d+\.\d+\.\d+-[a-z]+\.\d+$`

**If pre-release: AUTO-INVOKE `/solo-npm:prerelease`.** The user typed `/release` but the version state means they're continuing a pre-release line. The prerelease skill handles bump / promote / abort. End this skill's flow once prerelease returns control.

If stable: continue to B.3. (B.5's options become only the stable-mode prompt; pre-release options have moved to /solo-npm:prerelease.)

### B.3 Collect commits since `LATEST_TAG` (with strictness checks, Tier-3 F+K, v0.12.0)

```bash
# Tier-3 K: cap at 500 commits to keep parsing bounded
git log "${LATEST_TAG}..HEAD" --no-merges --max-count=500 --format='%H %s%n%b%n---'

# After the call, count how many commits exist in the range:
TOTAL_COMMITS=$(git rev-list --count --no-merges "${LATEST_TAG}..HEAD")
if [ "$TOTAL_COMMITS" -gt 500 ]; then
  echo "WARN: $TOTAL_COMMITS commits since ${LATEST_TAG}; showing first 500."
  echo "      Review the CHANGELOG carefully before approving — auto-bump may miss types"
  echo "      from the truncated tail (rare, but possible if breaking changes are clustered late)."
  echo "      Consider releasing more frequently."
fi
```

Parse each line as a conventional commit: `type(scope)?(!)?: subject`.
Drop commits that don't match the convention (typically merges).

**Type whitelist (Tier-3 F)**: validate the type field against the canonical conventional-commits whitelist. Unrecognized types get a "did you mean" hint:

```
TYPE_WHITELIST = feat fix chore docs style refactor perf test build ci revert

For each parsed commit:
  if type not in TYPE_WHITELIST:
    suggest = closest_levenshtein_match(type, TYPE_WHITELIST)
    surface "WARN: commit <SHA> uses unknown type '<type>'. Did you mean '<suggest>'? Treating as Other."

Special case — `breaking:` (non-standard but common typo for `feat!:`):
  if type == "breaking":
    surface "WARN: commit <SHA> uses 'breaking:' which is NOT canonical conventional-commits."
    surface "      The canonical breaking-change indicators are 'feat!:', 'fix!:', etc., or a 'BREAKING CHANGE:' footer."
    surface "      Treating as Other (NOT auto-promoting to major bump). If you intended major, amend the commit"
    surface "      to 'feat!:' or add a 'BREAKING CHANGE:' footer + re-run /release."

Subject length:
  if len(subject) > 72:
    surface "WARN: commit <SHA> subject is N chars (>72 conventional-commits guideline). Bump still derives correctly; just style note."
```

Group:

- **Breaking** — type ends with `!` OR body contains `BREAKING CHANGE:`
- **Features** — `feat`
- **Fixes** — `fix`
- **Performance** — `perf`
- **Reverts** — `revert`
- **Other** — `chore`, `docs`, `test`, `build`, `ci`, `style`, `refactor` (informational only)
- **Unknown** — anything else (surfaced as warning, treated as Other for bump derivation)

### B.4 Decide the bump (stable mode only)

Highest applicable wins:

| Condition | Bump |
|---|---|
| Any Breaking commit | major |
| Any Features commit (no Breaking) | minor |
| Any Fix / Performance / Revert commit (no Features / Breaking) | patch |
| Only Other commits | **STOP** — nothing to release |

Call the result `AUTO_BUMP_VERSION`.

If Phase 0's prompt-context extraction pre-filled `NEXT_VERSION`:
- If `NEXT_VERSION == AUTO_BUMP_VERSION`: silent agreement; proceed to B.5 with `NEXT_VERSION`.
- If `NEXT_VERSION != AUTO_BUMP_VERSION`: render BOTH in the plan summary and B.5 offers a 3-way choice.

Otherwise: `NEXT_VERSION = AUTO_BUMP_VERSION`.

### B.5 Render summary + AskUserQuestion (stable mode only)

Render this summary block to the chat (so it stays visible):

```
Release plan:
  Version       v{LATEST} → v{NEXT}  ({bump} — {N feat, M fix, ...})
  Commits       {N} since {LATEST_TAG}
                  {type}: {subject} ({hash})
                  ...
  Trust         ✓ already configured (provenance present)
                (or: "✗ skipped (custom registry)")
  CHANGELOG     {first 4 lines visible; "expand" to show full draft}
```

Then call the `AskUserQuestion` tool. Two cases:

**Default** — auto-bump matches user's intent (or no version named in prompt):

```
header:   "Release"
question: "Approve the release plan above?"
options:  Proceed with v{NEXT} / Abort
```

**Override variant** — Phase 0 named a version different from `AUTO_BUMP_VERSION`:

```
header:   "Release"
question: "Auto-bump suggests v{AUTO_BUMP}; you said v{NAMED}. Which?"
options:
  1. Proceed with v{NAMED} (your override)
  2. Proceed with v{AUTO_BUMP} (auto-bump from commits)
  3. Abort
```

That's it. **No `Override version` (free-form) and no `Edit changelog`** — both are friction. Power users who need them follow the "Manual overrides" guidance at the top of this skill.

On Proceed: continue to Phase C. On Abort: no state changes; end.

### B.6 Render the CHANGELOG draft

Prepend a section to `CHANGELOG.md` (compute, do NOT write yet):

```markdown
## [v{NEXT}](https://github.com/<REPO_SLUG>/compare/v{LATEST}...v{NEXT}) (YYYY-MM-DD)

### Breaking Changes

- subject ([hash](https://github.com/<REPO_SLUG>/commit/hash))

### Features

- subject ([hash](https://github.com/<REPO_SLUG>/commit/hash))

### Bug Fixes

- subject ([hash](https://github.com/<REPO_SLUG>/commit/hash))

### Performance

- subject ([hash](https://github.com/<REPO_SLUG>/commit/hash))
```

Only include sections with entries.

## Phase C.0 — Error-handling patterns (H2, H3, H4, H6 from `/unpublish` reference)

Before kicking off the execute steps, apply the standard solo-npm error patterns. Canonical wording lives in `/unpublish` Phases C.0–D.2; concrete adaptation per pattern:

- **H2 — `.solo-npm/state.json` corruption guard + atomic writes**: every `JSON.parse(state.json)` read in Phase A.3 (trust cache), A.5 (audit cache), and the post-publish reads in C.7.6 (bundle-size cache) must wrap in try/catch. On parse fail surface non-fatal warning *".solo-npm/state.json is malformed; treating as empty cache. Remove the file and re-run any solo-npm skill to regenerate."* Continue with empty defaults; do not block the release on a stale cache. **Writes must be atomic** per H7 / `/unpublish` Phase −1.4: write to `.solo-npm/state.json.tmp` then `fs.renameSync()`. Non-atomic `writeFileSync` truncates first, so a process killed mid-write leaves a corrupted cache. The C.7.6 cache update implements this directly.
- **H3 — Auth-window race**: `npm publish` runs in CI via OIDC (no local auth window), so this is mostly a no-op for the publish step itself. However, Phase G's chain to `/solo-npm:deprecate` invokes local `npm deprecate`; that chain target inherits its own Phase C.0 H3 / H1 / H5 handlers (`npm whoami` re-check + lock acquisition + EOTP detection), so H3 is satisfied transitively. Don't duplicate the check here.
- **H4 — Registry propagation lag retry**: Phase C.7 verify (`npm view <pkg>@<v>` post-publish, `npm view <pkg> dist-tags`) uses 3 attempts × 5s sleep before declaring inconsistency. Don't HARD STOP the release on lag — surface non-fatal note: *"Registry not yet reflecting publish after 15s — npm CDN may take up to 5 minutes; tag is on origin and CI succeeded, so this is verification lag, not a publish failure."*
- **H6 — Chain-target failure recovery**: chains into `/verify` (A.2, C.4), `/init` (A.3 WORKFLOWS_NONE auto-fix), `/trust` (A.3 delta), `/audit` (A.5 cached check), `/deprecate` (Phase G) — when any chain target STOPs, capture its verbatim diagnostic and surface in `/release` context with options: retry the chain / abort / skip-and-continue (where safe). Don't silently swallow.

## Phase C.0a — H5 concurrent-release lock (v0.11.0)

Two parallel `/release` runs on the same repo race the git tag and can double-publish. Acquire a per-repo lock immediately at Phase C start, before any git/npm mutation. Mirrors `/unpublish` Phase −1.8 (canonical) with a per-repo (not per-package) key since `/release` operates on the whole repo:

```bash
REPO_LOCK=".solo-npm/locks/$(git rev-parse --show-toplevel | sed 's|/|_|g').lock"
if [ -f "$REPO_LOCK" ]; then
  STALE_PID=$(cat "$REPO_LOCK" 2>/dev/null)
  if [ -n "$STALE_PID" ] && kill -0 "$STALE_PID" 2>/dev/null; then
    echo "ERROR: another solo-npm release is in flight on this repo (PID $STALE_PID)."
    echo "       Wait for it to finish, or kill PID $STALE_PID if hung."
    exit 1
  fi
  [ -n "$STALE_PID" ] && echo "WARN: removing stale repo lockfile $REPO_LOCK (PID $STALE_PID dead)"
  rm -f "$REPO_LOCK"
fi
mkdir -p .solo-npm/locks
echo $$ > "$REPO_LOCK"
trap 'rm -f "$REPO_LOCK"' EXIT
```

This complements (does NOT replace) the per-package locks acquired by `/deprecate`/`/dist-tag`/`/owner`/`/unpublish` — `/release` is one-repo-at-a-time but multi-package operations remain free to run concurrently.

## Phase C — Execute

After approval at B.5, run all of the following without further user
interaction. Halt on the first failure.

### C.1 Apply CHANGELOG and version

1. Prepend the rendered CHANGELOG entry to `CHANGELOG.md`.
2. Update `package.json#version` to `NEXT_VERSION`.
3. **(Monorepo only)** Update each `packages/*/package.json#version` to
   `NEXT_VERSION`.

### C.2 Commit — with pre-commit hook failure rollback (v0.11.0)

```bash
git add CHANGELOG.md package.json
# (Monorepo: also `git add packages/*/package.json`)

COMMIT_OUT=$(git commit -m "chore: release v${NEXT_VERSION}" 2>&1)
COMMIT_RC=$?
if [ $COMMIT_RC -ne 0 ]; then
  # Detect pre-commit hook rejection (vs other commit failures)
  if echo "$COMMIT_OUT" | grep -qiE "pre-commit hook|hook .* failed|hook declined"; then
    echo "ERROR: pre-commit hook blocked the release commit."
    echo "Hook output:"
    echo "$COMMIT_OUT" | sed 's/^/  /'
    echo
    echo "Working tree state: files are STAGED but NOT committed."
    echo
    echo "Recovery options:"
    echo "  a) Address the hook complaint (e.g., format/lint), then re-run /solo-npm:release."
    echo "     The skill is idempotent — it'll detect the un-committed CHANGELOG/package.json"
    echo "     and resume from C.2."
    echo "  b) Bypass the hook (emergency only): git commit --no-verify -m 'chore: release v${NEXT_VERSION}'"
    echo "     Then re-run /solo-npm:release to resume from C.3."
    echo "  c) Reset the staging area entirely: git reset HEAD -- CHANGELOG.md package.json"
    echo "     This rolls back C.1's CHANGELOG/version mutations to the staging area."
    exit 1
  fi
  # Other commit failures (rare — empty commit, etc.)
  echo "ERROR: git commit failed (exit $COMMIT_RC):"
  echo "$COMMIT_OUT" | sed 's/^/  /'
  exit 1
fi
```

### C.3 Push (commit only, no tag yet) — with rejection categorization

```bash
PUSH_OUT=$(timeout 60 git push 2>&1)
PUSH_RC=$?
if [ $PUSH_RC -ne 0 ]; then
  case "$PUSH_OUT" in
    *"non-fast-forward"*|*"Updates were rejected"*|*"fetch first"*)
      echo "ERROR: git push rejected (non-fast-forward). Someone else pushed since you last fetched."
      echo "Recovery: git fetch origin && git rebase origin/main"
      echo "         then re-run /solo-npm:release."
      ;;
    *"pre-receive hook"*|*"hook declined"*)
      echo "ERROR: git push rejected by server-side hook (pre-receive)."
      echo "Hook output:"
      echo "$PUSH_OUT" | sed 's/^/  /'
      echo "Recovery: address the hook's complaint (server-side, set by repo admin), amend the commit, then re-run /solo-npm:release."
      ;;
    *"pre-push hook"*)
      echo "ERROR: git push rejected by client-side pre-push hook (Tier-3 L, v0.12.0)."
      echo "Hook output:"
      echo "$PUSH_OUT" | sed 's/^/  /'
      echo "Recovery options:"
      echo "  a) Address the hook's complaint (e.g., test failure in pre-push hook), then re-run /solo-npm:release."
      echo "  b) Bypass the hook (emergency only): git push --no-verify"
      echo "     Then re-run /solo-npm:release to resume from C.5 tagging."
      echo "  c) Disable the hook for this repo: rm .git/hooks/pre-push  (or rename it)"
      ;;
    *"protected branch"*|*"branch protection"*|*"required status check"*)
      echo "ERROR: git push rejected by branch protection rules."
      echo "Recovery: this branch may not allow direct pushes. Push to a feature branch + open a PR,"
      echo "         or update the branch-protection rules if you have admin access."
      ;;
    *"Permission denied"*|*"Authorization failed"*|*"could not read Username"*|*"403"*)
      echo "ERROR: git push auth failed."
      echo "Recovery: check SSH key, credential helper, or token expiry."
      echo "         For HTTPS: re-run 'git config credential.helper store' or refresh the token."
      echo "         For SSH:   ssh-add -l   to verify your key is loaded."
      ;;
    *)
      echo "ERROR: git push failed (exit $PUSH_RC):"
      echo "$PUSH_OUT" | sed 's/^/  /'
      ;;
  esac
  # SSL/TLS error remediation per /unpublish Phase −1.10 (v0.12.0):
  # if PUSH_OUT matches SSL_ERROR / cert / unable to get local issuer / TLSV1_ALERT,
  # surface the four-option remediation block (transient retry / corp proxy / OS CA bundle / --insecure).
  ssl_remediation_if_applicable "$PUSH_OUT" || true
  exit 1
fi
```

### C.4 Final pre-tag verification

Re-invoke `/solo-npm:verify` (or `/verify` wrapper) against the bumped
state. Aborts here are rare but not hypothetical (e.g., a test that
depends on `package.json#version`).

If anything fails, **STOP**. Recovery: fix the issue, restart from
Phase A. The release commit is on origin but no tag has been pushed
yet, so the workflow won't run.

### C.5 Tag and push the tag — with collision pre-flight

**Pre-flight: tag collision check.** Before creating the tag locally, verify it doesn't already exist on the remote. The wrong-name re-release case (where the user just ran `/solo-npm:unpublish` and is republishing) and the manual-tag case both fire here.

```bash
# Check remote for the tag BEFORE local tag creation
if timeout 30 git ls-remote --tags origin "refs/tags/v${NEXT_VERSION}" 2>/dev/null | grep -q .; then
  echo "ERROR: tag v${NEXT_VERSION} already exists on origin."
  echo "       This usually means a prior /solo-npm:release got partway then failed,"
  echo "       OR someone created the tag manually,"
  echo "       OR /solo-npm:unpublish was run but the git tag wasn't cleaned up."
  echo
  echo "Investigate: git fetch --tags && git show v${NEXT_VERSION}"
  echo
  echo "If safe to overwrite (e.g., post-unpublish re-release):"
  echo "  git tag -d v${NEXT_VERSION}                            # remove local tag if any"
  echo "  git push origin :refs/tags/v${NEXT_VERSION}            # remove remote tag"
  echo "  /solo-npm:release                                      # retry"
  exit 1
fi

# Tag doesn't exist on remote — safe to create locally and push
git tag "v${NEXT_VERSION}"

# Push tag with the same rejection categorization as C.3
PUSH_OUT=$(timeout 60 git push --tags 2>&1)
PUSH_RC=$?
if [ $PUSH_RC -ne 0 ]; then
  # If the tag push fails, also clean up the local tag — otherwise the user
  # is left with a local-only tag that will conflict with the next /release attempt.
  git tag -d "v${NEXT_VERSION}" 2>/dev/null
  case "$PUSH_OUT" in
    *"already exists"*)
      echo "ERROR: race condition — tag v${NEXT_VERSION} appeared on origin between pre-flight and push."
      echo "Recovery: investigate as above; another release may have raced this one."
      ;;
    *"Permission denied"*|*"Authorization failed"*|*"403"*)
      echo "ERROR: git push --tags auth failed (commit pushed in C.3 but tag push rejected)."
      echo "Recovery: re-authenticate, then 'git tag v${NEXT_VERSION} && git push --tags'."
      ;;
    *"hook"*)
      echo "ERROR: tag push rejected by server-side hook:"
      echo "$PUSH_OUT" | sed 's/^/  /'
      ;;
    *)
      echo "ERROR: git push --tags failed (exit $PUSH_RC):"
      echo "$PUSH_OUT" | sed 's/^/  /'
      ;;
  esac
  exit 1
fi
```

This triggers the publish workflow on the new tag.

### C.6 Watch CI (with timeout cap, v0.11.0)

Skip this step entirely if `GH_AVAILABLE=0` (per Phase A.0 detection): surface "skipping CI watch — gh unavailable; check the workflow run manually at <repo-url>/actions" and continue to C.7.

```bash
timeout 1800 gh run watch --exit-status
```

`timeout 1800` (30min) bounds the wait against indefinite-hang scenarios (long CI, cancelled run that gh doesn't notice, network drop). gh's own default is also 30min but the explicit `timeout` makes the cap explicit + ours-not-gh's, and the surrounding error message becomes consistent regardless of how the timeout fires.

If CI fails, **STOP** and surface the error. The tag is on origin
(intentional record of intent); recovery is `gh run rerun <id>` after
fixing the cause. **Do not** bump the version unless the tarball needs
new content.

If the timeout fires (CI exceeded 30min), surface clearly:

```
WARN: gh run watch timed out after 30 minutes.
      The workflow may still be running — check at <repo-url>/actions
      Once CI completes, re-run /solo-npm:release to resume from C.7 verification.
      The release tag is already on origin; CI will continue independently.
```

**On CI success: refresh the trust-state cache.** A successful CI
publish via OIDC is proof that trust still works for every package
that just published. Update `.solo-npm/state.json`:

- Set `cache.trust.lastFullCheck` to `now` (extends the cache TTL).
- Append any newly-published packages to `cache.trust.configured` if
  not already present.

This means the next `/release` (within `ttlDays`, with no new packages
added) takes the **hot path** — zero npm calls for the trust check.

### C.7 Verify on the registry

Read `package.json#publishConfig.registry`.

**If unset or `https://registry.npmjs.org/`** (default public npm):

```bash
npm view "<PACKAGE_NAME>@${NEXT_VERSION}" version
npm view "<PACKAGE_NAME>@${NEXT_VERSION}" dist.attestations
```

The second should show `provenance: { predicateType:
"https://slsa.dev/provenance/v1" }`.

**If custom registry**:

```bash
npm view "<PACKAGE_NAME>@${NEXT_VERSION}" version --registry $REG
```

Skip the dist.attestations check (provenance only works on public npm).

**Monorepo**: iterate per package — read each `packages/*/package.json`'s
`name` field and run the appropriate `npm view` for each:

```bash
for pkg_json in packages/*/package.json; do
  pkg_name=$(node -p "require('./${pkg_json}').name")
  npm view "${pkg_name}@${NEXT_VERSION}" version
  npm view "${pkg_name}@${NEXT_VERSION}" dist.attestations  # if public registry
done
```

Skip any packages with `"private": true` in their `package.json`.

### C.7.5 Create GitHub Release with notes

After registry verify confirms the package is live, create a rich GitHub Release page so anyone watching the repo gets a notification with the just-published changelog content.

**Pre-flight**: distinguish "not installed" from "not authenticated":

```bash
if ! command -v gh >/dev/null 2>&1; then
  GH_STATE="not_installed"
elif ! gh auth status >/dev/null 2>&1; then
  GH_STATE="not_authenticated"
else
  GH_STATE="ready"
fi
```

If `not_installed` → **skip with**:

> ⚠ `gh` not installed; skipping GitHub Release creation. The git tag is pushed and the npm publish succeeded, but the GitHub Release page will be empty.
> Install from https://cli.github.com (covers macOS / Linux / Windows / other platforms).
> After install + `gh auth login`, manually create with `gh release create v${NEXT_VERSION}` if desired.

If `not_authenticated` → **skip with**:

> ⚠ `gh` installed but not authenticated; skipping GitHub Release creation. Run `gh auth login` then `gh release create v${NEXT_VERSION}` to create manually.

Either skip is **non-fatal** — release flow continues to C.8 normally.

If `ready` → proceed:

If `gh` is available:

```bash
# Extract the just-prepended CHANGELOG entry as the release notes body.
# Match from "## v<NEXT_VERSION>" up to (but not including) the next "## v" header.
NOTES=$(awk -v v="${NEXT_VERSION}" '
  $0 ~ "^## v" v "( |$)" { capture=1; next }
  capture && /^## v/ { exit }
  capture { print }
' CHANGELOG.md)

# Create the GitHub Release — warn-don't-fail on error (B5, v0.11.0)
RELEASE_OUT=$(timeout 30 gh release create "v${NEXT_VERSION}" \
  --title "v${NEXT_VERSION}" \
  --notes "${NOTES}" 2>&1)
RELEASE_RC=$?
if [ $RELEASE_RC -ne 0 ]; then
  case "$RELEASE_OUT" in
    *"already exists"*)
      echo "WARN: GitHub Release v${NEXT_VERSION} already exists (someone created it manually or a prior /release made it)."
      echo "      The npm publish succeeded; release page is just an artifact. No action needed."
      ;;
    *"403"*|*"Permission denied"*|*"not authorized"*)
      echo "WARN: gh release create failed (auth/permission): $RELEASE_OUT"
      echo "      The npm publish succeeded; create the release page manually:"
      echo "      gh auth refresh -s repo && gh release create v${NEXT_VERSION} --notes-from-tag"
      ;;
    *"422"*|*"unprocessable"*)
      echo "WARN: gh release create rejected the request: $RELEASE_OUT"
      echo "      The npm publish succeeded. Investigate via: gh release view v${NEXT_VERSION}"
      ;;
    *)
      echo "WARN: gh release create failed (exit $RELEASE_RC):"
      echo "$RELEASE_OUT" | sed 's/^/  /'
      echo "      The npm publish succeeded; the release page is secondary. Retry manually:"
      echo "      gh release create v${NEXT_VERSION} --notes-from-tag"
      ;;
  esac
  # Don't exit non-zero — npm publish already happened. Continue to C.7.6.
fi
```

The CHANGELOG extractor strips the version header line itself (the `gh release create --title` covers that) and stops at the next `## v...` header. If extraction returns empty (e.g., CHANGELOG format drift), fall back to passing the version-only title with no body and surface a warning.

**B5 rationale (v0.11.0)**: previously, `gh release create` failure halted the release. But the npm publish already succeeded — the release page is a secondary artifact. Demoting to warn-don't-fail means the user can manually create the release later (`gh release create v$V --notes-from-tag`) without re-publishing or rolling back.

### C.7.6 Update bundle-size baseline cache

After the publish succeeds, write the just-shipped version's `unpackedSize` to `.solo-npm/state.json#pkgCheck.lastSize` so the next `/verify` Step 5 Tier 4 has a baseline to compare against.

**Retention policy**: keep at most the **last 3 versions per package** (sorted by semver descending). Older entries are pruned as new ones land — the cache stays bounded over years of releases. Tier 4 only needs the most-recent prior version for delta computation, so 3 is sufficient depth.

**As of v0.17.0**, the size is read from a single bulk
`--verify-provenance --json` call (which already runs in Phase C.7 to
verify the SLSA attestation). For a monorepo, this is N-1 fewer npm
calls than the previous per-package `npm view dist.unpackedSize`
loop:

```bash
# unpackedSize comes from VerifyProvenanceReport.packages[].unpackedSize
# (added in npm-trust 0.11.0 / VerifyProvenanceReport schemaVersion 2).
# The verify report was already fetched in C.7 — reuse the parsed JSON
# instead of issuing a fresh npm view call.
SIZE=$(echo "$VERIFY_REPORT_JSON" | jq -r ".packages[] | select(.pkg == \"${PACKAGE_NAME}\") | .unpackedSize")
# Fallback for older CLIs that don't expose unpackedSize:
if [ -z "$SIZE" ] || [ "$SIZE" = "null" ]; then
  SIZE=$(npm view "${PACKAGE_NAME}@${NEXT_VERSION}" dist.unpackedSize)
fi

# Write to .solo-npm/state.json#pkgCheck.lastSize, then trim to last 3 versions per package:
node -e "
  const fs = require('fs');
  const semver = (a, b) => {
    const aP = a.split('.').map(Number);
    const bP = b.split('.').map(Number);
    for (let i = 0; i < 3; i++) if (aP[i] !== bP[i]) return bP[i] - aP[i];
    return 0;
  };
  const path = '.solo-npm/state.json';
  const state = JSON.parse(fs.readFileSync(path, 'utf8'));
  state.pkgCheck = state.pkgCheck || {};
  state.pkgCheck.lastSize = state.pkgCheck.lastSize || {};
  state.pkgCheck.lastSize['${PACKAGE_NAME}@${NEXT_VERSION}'] = ${SIZE};
  // Trim: keep last 3 versions per package
  const sameNameKeys = Object.keys(state.pkgCheck.lastSize)
    .filter(k => k.startsWith('${PACKAGE_NAME}@'))
    .sort((a, b) => semver(a.split('@').pop().split('-')[0], b.split('@').pop().split('-')[0]));
  for (const stale of sameNameKeys.slice(3)) delete state.pkgCheck.lastSize[stale];
  // Atomic write per H7 (Phase −1.4 of /unpublish) — write to .tmp then rename.
  // Non-atomic writeFileSync truncates first; a process killed mid-write would corrupt the cache.
  const tmp = path + '.tmp';
  fs.writeFileSync(tmp, JSON.stringify(state, null, 2) + '\n');
  fs.renameSync(tmp, path);
"
```

For monorepos: iterate per package, write per `<pkg>@<version>` entry, trim per package.

This cache feeds the next /verify's Tier 4 bundle-size regression check. First-run packages have no baseline; the cache builds organically over releases. The 3-version cap keeps `state.json` bounded.

### C.8 Final notification

Print:

```
Released v${NEXT_VERSION} to npm.
  Tarball: https://www.npmjs.com/package/<PACKAGE_NAME>/v/${NEXT_VERSION}
  CI:      <gh run url>
```

Then proceed to Phase G if this was a major version bump; otherwise end the skill.

## Phase G — Post-major deprecation chain (gated)

Only fires when this release was a **major version bump** — detected by comparing `MAJOR(NEXT_VERSION)` to `MAJOR(LATEST_TAG_AT_PHASE_A)`. Patch and minor releases skip Phase G entirely (jump straight to skill end after C.8).

After a successful major release (e.g., shipped `2.0.0` from a `1.x` line), the previous major is typically EOL. Surface an `AskUserQuestion` gate to deprecate it cleanly:

```
Header:   "Deprecate previous major"
Question: "Deprecate the v<PREV_MAJOR>.x line now?"
Options:
  - Yes — chain to /solo-npm:deprecate with default message "v<PREV_MAJOR>.x is EOL — migrate to v<NEXT_MAJOR>.0+" (Recommended)
  - Yes — let me customize the message — chain to /solo-npm:deprecate with prompt for the message
  - No — defer (I'll handle later via /solo-npm:deprecate)
```

On either Yes: invoke `/solo-npm:deprecate` with `RANGE = <PREV_MAJOR>.x` and the chosen `MESSAGE`. The deprecate skill runs its own Phase A → D and returns. The user gets two confirmations (Phase G gate + deprecate's Phase B gate) but that's deliberate — deprecation is destructive on registry state.

On `No — defer`: end the release skill with a footer hint:

> Released v<NEXT_VERSION>. To deprecate the v<PREV_MAJOR>.x line later: `/solo-npm:deprecate "v<PREV_MAJOR>.x is EOL — migrate to v<NEXT_MAJOR>.0+"`

## Failure modes and recovery

| Failure | Where | Recovery |
|---|---|---|
| Working tree dirty | A.1 | User commits or stashes; restart from A.1 |
| `/solo-npm:verify` fails | A.2, C.4 | Fix the issue; restart from A.1 |
| Doctor `summary.fail > 0` | A.3 | Fix the underlying environment issue (e.g., upgrade Node); restart |
| Network failure on doctor | A.3 | Retry after restoring network; restart from A.1 |
| User says `Abort` | B.5 | No state changes; end |
| Commit fails | C.2 | Inspect, fix, restart from A.1 (no commit was made) |
| Push fails (network or auth) | C.3 | Retry the push manually; if persistent, abort and clean up the local commit |
| Tag already exists | C.5 | **STOP** — someone else released this version; investigate before forcing |
| `git push --tags` fails | C.5 | Retry; check origin remote |
| CI fails | C.6 | `gh run rerun <id>` after fixing; do not bump version unless tarball needs new content |
| `npm view` lags showing the new version | C.7 | Retry after a minute; the registry takes a moment to propagate |

## What this skill does NOT do

- Auto-rerun CI on failure (most failures need human investigation).
- Auto-fallback to classic publish on CI failure (would lose provenance attestation).
- Auto-create `release.yml` or bootstrap trust on first run — those are
  the `/solo-npm:init` skill's job (which composes with `/solo-npm:trust`
  for the npm-side OIDC config); this skill assumes the env is already
  provisioned.
- Push branches other than `main` — assumes you're on the canonical
  release branch.
- Auto-merge PRs — PRs are disabled in this workflow.
