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

**Expected** (current as of v0.13.0):
- **Phase −1** (universal): tool detection, `.gitconfig` user check, env-vars OK, atomic state.json helper available, gh pre-flight (`GH_AVAILABLE=1` if gh authenticated). Silent on the happy path.
- **Phase 0**: prompt-context extraction. *"ship a release"* doesn't pre-fill a specific version → no override.
- **Phase 0.5**: regex validation — no extracted slots; nothing to validate.
- **Phase 0.5b**: shell-safety — no extracted slots; nothing to validate.
- **Phase A.1** (working tree clean): passes silently. Detached-HEAD detection passes; worktree detection passes; submodule detection passes.
- **Phase A.2** (verify): `/verify` invocation. Passes silently. Tier 3 secrets check passes. CRLF-in-tarball check passes for typical repos (no bin scripts with CRLF).
- **Phase A.3** (cache-aware trust check): hot path — `state.json#trust.configured` is fresh and includes this package; zero npm calls; silent.
- **Phase A.5** (audit cache check): hot path — `state.json#audit.tier1Count == 0` and `tier2Count == 0`; silent. (If tier1Count > 0, would HARD STOP with cached CVE list per v0.10.1.)
- **Phase B.3** (collect commits since LATEST_TAG): `git log` capped at `--max-count=500` (Tier-3 K, v0.12.0+). Conventional-commits parse rejects unknown types with "did you mean" hints (Tier-3 F, v0.12.0+).
- **Phase B.5** renders auto-bump suggestion (e.g., `1.5.1` from `fix:` commits) + changelog draft.
- ONE `AskUserQuestion`: *"Approve release plan?"* with options *Proceed with v1.5.1 / Abort*.
- **Phase C.0a** (concurrent-release lock, v0.11.0+): acquires `.solo-npm/locks/<repo-root-hash>.lock` with stale-PID auto-cleanup.
- **Phase C.1** (apply CHANGELOG + version): atomic state.json write per H7.
- **Phase C.2** (commit) with pre-commit hook failure rollback (Tier-2 B3, v0.11.0+).
- **Phase C.3** (push commit) with rejection categorization (non-FF / pre-receive / pre-push hook / branch protection / auth) + SSL/TLS remediation if applicable (Tier-3 B, v0.12.0+).
- **Phase C.4** (final pre-tag verify): re-runs `/verify`.
- **Phase C.5** (tag + push): pre-flight `git ls-remote --tags origin "refs/tags/v1.5.1"` (Tier-1 v0.10.1) — STOP if exists. Then `git tag` (sets `LOCAL_TAG_PENDING_PUSH` for SIGINT trap). Then `git push --tags` with same rejection categorization. On tag push failure, rolls back local tag.
- **Phase C.6** (CI watch): `timeout 1800 gh run watch --exit-status` (Tier-2 B1). Skipped if `GH_AVAILABLE=0`.
- **Phase C.7** (registry verify): post-publish `npm view` with H4 propagation-lag retry (3 × 5s). Surfaces non-fatal note if registry still inconsistent after 15s.
- **Phase C.7.5** (GitHub Release): `gh release create` with B5 warn-don't-fail behavior (publish already succeeded; release page is secondary).
- **Phase C.7.6** (bundle-size cache): atomic state.json update per H7.

**Verify**:
- `git log` shows the new release commit
- `git tag` shows `v1.5.1`
- `npm view <pkg>@1.5.1 version` returns `1.5.1`
- `npm view <pkg>@1.5.1 dist.attestations` returns provenance data
- GitHub Release page (`gh release view v1.5.1`) shows the CHANGELOG body
- `.solo-npm/state.json#pkgCheck.lastSize["<pkg>@1.5.1"]` is set
- `.solo-npm/locks/` is empty (lock released on EXIT trap)

### S2 — Pre-release start (new beta line)

**Setup**: clean tree; current version `1.5.0` (stable); want to start a `2.0.0-beta` line.

**Trigger**: *"start a beta for v2"*

