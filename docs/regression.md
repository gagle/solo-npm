# Regression scenarios

> Manual walkthrough used before each release as drift insurance. Catches behavioral changes between Claude versions or between skill-spec edits without requiring a full integration-test framework.

## How to use this doc

Before tagging a release:

1. Pick a representative consumer repo (rfc-bcp47 or ncbijs work well).
2. Walk through the scenarios below for the skills you've touched.
3. For scenarios you didn't touch, spot-check 2–3.
4. If any "Expected" deviates, investigate before shipping.

Each scenario has:
- **Setup**: state to put the repo in
- **Trigger**: what to type to Claude
- **Expected**: the agent's behavior (what it does, what it asks, what it outputs)
- **Verify**: how to confirm it worked end-to-end

## Scenarios

### S1 — Daily release on a clean repo

**Setup**: clean working tree; `package.json#version` is stable (e.g., `1.5.0`); a few `feat:` / `fix:` commits since the last tag.

**Trigger**: *"ship a release"*

**Expected**:
- Phase A silent (verify passes; trust + audit caches hot).
- Phase B renders auto-bump suggestion (e.g., `1.5.1` from `fix:` commits) + changelog draft.
- ONE `AskUserQuestion`: *"Approve release plan?"* with options *Proceed with v1.5.1 / Abort*.
- On Proceed: bumps `package.json`, prepends CHANGELOG, commits `chore: release v1.5.1`, pushes, tags `v1.5.1`, pushes tag, watches CI, verifies registry attestation, creates GitHub Release with the just-prepended CHANGELOG entry as body, updates `pkgCheck.lastSize`, prints final notification.

**Verify**:
- `git log` shows the new release commit
- `git tag` shows `v1.5.1`
- `npm view <pkg>@1.5.1 version` returns `1.5.1`
- `npm view <pkg>@1.5.1 dist.attestations` returns provenance data
- GitHub Release page (`gh release view v1.5.1`) shows the CHANGELOG body
- `.solo-npm/state.json#pkgCheck.lastSize["<pkg>@1.5.1"]` is set

### S2 — Pre-release start (new beta line)

**Setup**: clean tree; current version `1.5.0` (stable); want to start a `2.0.0-beta` line.

**Trigger**: *"start a beta for v2"*

**Expected**:
- Phase 0 extracts `IDENTIFIER=beta`, `BASE_BUMP=major`. Skips both START questions.
- Phase A pre-flight passes (release.yml has dist-tag step from /init scaffold).
- Phase B renders plan: `1.5.0 → 2.0.0-beta.0`. ONE `AskUserQuestion`: *Proceed with v2.0.0-beta.0 / Abort*.
- On Proceed: bumps to `2.0.0-beta.0`, commits, tags, pushes, CI publishes to `@next`.
- Phase C.7 confirms `npm view dist-tags.next === "2.0.0-beta.0"`.
- Phase C.7.5 creates GitHub Release with `--prerelease` flag.

**Verify**:
- `npm view <pkg> dist-tags.next` returns `2.0.0-beta.0`
- `npm view <pkg> dist-tags.latest` unchanged at `1.5.0`
- GitHub Release shows the "Pre-release" badge

### S3 — Pre-release bump → promote → cleanup chain

**Setup**: continue from S2 — repo is on `2.0.0-beta.0`.

**Trigger 1**: *"ship the next beta"* → expects auto-chain into /prerelease BUMP path.

**Expected 1**:
- /release Phase B.2 detects pre-release-shape version → auto-chains into `/solo-npm:prerelease`.
- /prerelease Phase B BUMP/PROMOTE path: AskUserQuestion *Bump pre-release counter / Promote to stable / Abort*.
- User picks Bump → publishes `2.0.0-beta.1` to `@next`.

**Trigger 2**: *"promote v2 to stable"* → expects PROMOTE path + post-promote cleanup chain.

**Expected 2**:
- /prerelease detects pre-release state.
- AskUserQuestion gates: pick Promote → publishes `2.0.0` to `@latest`.
- Phase E AskUserQuestion: *"Remove stale @next now? (currently points at 2.0.0-beta.1, but @latest is now 2.0.0)"*.
- User picks Yes → chains to `/solo-npm:dist-tag cleanup-stale`.
- /dist-tag removes `@next` from affected packages.

**Verify**:
- `npm view <pkg> dist-tags.latest` returns `2.0.0`
- `npm view <pkg> dist-tags.next` returns nothing (cleaned up)

### S4 — Hotfix on a maintenance line with forward-port

**Setup**: main is on `2.0.0` (stable, just promoted). Old `1.x` users hit a bug. No `1.x` branch yet.

**Trigger**: *"users on v1.5 are hitting a bug where the rate limiter doesn't honor Retry-After. Fix it on the v1 line and ship."*

