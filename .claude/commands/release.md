---
description: Universal release entry point — tag-triggered npm release with OIDC provenance and one approval gate. Triggers from prompts like "ship it", "release this", "cut a release", "time to release v0.5.4", "push to npm", "get on npm". Auto-chains to /solo-npm:prerelease on pre-release version. Three phases (pre-flight, plan, execute). Wrap via .claude/skills/release/SKILL.md for repo-specific narrative.
---

# Release

> **This is the opinionated baseline.** Consumer repos typically wrap it
> via `.claude/skills/release/SKILL.md` to add repo-specific narrative
> (workspace shape, prepare-dist usage, custom verification commands).
> Invoke `/solo-npm:release` directly only when you want the unmodified
> baseline.

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

## Phase A — Pre-flight

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

If verification fails, **STOP** and surface the error.

### A.3 Trust + environment check (cache-aware)

For monorepos with many packages, running the full `--doctor` on every
release iterates ~3 npm calls per package — slow and mostly redundant
once trust is established. Phase A.3 uses a local cache at
`.solo-npm/state.json` to skip per-package work when nothing has
changed.

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
| **Cold path** (full check) | `cacheStale` is true | Run `pnpm exec npm-trust --doctor --json --workflow <RELEASE_WORKFLOW>`. Apply the issue-code branching in Step 3 below. After success, **rewrite** `cache.trust.configured` with the current full list of `packages[]` where `trustConfigured == true OR provenancePresent == true`; refresh `lastFullCheck` to now. Save. |
| **Targeted path** (delta only) | Cache fresh, `newPackages` non-empty | Run `pnpm exec npm-trust --packages <newPackages...> --list --json`. If any returned packages need config, auto-invoke `/solo-npm:trust --packages <newPackages...>`. After trust returns success, **append** `newPackages` to `cache.trust.configured`. Save. (Don't refresh `lastFullCheck` — the rest of the package set was not re-verified.) |
| **Hot path** (zero npm calls) | Cache fresh, `newPackages` empty | Skip the trust check. Run inline env checks instead: `node -v` returns ≥24, `npm -v` returns ≥11.5.1, `git remote get-url origin` succeeds, `.github/workflows/<RELEASE_WORKFLOW>` exists. Proceed to Phase B. |

**Forcing a full re-verify.** Delete `.solo-npm/state.json` (or set
`lastFullCheck` to null) to force the cold path on the next release —
the explicit escape hatch for "I just changed something on the npm
registry side; re-verify everything."

#### Step 3 — Issue-code branching (cold path only)

When the doctor JSON is parsed, apply this branching. **Auto-fix when
the next action is unambiguous; do NOT pause to offer alternative
invocation paths.**

**Auto-fix (no user prompt):**

| Detected | Auto-action |
|---|---|
| `WORKFLOWS_NONE` | Auto-invoke `/solo-npm:init` to scaffold `release.yml`. After init completes, re-run A.3 (now with workflows in place). |
| Any package with `trustConfigured == false` AND `provenancePresent == false` | Auto-invoke `/solo-npm:trust`. After trust completes, re-run A.3. |

**Hard stops (no auto-fix possible; surface to user and end):**

| Detected | Action |
|---|---|
| `summary.fail > 0` (any other failure) | STOP. Surface the failing issue verbatim. |
| `WORKSPACE_NOT_DETECTED`, `REPO_NO_REMOTE` | STOP. The user must fix the repo structure. |
| `PACKAGE_NOT_PUBLISHED` for any package | STOP. First publish must be manual: surface the package list and instruct the user — `npm publish --provenance=false --access public` once per package, then re-run `/release`. |
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

### B.3 Collect commits since `LATEST_TAG`

```bash
git log "${LATEST_TAG}..HEAD" --format='%H %s'
```

Parse each line as a conventional commit: `type(scope)?(!)?: subject`.
Drop commits that don't match the convention (typically merges).

Group:

- **Breaking** — type ends with `!` OR body contains `BREAKING CHANGE:`
- **Features** — `feat`
- **Fixes** — `fix`
- **Performance** — `perf`
- **Reverts** — `revert`
- **Other** — `chore`, `docs`, `test`, `build`, `ci`, `style`, `refactor` (informational only)

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

## Phase C — Execute

After approval at B.5, run all of the following without further user
interaction. Halt on the first failure.

### C.1 Apply CHANGELOG and version

1. Prepend the rendered CHANGELOG entry to `CHANGELOG.md`.
2. Update `package.json#version` to `NEXT_VERSION`.
3. **(Monorepo only)** Update each `packages/*/package.json#version` to
   `NEXT_VERSION`.

### C.2 Commit

```bash
git add CHANGELOG.md package.json
# (Monorepo: also `git add packages/*/package.json`)
git commit -m "chore: release v${NEXT_VERSION}"
```

### C.3 Push (commit only, no tag yet)

```bash
git push
```

### C.4 Final pre-tag verification

Re-invoke `/solo-npm:verify` (or `/verify` wrapper) against the bumped
state. Aborts here are rare but not hypothetical (e.g., a test that
depends on `package.json#version`).

If anything fails, **STOP**. Recovery: fix the issue, restart from
Phase A. The release commit is on origin but no tag has been pushed
yet, so the workflow won't run.

### C.5 Tag and push the tag

```bash
git tag "v${NEXT_VERSION}"
git push --tags
```

This triggers the publish workflow on the new tag.

### C.6 Watch CI

```bash
gh run watch --exit-status
```

If CI fails, **STOP** and surface the error. The tag is on origin
(intentional record of intent); recovery is `gh run rerun <id>` after
fixing the cause. **Do not** bump the version unless the tarball needs
new content.

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
