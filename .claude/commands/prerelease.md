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
2. Verify `release.yml` includes the dist-tag detection step (grep for `EXPLICIT_TAG=` or the three-layer logic). If missing, STOP with:

   > Your `release.yml` doesn't detect dist-tag from version. Pre-releases would publish to `@latest` and clobber stable users. Run `/solo-npm:init` to refresh the workflow template (idempotent — only updates `release.yml` if it's missing the dist-tag step). Then re-run `/solo-npm:prerelease`.

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

## Phase C — Execute

After approval, run all of the following without further user interaction. Halt on first failure.

### C.1 Apply CHANGELOG and version

1. Prepend the rendered CHANGELOG entry to `CHANGELOG.md`.
2. Update `package.json#version` to `NEXT_VERSION` via:

   - START or PROMOTE: `npm version <NEXT_VERSION> --no-git-tag-version` (or direct edit + manifest write for monorepos with per-package versions).
   - BUMP: `npm version prerelease --no-git-tag-version`.

3. (Monorepo only) Update each `packages/*/package.json#version` to `NEXT_VERSION`.

### C.2 Commit

```bash
git add CHANGELOG.md package.json
# (Monorepo: also `git add packages/*/package.json`)
git commit -m "chore: release v${NEXT_VERSION}"
```

### C.3 Push

```bash
git push
```

### C.4 Final pre-tag verification

Re-invoke `/solo-npm:verify` (or `/verify` wrapper). Halt on failure.

### C.5 Tag and push

```bash
git tag "v${NEXT_VERSION}"
git push --tags
```

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