**Expected**:
- Phase 0 extracts `TARGET_MAJOR=1`, `FIX_DESCRIPTION="rate limiter Retry-After bug"`.
- Phase A: only one candidate major (1) → auto-pick, no prompt.
- Phase B: creates `1.x` branch from `v1.5.0`, switches to it.
- Phase C: target dist-tag = `@v1` (because main has shipped 2.0.0; 1.x is now legacy).
- Phase D.2 (describe-the-fix): if agent-skills installed, delegates; otherwise applies fix directly.
- /verify passes.
- Phase E.3: *Proceed with v1.5.1 (publishes to @v1) / Abort*. User Proceeds.
- Phase E.4: sets `publishConfig.tag = "v1"`, commits, tags, pushes, CI publishes.
- Phase E.5: `npm view <pkg> dist-tags.v1 === "1.5.1"`; `dist-tags.latest` unchanged at `2.0.0`.
- Phase E.6: GitHub Release with `--latest=false --target 1.x`.
- Phase F: `git checkout main`.
- Phase F.5: AskUserQuestion *Forward-port to main? Yes ship / Yes cherry-pick only / No skip*.
- User picks "Yes — cherry-pick and ship" → cherry-pick succeeds (main has same code path) → chains to `/release` → ships `2.0.1` to `@latest`.

**Verify**:
- `git branch -a` shows `1.x` branch
- `npm view <pkg> dist-tags` shows `latest: 2.0.1, v1: 1.5.1`
- GitHub Releases UI: latest badge on `v2.0.1`; `v1.5.1` exists but doesn't have latest badge

### S5 — CVE response with deprecate

**Setup**: a Tier-1 CVE has landed in a runtime dep (e.g., `tar-fs@2.1.2`).

**Trigger**: *"audit my deps and fix"*

**Expected**:
- /audit runs pnpm audit; classifies tar-fs into Tier 1.
- Phase 4 renders per-tier table with the advisory.
- Phase 5 AskUserQuestion: *Fix Tier 1 now (Recommended) / Fix Tier 1 + 2 / Deprecate affected versions / Show details / Defer*.
- User picks "Fix Tier 1 now" → chains to `/solo-npm:deps cve-tier-1`.
- /deps Phase 1: filters to the tar-fs upgrade; classifies as CVE-driven.
- /deps Phase 4: snapshots, runs `pnpm update tar-fs@<fix-version>`, runs /verify, commits on pass.
- /audit Phase 6: writes back `tier1Count: 0` to state.json.

**Verify**:
- `git log` shows `chore(deps): upgrade tar-fs to <fix> (GHSA-xxxx)`
- `pnpm audit` reports zero advisories on the affected dep
- `.solo-npm/state.json#audit.tier1Count === 0`

Then: *"ship the patch"* → /release ships `1.5.2` (or whatever) with the dep upgrade included.

### S6 — Botched-release recovery

**Setup**: just shipped `1.6.0` with a bug; users are getting bad installs.

**Trigger 1**: *"repoint @latest to 1.5.2 — 1.6.0 has a bug"*

**Expected**:
- /dist-tag Phase 0: `OPERATION=repoint, TAG=latest, VERSION=1.5.2, SCOPE=current`.
- Phase A: auth check (npm whoami). If logged out, foolproof handoff.
- Phase B: render *Repoint @latest from 1.6.0 → 1.5.2 / Abort*. User Proceeds.
- Phase C: `npm dist-tag add <pkg>@1.5.2 latest`.

**Verify**: `npm view <pkg> dist-tags.latest` returns `1.5.2`.

**Trigger 2**: *"now mark 1.6.0 as do-not-use"*

**Expected**:
- /deprecate Phase 0: `OPERATION=deprecate, RANGE=1.6.0, MESSAGE` derived.
- Phase A pre-flight: `RANGE` is concrete (not unbounded), MESSAGE non-empty → passes.
- Phase B: render plan + Proceed gate.
- Phase C: `npm deprecate <pkg>@1.6.0 "<msg>"`.

**Verify**: `npm view <pkg>@1.6.0 deprecated` returns the message.

**Trigger 3** (fix): *"ship 1.6.1"*

**Expected**: /release ships `1.6.1` with the fix; `@latest` moves to 1.6.1 automatically (npm publish behavior). The `1.6.0` deprecation persists; that's OK.

### S7 — First publish on a brand-new repo

**Setup**: fresh repo with `pnpm init` done, no `.claude/`, no release.yml, no published version.

**Trigger**: *"integrate solo-npm into this repo and bootstrap"*

