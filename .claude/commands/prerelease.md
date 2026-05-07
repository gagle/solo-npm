---
description: Pre-release lifecycle — start a new pre-release line (alpha/beta/rc), bump the counter on an in-progress line, or promote a pre-release to stable. Triggers from natural prompts like "start a beta for v2", "cut a release candidate", "publish 2.0.0-beta.1", "ship a beta of breaking changes", "promote the beta to stable", or "next channel publish for testing". AI-driven: the skill asks structured questions; the agent handles all package.json edits, commits, tags, and publishes.
---

# Pre-release

Pre-release lifecycle operations: **start** a new pre-release line, **bump** the counter on an in-progress line, or **promote** a pre-release to stable. AI-driven end-to-end: the user expresses intent via prompt; the agent handles every code/git operation.

## When to use

You want to publish a pre-release version (e.g., `1.1.0-beta.1`) so opt-in users can install via `npm i pkg@next` while users on `npm i pkg` (the `latest` channel) stay undisturbed.

Common scenarios:

1. **Major version transitions** — preview breaking changes before locking in `2.0.0`. Ship `2.0.0-beta.1`, gather feedback, fix issues, promote.
2. **Risky features / refactors** — verify in production-like conditions before promoting.
3. **Experimental APIs** — early-adopter feedback on opt-in interfaces.
4. **Downstream coordination** — a consumer needs to test against unreleased changes via `@next`.
5. **First-publish OIDC verification** — once-per-repo lifetime; ship `0.1.0-test.1` to verify the tag-triggered CI publishes correctly.

For ordinary stable releases on the current major, use `/release`.

## How it works

Three operations in one skill, auto-detected from current state:

| `package.json#version` shape | Operation |
|---|---|
| Stable (`x.y.z`) | **START** a new pre-release line |
| Pre-release (`x.y.z-id.n`) | **BUMP** counter or **PROMOTE** to stable |

## Phase 0 — Read prompt context

Scan the user's invoking prompt (and recent chat turns if relevant) for hints that pre-fill subsequent questions. Don't ask redundantly when the user already said it.

| Prompt mentions | Pre-fill |
|---|---|
| `alpha`, `beta`, `rc`, `experimental`, `pre`, `next` | `IDENTIFIER` |
| `major`, `minor`, `patch`, or specific target version (`v2`, `2.x`, `2.0.0`) | `BASE_BUMP` (or full version if they named one) |
| Full pre-release version (`v2.0.0-beta.1`, `1.5.0-rc.0`) | `NEXT_VERSION` directly — skip both START questions |

Examples:
- *"Start a beta for v2"* → `IDENTIFIER=beta`, `BASE_BUMP=major` (since current is 1.x). Skip both START questions.
- *"Cut a release candidate"* → `IDENTIFIER=rc`. Still ask base bump.
- *"Publish v2.0.0-beta.1"* → `NEXT_VERSION=2.0.0-beta.1` directly.
- *"Start an alpha"* → `IDENTIFIER=alpha`. Still ask base bump.
- Bare *"start a pre-release"* → ask both START questions.

If extraction is ambiguous (user said "alpha" but mentioned no base), pre-fill what's clear and ask the rest. Never over-extract.

## Phase A — Pre-flight + state detection

1. Working tree clean (`git status --porcelain`); else STOP with the standard "commit or stash first" message.
2. Verify `release.yml` includes the dist-tag detection step (grep for `EXPLICIT_TAG=` or the three-layer logic). If missing, **auto-chain to `/solo-npm:init --refresh-yml` after a single approval gate**:

   ```
   Header:   "Refresh release.yml"
   Question: "Your release.yml is missing the dist-tag detection step. Without it, pre-releases would publish to @latest and clobber stable users. Refresh now?"
   Options:
     - Refresh release.yml and continue (Recommended) — runs /solo-npm:init --refresh-yml
     - Abort
   ```

   On `Refresh and continue`: invoke `/solo-npm:init --refresh-yml` (it performs targeted insert + commits + pushes a single `chore: refresh release.yml for dist-tag detection` commit). Then re-enter Phase A from step 1 (working tree is clean again because the refresh started clean and made one new commit).

   On `Abort`: STOP gracefully. Print: *"Aborted. Re-run `/solo-npm:prerelease` when ready."*