**Expected** (current as of v0.13.0):
- Phase 0 extracts `IDENTIFIER=beta`, `BASE_BUMP=major`. Skips both START questions.
- **Phase 0.5 IDENTIFIER validation** (v0.11.0+): regex `^(alpha|beta|rc|canary|experimental|next|preview|edge)$` passes for `beta`. Typos like `betta` would STOP with did-you-mean.
- **Phase 0.5b shell-safety** (v0.11.0+): no metacharacters in `IDENTIFIER` or `BASE_BUMP`; passes.
- **Phase A pre-flight**: working-tree clean. Verifies `release.yml` includes the dist-tag step (the three-layer detection from `/init` scaffold). If missing, would auto-chain to `/init --refresh-yml` with a single AskUserQuestion gate.
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
- **Phase E gate is PROMOTE-only conditional** (v0.15.0 clarification): the "Remove stale @next?" gate fires ONLY when the operation is PROMOTE; START and BUMP do not trigger it (cleanup wouldn't make sense — the @next line is intentionally active).
- For PROMOTE: Phase E AskUserQuestion *"Remove stale @next now? (currently points at 2.0.0-beta.1, but @latest is now 2.0.0)"*.
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
- Phase E.4: sets `publishConfig.tag = "v1"` via **atomic write** (per H7 / Phase −1.4: write to package.json.tmp + rename — a killed-mid-write process never leaves a corrupt package.json), commits, tags, pushes, CI publishes.
- Phase E.5: `npm view <pkg> dist-tags.v1 === "1.5.1"`; `dist-tags.latest` unchanged at `2.0.0`.
- Phase E.6: GitHub Release with `--latest=false --target 1.x`.
- Phase F: `git checkout main`.
- **Phase F.5 forward-port is gated by AskUserQuestion EVEN when cherry-pick succeeds cleanly** (v0.15.0 clarification — the gate isn't conditional on conflict; the user is always asked because forward-port is a deliberate decision): *Forward-port to main? Yes ship / Yes cherry-pick only / No skip*.
- User picks "Yes — cherry-pick and ship" → cherry-pick succeeds (main has same code path) → chains to `/release` → ships `2.0.1` to `@latest`.

**Verify**:
- `git branch -a` shows `1.x` branch
- `npm view <pkg> dist-tags` shows `latest: 2.0.1, v1: 1.5.1`
- GitHub Releases UI: latest badge on `v2.0.1`; `v1.5.1` exists but doesn't have latest badge

### S5 — CVE response with deprecate

**Setup**: a Tier-1 CVE has landed in a runtime dep (e.g., `tar-fs@2.1.2`).

**Trigger**: *"audit my deps and fix"*

**Expected** (current as of v0.13.0):
- /audit runs pnpm audit; classifies tar-fs into Tier 1.
- **Phase 1 summary-first parse** (Tier-3 F, v0.12.0+): for repos > 500 deps, severity counts parsed first; Tier 1/2 advisory detail drilled into; Tier 3/4 deferred to opt-in `--full`.
- Phase 4 renders per-tier table with the advisory.
- Phase 5 AskUserQuestion: *Fix Tier 1 now (Recommended) / Fix Tier 1 + 2 / Deprecate affected versions / Show details / Defer*.
- User picks "Fix Tier 1 now" → chains to `/solo-npm:deps cve-tier-1`.
- **H6 chain-failure recovery (v0.11.0+)**: if `/deps` STOPs (lockfile conflict, registry unreachable, etc.), `/audit` captures the diagnostic and surfaces retry/switch-to-deprecate/audit-only/abort options.
- /deps Phase 1: filters to the tar-fs upgrade; classifies as CVE-driven.
- /deps Phase 4: snapshots, runs `pnpm update tar-fs@<fix-version>`, runs /verify, commits on pass.
- /audit Phase 6: writes back `tier1Count: 0` to state.json (atomic write per H7).
- **Effect on next `/release`**: Phase A.5 audit cache check (v0.10.1+) sees `tier1Count == 0` and proceeds. Had the user picked "Defer" instead, the cache would still show `tier1Count > 0` and the next `/release` would HARD STOP at A.5 with the cached CVE list.

**Verify**:
- `git log` shows `chore(deps): upgrade tar-fs to <fix> (GHSA-xxxx)`
- `pnpm audit` reports zero advisories on the affected dep
- `.solo-npm/state.json#audit.tier1Count === 0`

Then: *"ship the patch"* → /release ships `1.5.2` (or whatever) with the dep upgrade included.

### S6 — Botched-release recovery

**Setup**: just shipped `1.6.0` with a bug; users are getting bad installs.

**Trigger 1**: *"repoint @latest to 1.5.2 — 1.6.0 has a bug"*

**Expected** (current as of v0.15.0):
- /dist-tag Phase 0: `OPERATION=repoint, TAG=latest, VERSION=1.5.2, SCOPE=current`.
- Phase 0.5 regex validation passes (TAG, VERSION, SCOPE all match their regexes); Phase 0.5b shell-safety passes (no metacharacters).
- Phase A: auth check (npm whoami). If logged out, foolproof handoff.
- Phase B: render *Repoint @latest from 1.6.0 → 1.5.2 / Abort*. User Proceeds.
- **Phase C.0a** (v0.11.0+): H3 auth-race re-check (re-runs `npm whoami` since user paused at the Phase B gate); H5 lock acquisition at `.solo-npm/locks/<pkg>.lock` with stale-PID auto-cleanup.
- Phase C: `npm dist-tag add <pkg>@1.5.2 latest`. EOTP/H1 manual handoff fires if 2FA-on-writes is enabled.

**Verify**: `npm view <pkg> dist-tags.latest` returns `1.5.2`.

**Trigger 2**: *"now mark 1.6.0 as do-not-use"*

**Expected**:
- /deprecate Phase 0: `OPERATION=deprecate, RANGE=1.6.0, MESSAGE` derived.
- Phase 0.5 + 0.5b pass.
- Phase A pre-flight: `RANGE` is concrete (not unbounded), MESSAGE non-empty → passes.
- Phase B: render plan + Proceed gate (single gate — `/deprecate` is non-destructive enough that one gate suffices; the dual-gate pattern is reserved for `/unpublish`).
- **Phase C.0a** (v0.11.0+): H3 auth-race re-check + H5 lock acquisition (same canonical pattern as `/dist-tag`).
- Phase C: `npm deprecate <pkg>@1.6.0 "<msg>"`. MESSAGE is passed via env-var (Phase 0.5b convention) so any benign parens/quotes inside don't disrupt shell interpolation.

**Verify**: `npm view <pkg>@1.6.0 deprecated` returns the message.

**Trigger 3** (fix): *"ship 1.6.1"*

**Expected**: /release ships `1.6.1` with the fix; `@latest` moves to 1.6.1 automatically (npm publish behavior). The `1.6.0` deprecation persists; that's OK.

### S7 — First publish on a brand-new repo

**Setup**: fresh repo with `pnpm init` done, no `.claude/`, no release.yml, no published version.

**Trigger**: *"integrate solo-npm into this repo and bootstrap"*

**Expected** (current as of v0.13.0):
- Plugin install prompts (one-time per machine).
- /init Phase 1: scaffolds `release.yml` + `publishConfig` + `.nvmrc` + wrappers + `.solo-npm/state.json` (workspace edge checks per Tier-3 H: STOPs if pnpm-workspace.yaml glob expands to zero matches; surfaces "publishing only N of M packages" for mixed `private: true` + public).
- **Phase 1c .npmrc API** (Tier-3 E, v0.12.0+): scope→registry detection uses `npm config get @<scope>:registry` rather than ad-hoc `.npmrc` grep.
- **Phase 1d pkg-check validation loop** (v0.12.0+, load-bearing):
  1. Invokes `/verify --pkg-check-only`.
  2. On `PKG_CHECK_OK` → continue to 1e.
  3. On `PKG_CHECK_FAIL` with auto-fix offers → AskUserQuestion → apply selected fixes → **re-runs pkg-check in a loop** until clean or unfixable.
  4. On unresolvable errors → HARD STOP with *"Init scaffolded the workflow files but `package.json` has unresolvable issues."*
  5. **On secrets detection (Tier 3)** → HARD STOP with verbatim secrets-detection block; does NOT proceed to commit.
  6. **H6 chain-failure recovery** (v0.11.0+): if `/verify` itself errors, surfaces retry/skip/abort gate.
- Phase 1e: AskUserQuestion *Commit + push (Recommended) / Commit only / Stage only / Skip*.
- Phase 2a: `npm whoami` check + foolproof login handoff if needed.
- Phase 2b: AskUserQuestion *Run `npm publish --provenance=false --access public` now to claim the package name `<NAME>`? Yes / I'll publish manually / Abort*.
- **Phase 2c**: H3 auth-race re-check before publish (v0.11.0+); EOTP/H1 manual handoff if 2FA-on-writes is enabled.
- User picks Yes → agent runs publish.
- Phase 3: chains into `/solo-npm:trust`. Trust wizard runs; user does web 2FA. **H6 chain-failure recovery**: if `/trust` STOPs, surfaces retry/skip/abort.
- Phase 4: prints "Init complete".

**Verify**:
- `npm view <pkg>` returns the published version
- `npm-trust --doctor` reports no `PACKAGE_NOT_PUBLISHED` issues
- `.solo-npm/state.json#trust.configured` lists the package

### S8 — Bundle-size regression detection

**Setup**: repo with `1.5.0` published; a `pkgCheck.lastSize` baseline exists. Make a code change that causes a 50%+ increase in tarball size (e.g., import a large dep without marking external).

**Trigger**: *"verify"*

**Expected** (current as of v0.15.0):
- /verify Steps 1–4 (lint/typecheck/test/build) pass.
- Step 5 Tier 1 (publint), Tier 2 (manual), Tier 3 (pack-audit) pass.
- Step 5 Tier 4 detects the regression: prior baseline was X bytes; current is Y bytes; delta > 25% → warning. Cache lookup uses `state.json#pkgCheck.lastSize["<pkg>@<prior-version>"]`; first-run packages have no baseline (cache populates organically over releases — silent no-op on first run, not a warning).
- Top-5-largest-files breakdown surfaced.

**Cache mechanics** (v0.11.0+, clarified v0.15.0):
- Subsequent `/release` Phase C.7.6 writes the just-shipped `unpackedSize` to `state.json#pkgCheck.lastSize` via **atomic write** (per H7 / Phase −1.4: `.solo-npm/state.json.tmp` + `fs.renameSync()`).
- Cache is **trimmed to the last 3 versions per package** to keep state.json bounded — older entries deleted on each new write.
- For monorepos: cache is keyed `<pkg-name>@<version>`, so per-package histories accumulate independently.

**Verify**: warning includes file paths and sizes; user can identify the bloat source. After a successful `/release`, `state.json#pkgCheck.lastSize["<pkg>@<new-version>"]` is set; older entries (>3 versions back per package) are pruned.

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

**Expected** (current as of v0.15.0 — Phase A.3 cache logic clarified):
- /status reads cached state.json with **conditional logic** (not a single "hot path"):
  - **Hot path**: cache fresh (`now - lastFullCheck < ttlDays`) AND no new packages detected (workspace.packages set ⊆ cache.trust.configured) → **zero npm calls** for trust state. Pure cache render.
  - **Targeted path**: cache fresh AND new packages detected → run `npm-trust --doctor` only on the deltas (new packages), merge results into cache.
  - **Cold path**: cache stale (`now - lastFullCheck > ttlDays`) → full `npm-trust --doctor` against all packages, refresh cache wholesale.
  - On fresh-machine clone with committed `state.json`: cache is fresh + no new packages typically → hot path. Dashboard renders without npm calls.
- Trust state column populated from cache (whichever path).
- Portfolio health section populated from `state.json#audit` + `state.json#pkgCheck` cache.
- Maintenance lines populated from local git (`git ls-remote --heads origin "*.x"`).

**Verify**: dashboard renders quickly; no rate-limit-style delays. The conditional path that fires depends on cache state vs workspace state — for a typical fresh-clone-with-committed-state.json, hot path fires.

### S13 — Wrong-name unpublish + auto-cleanup (happy path within 72h)

**Setup**: a fresh repo just published `@gagel/eutils@1.0.0` (typo — should have been `@gagle/eutils`) about 5 minutes ago via `/release`. Git tag `v1.0.0` and a GitHub Release `v1.0.0` exist.

**Trigger**: *"I shipped @gagel/eutils — wrong scope. Delete it."*

**Expected**:
- /unpublish Phase 0 extracts OPERATION=unpublish-version, NAME=@gagel/eutils, VERSION=1.0.0.
- Phase A.1 npm whoami passes.
- Phase A.3 calls deps.dev `/dependents` — hits 404 (not yet indexed). Treated as `dependentCount = 0` with a non-fatal note about indexing lag.
- Phase A.4 renders eligibility table; v1.0.0 shows `✓ eligible (within 72h)`.
- Phase B Gate 1 surfaces Deprecate (recommended) vs Unpublish vs Abort. User picks Unpublish.
- Gate 2 fires for explicit destructive confirmation.
- Phase C.0 re-runs `npm whoami` (auth-window race re-check) and acquires `.solo-npm/locks/@gagel_eutils.lock`.
- Phase C.1 runs `npm unpublish @gagel/eutils@1.0.0`.
- Phase D.1 verifies via `npm view` with 3 × 5s retry (may need at least one retry due to CDN propagation).
- Phase D.3 fires the auto-cleanup gate (Yes both / Just tags / No / Abort). User picks "Yes, both".
- `git tag -d v1.0.0 && git push origin :refs/tags/v1.0.0` runs successfully.
- `gh release delete v1.0.0 --yes` runs successfully.
- Final summary shows everything cleaned, including the deps.dev not-indexed note as historical context.

**Verify**:
- `npm view @gagel/eutils@1.0.0` returns 404.
- `git tag --list v1.0.0` returns empty.
- `gh release view v1.0.0` returns "release not found".
- `.solo-npm/state.json#pkgCheck.lastSize` no longer contains `@gagel/eutils@1.0.0`.
- `.solo-npm/locks/` does not contain a stale lock file (trap cleanup ran).

### S14 — Blocked-dependents HARD STOP (no override)

**Setup**: published package with at least one dependent on the registry per deps.dev (any sufficiently-old solo-npm-managed package will work — pick one with a real consumer).

**Trigger**: *"unpublish @ncbijs/eutils@1.5.0 — I want it gone."*

**Expected**:
- Phase A.3 calls deps.dev `/dependents` → returns 200 with `dependentCount > 0`.
- Phase A.4 renders eligibility with the failing-criterion detail: `❌ blocked: dependents (<N>)`.
- Skill **HARD STOPs before any gate fires**, surfacing: *"`<NAME>@<VERSION>` has `<N>` dependents (per deps.dev). Cannot unpublish without breaking them. Deprecate instead, or contact those dependents to migrate first. There is no `--force-with-dependents` override in this skill."*
- No `npm unpublish` call ever runs.

**Verify**: no destructive action; npm registry untouched; `.solo-npm/state.json` unchanged.

### S15 — Auth-window race fires (Phase C.0 re-check)

**Setup**: simulate auth expiry by running `/unpublish` to Gate 2, then in a separate terminal run `npm logout`, then complete Gate 2.

**Trigger**: any `/unpublish` invocation that reaches Gate 2.

**Expected**:
- Phase A.1 `npm whoami` passes (user authenticated at start).
- User pauses at Gate 2 long enough to logout in another terminal.
- User confirms Gate 2 ("I understand and want to proceed").
- Phase C.0 re-runs `npm whoami` and detects logged-out state.
- Skill halts before any `npm unpublish` call. Surfaces the foolproof `npm login` handoff again with an AskUserQuestion gate ("Done with re-login?") — same pattern as Phase A.1.
- After re-login, Phase C.0 re-checks and proceeds.

**Verify**: registry was never touched during the logged-out window. The lock file `.solo-npm/locks/<name>.lock` was either never acquired (if halt happened before C.0 lock step) or properly released via trap.

### S16 — Concurrent-lock refusal

**Setup**: in terminal A, start `/unpublish @scope/foo` and pause it at Gate 2 (don't approve). In terminal B, also start `/unpublish @scope/foo`.

**Trigger**: terminal B's `/unpublish` invocation.

**Expected**:
- Terminal A holds `.solo-npm/locks/@scope_foo.lock` (acquired in Phase C.0 — but the lock is acquired AFTER Gate 2 confirmation, so this scenario requires terminal A to be PAST Gate 2 already).
- More realistic: terminal A is in C.1 mid-execution. Terminal B reaches Phase C.0 and finds the lock held by A's PID.
- Terminal B halts cleanly with: *"ERROR: another /unpublish run holds .solo-npm/locks/@scope_foo.lock (PID `<pid-of-A>`). Wait for it to finish, or remove the stale lockfile if the process is dead."*
- Terminal A continues unaffected.

**Verify**: terminal A completes successfully. Terminal B never executed any `npm unpublish` call. After terminal A completes, the lock file is gone (trap cleanup); terminal B can be re-run successfully.

### S17 — deps.dev 404 treated as 0 dependents (not API-down)

**Setup**: publish a fresh new package (within the last few minutes — definitely before deps.dev has indexed it). Confirm `curl -s -o /dev/null -w "%{http_code}" https://api.deps.dev/v3/systems/npm/packages/<name>/versions/<v>:dependents` returns `404`.

**Trigger**: `/unpublish <fresh-pkg>@<v>`.

**Expected**:
- Phase A.3 detects `HTTP_STATUS = 404` and DOES NOT route into the "API unreachable, STOP" branch.
- Phase A.3 surfaces a non-fatal note: *"deps.dev has not yet indexed `<pkg>` (typically 5–60min after first publish). Treating as 0 dependents — true for any package no consumer has had time to install yet."*
- Phase A.4 renders eligibility with `Status` column suffixed `(deps.dev not yet indexed)`.
- The skill proceeds normally (does NOT block on deps.dev being unindexed).

**Verify**: this is the load-bearing test for the wrong-name fast-cleanup happy path. If S17 fails (skill HARD STOPs on 404), `/unpublish` is broken for the exact use case it was designed to handle.

### S18 — Detached-HEAD STOP (v0.11.0 A1)

**Setup**: any solo-npm-managed repo. Detach HEAD: `git checkout HEAD~1` (or any specific SHA without `-b`).

**Trigger**: `/solo-npm:release` (or any commit/tag-creating skill).

**Expected**: Phase A.1 detects `! git symbolic-ref -q HEAD` and HARD STOPs with: *"detached HEAD detected (currently at <SHA>). Commits or tags created here would become unreachable when you next checkout a branch. Fix: git checkout main, then re-invoke."*

**Verify**: no commits or tags were created. Working tree unchanged.

### S19 — Worktree STOP (v0.11.0 A2)

**Setup**: `git worktree add ../solo-npm-tmp main` and `cd` into the new worktree.

**Trigger**: `/solo-npm:release` or `/solo-npm:init`.

**Expected**: Phase A detects worktree state (`git rev-parse --git-path HEAD` resolves outside `.git/HEAD`) and HARD STOPs with: *"detected git worktree (not the main working directory). solo-npm skills assume the main worktree because .solo-npm/ artifacts and package.json scaffolding decisions are tied to it. Fix: cd $(git worktree list | head -1)."*

**Verify**: no scaffolding written; no commits; main worktree's state.json untouched.

### S20 — git stash pop conflict in /deps rollback (v0.11.0 B2)

**Setup**: contrived. Force a state where `/deps`'s rollback `git stash pop` will conflict — e.g., manually run `git stash` after Phase 4a's snapshot but before its pop completes, then trigger a verify failure.

**Trigger**: `/deps cve-tier-1` against a repo where the test suite will fail post-upgrade.

**Expected**: when the per-batch `/verify` fails and Phase 4d rolls back, `git stash pop` hits a merge conflict. The skill detects the conflict (stderr matches `conflict|CONFLICT|merge`), surfaces conflicted files via `git diff --name-only --diff-filter=U`, and presents three recovery options (resolve-then-drop / discard / re-apply on clean tree).

**Verify**: stash is still on the stack (`git stash list` shows it); no further automatic action was taken; user is unblocked because they got a concrete path forward.

### S21 — Pre-commit hook rollback in /release (v0.11.0 B3)

**Setup**: install a `.git/hooks/pre-commit` that exits non-zero (e.g., always-fails or a real lint hook with violations in CHANGELOG.md).

**Trigger**: `/solo-npm:release`.

**Expected**: Phase C.2 detects the hook rejection (stderr matches `pre-commit hook|hook .* failed|hook declined`), surfaces the hook output verbatim, AND presents three recovery options:
- (a) Address the hook complaint + re-run /release (idempotent)
- (b) Bypass via `git commit --no-verify -m '...'` then resume
- (c) Reset staging via `git reset HEAD -- CHANGELOG.md package.json`

**Verify**: no commit was made; staging area still contains the CHANGELOG/package.json mutations from C.1; user can pick a clear path forward.

### S22 — Concurrent /release lock refusal (v0.11.0 C3)

**Setup**: in terminal A, start `/solo-npm:release` and pause it at Phase B's plan-approval gate (don't pick Proceed yet). In terminal B (same repo cwd), invoke `/solo-npm:release`.

**Expected**: terminal A holds `.solo-npm/locks/<repo-root>.lock` after passing Phase C.0a (this requires terminal A to have approved Phase B already; for the actual race, simulate by manually creating the lockfile with terminal A's PID). Terminal B reaches Phase C.0a and detects the live lock holder, halting cleanly with: *"another solo-npm release is in flight on this repo (PID <pid>). Wait for it to finish, or kill PID <pid> if hung."*

**Verify**: terminal A completes successfully. Terminal B never executed C.1 onwards. After A finishes, the lock file is gone (trap cleanup); B can be re-run.

### S23 — IDENTIFIER typo rejected in /prerelease (v0.11.0 E2)

**Setup**: any repo with a stable version on `@latest`.

**Trigger**: *"start an alpa pre-release line for v2"* (typo: "alpa" instead of "alpha").

**Expected**: Phase 0 extracts `IDENTIFIER=alpa`. Phase 0.5 validation regex `^(alpha|beta|rc|canary|experimental|next|preview|edge)$` rejects it. Skill HARD STOPs with: *"IDENTIFIER='alpa' is not a recognized pre-release identifier. Did you mean 'alpha'? Valid: alpha, beta, rc, canary, experimental, next, preview, edge."*

**Verify**: no `npm version` call; no commits; no tags. The user gets a clear "did you mean" hint and can re-invoke with the correct identifier.

### S24 — H8 rate-limit backoff fires on 429 (v0.12.0 A)

**Setup**: contrived. Mock `npm view` to return HTTP 429 with `npm ERR! 429` in stderr (or wait for natural rate-limit during a portfolio-wide /status on a slow registry).

**Trigger**: `/solo-npm:status` against a portfolio of 5+ packages where one package's registry call returns 429.

**Expected**: that package's call passes through `npm_with_h8_backoff` (Phase −1.9 helper). On 429, the helper retries with exponential delays (1s → 2s → 4s → 8s) plus ±20% jitter, max 4 attempts. If all retries hit 429, the dashboard renders that package's row as `(rate-limited)` rather than `—` — disambiguating "throttled" from "unavailable".

**Verify**: WARN messages "rate-limited on attempt N; retrying in Ks" appear in the trace. The other packages' rows render normally (the failure is per-package, not whole-skill).

### S25 — SSL/TLS error surfaces 4-option remediation (v0.12.0 B)

**Setup**: simulate by setting `GIT_SSL_NO_VERIFY=false` and pointing git at a self-signed remote, OR wait for a real transient SSL_ERROR from github.com (we hit it on the v0.10.1 push).

**Trigger**: any skill that calls `git push` or `curl` to deps.dev/npmjs API.

**Expected**: stderr matches the SSL pattern (`SSL_ERROR_*` / `cert*` / `unable to get local issuer` / `TLSV1_ALERT`). The skill captures the failure, surfaces the standard 4-option remediation:
1. Transient retry hint
2. Corporate-proxy CA bundle config
3. OS CA bundle update commands (brew / update-ca-certificates / update-ca-trust)
4. Last-resort `--insecure` / `GIT_SSL_NO_VERIFY=true` (DEV ONLY)

**Verify**: the user gets actionable remediation, not just a verbatim curl/git stderr dump. Most users hit case 1 (transient) and recover by re-running the skill within 30s.

### S26 — SIGINT mid-skill cleanup (v0.12.0 C)

**Setup**: invoke `/solo-npm:release` and pause it post-Phase-C.5 (after `git tag` ran but before `git push --tags` succeeded). Press Ctrl+C.

**Expected**: the SIGINT trap from `/unpublish` Phase −1.11 fires:
- WARN message "interrupted by user. Cleaning up in-flight state…"
- Detects `LOCAL_TAG_PENDING_PUSH=v<NEXT_VERSION>` is set
- Runs `git tag -d v<NEXT_VERSION>` to remove the local-only tag
- The EXIT trap also fires afterward (lockfile cleanup)
- Final message: "Cleanup complete. Safe to re-invoke the skill."
- Exit code: 130 (standard SIGINT exit)

**Verify**: `git tag --list v<NEXT_VERSION>` returns empty. `.solo-npm/locks/<repo-root-hash>.lock` is gone. Re-invoking `/solo-npm:release` works without tag-collision errors.

### S27 — `.npmrc` API used correctly for scope detection (v0.12.0 E)

**Setup**: project with `.npmrc` containing `@myorg:registry=https://registry.example.com/` set.

**Trigger**: `/solo-npm:init` on this project.

**Expected**: Phase 1a uses `npm config get @myorg:registry` (not grep on `.npmrc`). Returns `https://registry.example.com/`. Phase 1c detects this is already mapped and skips writing.

**Verify**: ad-hoc `.npmrc` grep is NOT invoked (search the trace for `grep .npmrc` patterns — should be absent in v0.12.0). The `npm config get` call is invoked instead. The output handles the "undefined" literal vs empty string ambiguity correctly per the BCL guard.

### S28 — Conventional-commits typo rejected with did-you-mean (v0.12.0 F)

**Setup**: a repo where the commit since the last tag is `breaking: drop Node 18 support` (non-canonical type instead of `feat!:`).

**Trigger**: `/solo-npm:release`.

**Expected**: Phase B.3 type-whitelist check classifies `breaking:` as unrecognized, surfaces:
- WARN "commit <SHA> uses 'breaking:' which is NOT canonical conventional-commits"
- Suggests the canonical alternatives: `feat!:`, `fix!:`, or `BREAKING CHANGE:` footer
- Treats as Other (informational only) — does NOT auto-promote to major bump
- Phase B.4 sees only Other commits → STOPs with "nothing to release"

**Verify**: no version bump derived. User can amend the commit to `feat!:` and re-run /release.

### S29 — gh GraphQL falls back to per-repo on graphql failure (v0.12.0 I)

**Setup**: a portfolio with 25 unique repos. Use a `gh` token without sufficient repo scope (or simulate by malforming the GraphQL query).

**Trigger**: `/solo-npm:status`.

**Expected**: Phase 2 triggers GraphQL fan-in (>20 repos threshold). The single `gh api graphql` call fails (insufficient scope, malformed query, etc.). Skill surfaces "WARN: gh api graphql fan-in failed: <verbatim>. Falling back to per-repo individual gh repo view + gh run list calls." Then proceeds with per-repo fallback, each call wrapped in H8 backoff.

**Verify**: the dashboard still renders (fallback succeeded). User sees the GraphQL warning in the trace + can decide whether to re-run with a higher-scope token.

### S30 — Submodule state STOPs the release (v0.13.0 A)

**Setup**: a repo with `.gitmodules` declaring a submodule. Modify the submodule's working tree without committing the supermodule update (`git submodule status` shows `+<sha>` for the modified submodule).

**Trigger**: `/solo-npm:release`.

**Expected**: Phase A's −1.6b submodule check detects the `+` line. HARD STOPs with: *"$N submodule(s) have local modifications not matching the supermodule's tracked SHA. Either commit the supermodule update OR reset the submodule (`git submodule update --recursive`)."*

**Verify**: no commit, no tag, no publish. Working tree unchanged.

### S31 — Proactive rate-limit slowdown before 429 (v0.13.0 B)

**Setup**: a portfolio of 30+ packages on GitHub. Use a `gh` token close to its 5000/hr quota (most easily simulated by setting `GH_RATE_REMAINING` low manually, or by running `/status --fresh` repeatedly until quota drops).

**Trigger**: `/solo-npm:status`.

**Expected**: `gh_with_rate_tracker` periodically calls `gh api rate_limit`. When `core.remaining < 100`, surfaces *"GitHub API quota low (X remaining); sleeping Ns until reset"* and sleeps. Resumes after the reset. Never hits an actual 429.

**Verify**: trace shows the WARN about quota; total runtime longer than usual but completes successfully without per-package "rate-limited" rows.

### S32 — Shell-safety regex rejects `;` in package name (v0.13.0 E)

**Setup**: invoke `/solo-npm:dist-tag` with prompt: *"add @canary to 1.6.0 on @scope/foo; rm -rf $HOME"*.

**Expected**: Phase 0 extracts `SCOPE=@scope/foo; rm -rf $HOME`. Phase 0.5 regex (`^@[a-z0-9][a-z0-9._-]*/[a-z0-9][a-z0-9._-]*$`) rejects it because the value contains characters outside the allowed set. Phase 0.5b shell-safety check is the second-line-of-defense and would also reject (semicolon + `$`).

**Verify**: HARD STOP with diagnostic naming the offending characters. No `npm dist-tag` call ever runs. The user's HOME directory is intact.

### S33 — CRLF in bin script HARD STOPs verify (v0.13.0 G)

**Setup**: a repo with `bin/cli.js` containing CRLF line endings (e.g., on a Windows checkout with `core.autocrlf=true`). The file's first line is `#!/usr/bin/env node\r\n`.

**Trigger**: `/solo-npm:verify` or `/solo-npm:release` Phase A.2.

**Expected**: Tier 3's CRLF-in-tarball check packs the tarball, extracts to a temp dir, scans `bin/cli.js` for `\r`, finds it. HARD STOPs with: *"❌ CRLF DETECTED IN PUBLISHED EXECUTABLES … This breaks Unix consumers. Fix BEFORE releasing: 1. git config core.autocrlf input 2. git checkout -- <file> 3. Re-run /solo-npm:verify --pkg-check-only."*

**Verify**: no publish. The user gets a clear remediation path. After fixing (`core.autocrlf=input` + re-checkout), the same `/verify` run passes Tier 3.

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