**Expected**:
- Plugin install prompts (one-time per machine).
- /init Phase 1: scaffolds `release.yml` + `publishConfig` + `.nvmrc` + wrappers + `.solo-npm/state.json`.
- Phase 1d: invokes `/verify --pkg-check-only`. May surface warnings (missing description, keywords, repository.url). Auto-fix offers for derivable fields.
- Phase 1e: AskUserQuestion *Commit + push (Recommended) / Commit only / Stage only / Skip*.
- Phase 2: `npm-trust --doctor` detects PACKAGE_NOT_PUBLISHED. `npm whoami` check + foolproof login handoff if needed.
- Phase 2 AskUserQuestion: *Run `npm publish --provenance=false --access public` now to claim the package name `<NAME>`? Yes / I'll publish manually / Abort*.
- User picks Yes → agent runs publish.
- Phase 3: chains into `/solo-npm:trust`. Trust wizard runs; user does web 2FA.
- Phase 4: prints "Init complete".

**Verify**:
- `npm view <pkg>` returns the published version
- `npm-trust --doctor` reports no `PACKAGE_NOT_PUBLISHED` issues
- `.solo-npm/state.json#trust.configured` lists the package

### S8 — Bundle-size regression detection

**Setup**: repo with `1.5.0` published; a `pkgCheck.lastSize` baseline exists. Make a code change that causes a 50%+ increase in tarball size (e.g., import a large dep without marking external).

**Trigger**: *"verify"*

**Expected**:
- /verify Steps 1–4 (lint/typecheck/test/build) pass.
- Step 5 Tier 1 (publint), Tier 2 (manual), Tier 3 (pack-audit) pass.
- Step 5 Tier 4 detects the regression: prior baseline was X bytes; current is Y bytes; delta > 25% → warning.
- Top-5-largest-files breakdown surfaced.

**Verify**: warning includes file paths and sizes; user can identify the bloat source.

### S9 — Stale @next warning + cleanup chain

**Setup**: repo just promoted (S3); `@next` registry tag still points at the old beta.

**Trigger**: *"how are my packages doing?"*

**Expected**:
- /status renders dashboard.
- "Stale-@next" warning surfaces above the table: *"@next is stale on <pkg> (points at 2.0.0-beta.1, but @latest is 2.0.0). Cleanup: /solo-npm:dist-tag cleanup-stale"*.
- "Portfolio health" section shows audit + pkgCheck cache state.

**Trigger 2**: *"cleanup stale @next"*

**Expected**: /dist-tag cleanup-stale removes the @next pointer; warning gone on next /status.

### S10 — Verify catches secrets in tarball (HARD STOP)

**Setup**: accidentally `git add .env` (or similar secret file).

**Trigger**: *"verify"*

**Expected**:
- Step 5 Tier 3 npm pack --dry-run detects `.env` in tarball.
- HARD STOP with verbatim secrets-detection block: file list + remediation steps (`git rm --cached`, `.gitignore`, `.npmignore`, **rotate credentials**) + "/release will not proceed".
- /release also blocked because Phase A.2 invokes /verify and gets the same HARD STOP.

**Verify**: agent does NOT proceed past the secrets detection. User must clean up before release.

### S11 — `gh` not authenticated (graceful skip)

**Setup**: `gh auth logout` to simulate `gh` unavailable.

**Trigger**: *"ship a release"*

**Expected**:
- /release runs through phases normally up to Phase C.7.5 (GitHub Release creation).
- Phase C.7.5 detects `gh auth status` fails → surfaces non-fatal warning: *"gh not authenticated; skipping GitHub Release creation. The git tag is pushed and the npm publish succeeded, but the GitHub Release page will be empty."*
- Skill continues to C.7.6 (cache update) and C.8 (final notification).

**Verify**: tag is pushed; npm package is live; GitHub Release page is empty (expected).

### S12 — Cross-machine resume (committed state.json)

**Setup**: repo with `.solo-npm/state.json` committed. Open the repo in Claude Code on a fresh machine (or after `git clone`).

**Trigger**: *"how are my packages doing?"*

**Expected**:
- /status reads cached state.json without making fresh npm calls (hot path).
- Trust state column populated from cache.
- Portfolio health section populated from cache.
- Maintenance lines populated from local git.

**Verify**: dashboard renders quickly; no rate-limit-style delays.

## Drift indicators to watch

When walking through these scenarios, **suspect drift** if you see:

- A Phase that's described in the skill body but doesn't fire when expected
- An `AskUserQuestion` with different option text than the spec describes
- An auto-chain that doesn't trigger when its condition is met
- A skill chain target that 404s (referenced skill missing)
- Verbose output that doesn't match the spec's documented output format

Investigate any of those before shipping.

## Updating this doc

When adding a new skill, add at least one regression scenario. When changing an existing skill's behavior in a way that affects user-visible flow, update the affected scenario. The discipline cost is low (~5 min per change); the regression-prevention value is real.