3. Read `package.json#version` and `LATEST_TAG` (`git tag --list 'v*' --sort=-v:refname | head -1`).
4. Detect current state — Stable shape (`^\d+\.\d+\.\d+$`) or Pre-release shape (`^\d+\.\d+\.\d+-[a-z]+\.\d+$`).

## Phase B — Branch on state

### START path (current version is stable)

If Phase 0 didn't already pre-fill `IDENTIFIER`, ask:

```
Header:   "Pre-release identifier"
Question: "Pre-release identifier?"
Options:  alpha / beta / rc / Other (specify)
```

If Phase 0 didn't already pre-fill `BASE_BUMP` (and didn't pre-fill full `NEXT_VERSION`), ask:

```
Header:   "Base version"
Question: "Base version bump from v{CURRENT}?"
Options:  patch (v{CURRENT+patch}) / minor (v{CURRENT+minor}) / major (v{CURRENT+major}) / Other (specify)
```

Compute `NEXT_VERSION = <bumped-base>-<id>.0` (unless Phase 0 pre-filled it).

Render plan summary + changelog draft to chat (visible). Then:

```
Header:   "Release"
Question: "Approve release plan?"
Options:  Proceed with v{NEXT_VERSION} / Abort
```

### BUMP/PROMOTE path (current version is pre-release)

Render plan summary + changelog draft (single bullet referencing the pre-release context) to chat.

```
Header:   "Release"
Question: "Approve release plan?"
Options:  Bump pre-release counter (v{CURRENT} → v{NEXT_BUMP}) / Promote to stable (v{STABLE}) / Abort
```

Where `NEXT_BUMP` is computed via `npm version prerelease` semantics (e.g., `1.1.0-beta.1` → `1.1.0-beta.2`), and `STABLE` is the base version stripped of pre-release suffix (e.g., `1.1.0-beta.3` → `1.1.0`).

If Phase 0 extracted a hint like "promote", pre-select that option (still surface the prompt for explicit approval — the publish itself is destructive).

## Phase C.0 — Error-handling patterns (H2, H3, H4, H6 from `/unpublish` reference)

Before the execute steps, apply the standard solo-npm error patterns. Canonical wording lives in `/unpublish` Phases C.0–D.2:

- **H2 — `.solo-npm/state.json` corruption guard**: state.json reads (trust cache lookups during Phase A) wrap in try/catch. On parse fail surface non-fatal warning, treat as empty cache, continue.
- **H3 — Auth-window race**: pre-release publish runs in CI via OIDC, so the local-auth race window doesn't apply to the publish itself. The chain to `/solo-npm:dist-tag cleanup-stale` (Phase E PROMOTE) inherits the dist-tag skill's own Phase C.0 H3/H1 handlers, so the race is covered transitively.
- **H4 — Registry propagation lag retry**: Phase C.7 verify (`npm view <pkg> dist-tags.next` etc.) uses 3 attempts × 5s sleep before declaring inconsistency. Don't HARD STOP — surface non-fatal note about CDN lag and continue to summary.
- **H6 — Chain-target failure recovery**: chains into `/verify` (Phase C.4) and `/dist-tag cleanup-stale` (Phase E PROMOTE). Capture child STOP messages verbatim and surface in `/prerelease` context with retry/abort options.

## Phase C — Execute

After approval, run all of the following without further user interaction. Halt on first failure.

### C.1 Apply CHANGELOG and version

1. Generate the CHANGELOG entry per operation:

   - **START**: standard auto-generated entry derived from commits since `LATEST_TAG` (e.g., `v1.5.0` → `1.6.0-beta.0` covers the same commits the equivalent stable bump would have covered). Conventional-commits parse: `feat:` → Features, `fix:` → Bug Fixes, breaking changes → Breaking. Render as a `## v{NEXT_VERSION} ({DATE})` block.

   - **BUMP**: minimal entry covering only commits since the *previous pre-release tag* (e.g., for `1.6.0-beta.1` → `1.6.0-beta.2`, walk `git log v1.6.0-beta.1..HEAD`). Each beta increment gets its own delta entry for engineer-facing history.

   - **PROMOTE**: **aggregate** all changes since the last *stable* tag into one comprehensive entry. This covers everything that happened across `beta.0` → `beta.N` plus the promote commit itself.

     ```bash
     # Find the last stable tag (excludes pre-releases):
     LAST_STABLE_TAG=$(git tag --list 'v*' --sort=-v:refname | grep -vE -- '-[a-z]+\.[0-9]+$' | head -1)
     # Walk commits since then:
     git log "${LAST_STABLE_TAG}..HEAD" --format='%H %s'
     ```

     Conventional-commits parse the full range:
     - `feat(scope)?: ...` → Features
     - `fix(scope)?: ...` → Bug Fixes
     - `BREAKING CHANGE:` in body or `feat!:` / `fix!:` → Breaking Changes
     - `chore: release v...` → skip (release commits)
     - `chore:`, `docs:`, `refactor:`, `test:` → skip unless marked `!` (breaking)

     Render as `## v{STABLE_VERSION} ({DATE})` with sections in order: Breaking Changes, Features, Bug Fixes.

     **Dedup is automatic**: each commit appears once in the range, so identical fixes that landed in beta.1 don't double-count.

2. Prepend the entry to `CHANGELOG.md`. For PROMOTE specifically: the new stable entry sits **above** the existing per-beta entries (do NOT delete or condense the per-beta entries — they remain as engineer-facing historical record).

3. Update `package.json#version` to `NEXT_VERSION` via:

   - START or PROMOTE: `npm version <NEXT_VERSION> --no-git-tag-version` (or direct edit + manifest write for monorepos with per-package versions).
   - BUMP: `npm version prerelease --no-git-tag-version`.

4. (Monorepo only) Update each `packages/*/package.json#version` to `NEXT_VERSION`.

### C.2 Commit

```bash
git add CHANGELOG.md package.json
# (Monorepo: also `git add packages/*/package.json`)
git commit -m "chore: release v${NEXT_VERSION}"
```

### C.3 Push (commit only) — with rejection categorization

Apply the same git-push-rejection categorization as `/solo-npm:release` C.3 (canonical reference). Capture stderr, switch on common rejection patterns (non-fast-forward, server-side hook, branch protection, auth fail), and surface the matching remediation. Wrap with `timeout 60` to avoid indefinite hangs on stalled remotes. On any failure, halt before tagging.

### C.4 Final pre-tag verification

Re-invoke `/solo-npm:verify` (or `/verify` wrapper). Halt on failure.

### C.5 Tag and push — with collision pre-flight

Apply the same tag-collision pre-flight as `/solo-npm:release` C.5 (canonical reference). Pre-flight steps:

1. `timeout 30 git ls-remote --tags origin "refs/tags/v${NEXT_VERSION}"` — if the tag already exists on remote, **STOP** with a remediation block (typically the user should `git push origin :refs/tags/v${NEXT_VERSION}` first, or a prior /prerelease run failed mid-flow and left the tag).
2. Only then `git tag "v${NEXT_VERSION}"` locally.
3. `timeout 60 git push --tags` with the same rejection categorization as C.3. On failure: clean up the local tag (`git tag -d`) so the next attempt can re-tag.

### C.6 Watch CI

```bash
gh run watch --exit-status
```

If CI fails, STOP and surface the error.

### C.7 Verify on the registry

For pre-release versions on public npm:

```bash
npm view "<PACKAGE_NAME>@${NEXT_VERSION}" version
npm view "<PACKAGE_NAME>@${NEXT_VERSION}" dist.attestations
npm view "<PACKAGE_NAME>" dist-tags.next
```

The third should equal `NEXT_VERSION` — confirms the publish landed on `@next` (not `@latest`). For PROMOTE operations, also verify `dist-tags.latest` updated.

For monorepo: iterate per package.

### C.7.5 Create GitHub Release with notes (marked pre-release)

Same pattern as `/release` Phase C.7.5, with two adjustments:
- Append `--prerelease` flag so the GitHub Release page shows the "Pre-release" badge.
- For START operations: the CHANGELOG entry is for the pre-release line's first version (e.g., `v1.6.0-beta.0`); use that as both the tag and the title.
- For BUMP operations: same — the per-bump CHANGELOG entry becomes the release notes body.
- For PROMOTE operations: the aggregated stable CHANGELOG entry (covering all betas in the line) is the body. NOTE: PROMOTE results in a stable release, NOT a pre-release; do NOT append `--prerelease` for PROMOTE.

```bash
# Pre-flight (same as /release C.7.5 — distinguish not-installed vs not-authenticated):
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

  # Determine the prerelease flag based on operation:
  if [ "${OPERATION}" = "PROMOTE" ]; then
    PRERELEASE_FLAG=""  # stable release
  else
    PRERELEASE_FLAG="--prerelease"
  fi

  gh release create "v${NEXT_VERSION}" \
    --title "v${NEXT_VERSION}" \
    --notes "${NOTES}" \
    ${PRERELEASE_FLAG}
fi
```

### C.7.6 Update bundle-size baseline cache

Same pattern as `/release` Phase C.7.6 — write the new version's `unpackedSize` to `.solo-npm/state.json#pkgCheck.lastSize` keyed by `<pkg>@<version>`, then trim to keep at most the **last 3 versions per package** (semver-sorted descending). Pre-release versions feed the cache like stable versions; the trim treats them uniformly.

### C.8 Final notification

```
Released v${NEXT_VERSION} to npm (@${dist-tag}).
  Tarball: https://www.npmjs.com/package/<PACKAGE_NAME>/v/${NEXT_VERSION}
  CI:      <gh run url>
```

## Phase D — Post-release cache update

Refresh `.solo-npm/state.json`:
- `cache.trust.lastFullCheck = now` (the CI's successful publish proves trust still works for every package that just published).
- For START operations: package was likely already in `cache.trust.configured`; no new entries needed.

## Phase E — Stale `@next` cleanup offer (PROMOTE only)

Skipped automatically for START and BUMP operations. Fires only after a successful PROMOTE — at which point `@next` may still point at the old beta version (npm doesn't auto-cleanup), and consumers running `npm i pkg@next` would still install the now-superseded beta instead of the stable.

Surface an `AskUserQuestion` gate:

```
Header:   "Cleanup @next"
Question: "Remove stale @next now? (it currently points at v${PREV_BETA_VERSION}, but @latest is now v${STABLE_VERSION})"
Options:
  - Yes — chain to /solo-npm:dist-tag cleanup-stale (Recommended)
  - No — leave for later (you can run /solo-npm:dist-tag cleanup-stale anytime)
```

On Yes: invoke `/solo-npm:dist-tag cleanup-stale` (single-package or workspace-wide depending on this repo's scope). The dist-tag skill runs its own Phase A → D and returns.

On No: end with a footer hint:

> Promoted to v${STABLE_VERSION}. To cleanup @next later: `/solo-npm:dist-tag cleanup-stale`

## Failure modes

| Failure | Where | Recovery |
|---|---|---|
| Working tree dirty | Phase A | Commit or stash; re-run |
| `release.yml` missing dist-tag step | Phase A | `/solo-npm:init` refreshes template |
| /verify fails | C.4 | Fix the issue; re-run |
| CI fails | C.6 | `gh run rerun <id>` after fixing |
| `npm view dist-tags.next` doesn't match | C.7 | Registry propagation lag (retry) or release.yml's dist-tag step is broken (re-init) |

## What this skill does NOT do

- Auto-fallback to stable if pre-release fails — keeps the user in control.
- Override `package.json#version` arbitrarily without an identifier or base-bump signal.
- Touch other packages in monorepos when only one is in pre-release. Solo-dev unified-versioning means all packages move together; if you want per-package pre-release, that's a future v0.6.0+ feature.
