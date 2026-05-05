# Changelog

## v0.7.0 — quality-of-life polish (no new skills, no scope expansion)

Five enhancements across existing skills + one doc fix. Skill count stays at 12; no new tooling, no CI scripts, no new dependencies. Pure value-add to existing workflows.

### GitHub Release notes auto-generation (`/release`, `/prerelease`, `/hotfix`)

`/release` Phase C.7.5, `/prerelease` Phase C.7.5, and `/hotfix` Phase E.6 now invoke `gh release create` after the registry verification step, using the just-prepended `CHANGELOG.md` entry as the release notes body. Repo watchers get a rich GitHub notification on every release; the GitHub Release page is no longer empty.

Variants:
- Pre-releases (`/prerelease` START + BUMP) get `--prerelease`. PROMOTE results in a stable release; no flag.
- Legacy-line hotfixes (target dist-tag = `@v<TARGET_MAJOR>`) get `--latest=false` so the GitHub Releases UI keeps the "Latest" badge on the current main-line stable.
- Hotfixes attach to the maintenance branch via `--target <TARGET_MAJOR>.x`.

Pre-flight checks `gh auth status`. If `gh` is unavailable or unauthenticated, the step is **gracefully skipped** with a non-fatal warning — the npm publish + git tag still go through; only the GitHub Release page stays empty.

CHANGELOG entry extraction uses an awk-based extractor that finds the matching version header and stops at the next `## v` line. Robust to format variations.

### Bundle-size Tier 4 in `/verify` Step 5

A 4th tier in pkg-check tracks tarball size over time:

- After Tier 3's `npm pack --dry-run` parses `unpackedSize`, look up the previous published version's recorded size in `.solo-npm/state.json#pkgCheck.lastSize` keyed by `<pkg>@<version>`.
- Compute `delta = (current - prior) / prior * 100`.
- `delta > +25%` → warning + top-5-largest-files breakdown for investigation.
- `delta < -25%` → info (significant shrinkage; usually intentional).
- First-run packages have no baseline; cache builds organically over releases.
- Major version bumps (1.5.0 → 2.0.0) suppress the warning — major versions are expected to grow.

The cache is updated by `/release` Phase C.7.6, `/prerelease` Phase C.7.6, and `/hotfix` Phase E.7 after each successful publish (writes the new version's `unpackedSize` to `pkgCheck.lastSize`).

Optional integration with `size-limit`: if the repo has a `size-limit` config, also runs `pnpm dlx size-limit` and surfaces failing budgets as Tier 4 errors. Opt-in via config presence; we don't add the dep.

No auto-fix — size deltas need human investigation.

### `/status` Portfolio health section

`/status` Phase 2 now also reads `.solo-npm/state.json#audit` and `#pkgCheck`. Phase 3 renders a new "Portfolio health" section after Maintenance lines:

```
Portfolio health:
  Audit:      Tier-1: 0  Tier-2: 0      (last scan: 2d ago)
  Pkg-check:  errors: 0  warnings: 3    (last check: 6h ago)
  Action:     none — all clean.
```

Cache-stale cells substitute *"(stale — run `/audit` to refresh)"*. Missing-cache cells substitute *"(no scan yet — run `/solo-npm:audit`)"*.

Default behavior: zero npm calls; pure cache reads. Optional `--fresh` flag forces re-run of `/audit` + `/verify --pkg-check-only` to populate fresh values.

This makes `/status` a one-stop morning health check — replaces the routine of invoking `/status` then `/audit` then `/verify` separately.

### `/init` Phase 2 guided initial-publish

The v0.6.x STOP-and-resume pattern (*"go publish manually then re-invoke /init"*) is replaced with a guided flow:

1. **Auth check**: `npm whoami`. If not authenticated, surface foolproof `npm login` handoff (numbered web 2FA steps).
2. **Per-package publish gate** via `AskUserQuestion`: *"Run `npm publish --provenance=false --access public` now to claim the package name `<NAME>`?"* Options: Yes — publish now / I'll publish manually / Abort.
3. **Execute on Yes**: agent runs the publish; surfaces success or verbatim npm error on failure.
4. **Continues to Phase 3** trust config without requiring re-invocation.

Keeps the user-in-the-loop AskUserQuestion gate (publish is destructive — claims the name on registry) while removing the awkward stop-and-resume. Manual path still available for users who prefer to run `npm publish` themselves.

### `docs/prompts.md` refresh

- Fixed line 244–248 ("forward-port is currently manual" — automated since v0.6.0 via `/hotfix` Phase F.5 cherry-pick).
- Added 3 new v0.7.0 scenarios: GitHub Release page auto-fill, bundle-size regression catch, one-stop `/status` health check.

### Out of scope (rejected during planning)

- `/rollback` skill — npm doesn't unpublish; "fix and bump" is canonical recovery for solo-dev. Existing `/dist-tag repoint` + `/deprecate` covers the rare slow-fix scenario.
- `/solo-npm:doc-sync` skill / `scripts/check-docs.ts` Node tool / Claude-in-CI E2E tests — drift-prevention infrastructure for a once-a-quarter bug class. Cost-benefit didn't pencil out for a solo-dev repo. Accept that drift is a discipline concern.

### Migration

No consumer-repo changes needed. New behavior is additive:

- GitHub Release creation skips silently if `gh` is unavailable (existing release flow continues).
- Bundle-size Tier 4 records first-run baselines silently; warnings start firing on the second published version per package.
- `/status` Portfolio health section renders "(no scan yet)" hints if the cache is empty; populates after first `/audit` and `/verify --pkg-check-only` runs.
- `/init` Phase 2 guided publish: existing behavior fully preserved via the "I'll publish manually" option.

## v0.6.1 — pkg-check enhancement: publint + tarball-content audit + .gitignore-vs-files divergence

Closes the "MED priority partial" pre-publish manifest validation gap from `docs/npm-coverage.md`. The v0.6.0 `/verify` Step 5 spec was prescriptive prose that Claude read and applied manually — slow, inconsistent, didn't cover everything. v0.6.1 replaces the manual checklist with a 3-tier check backed by industry-standard tooling.

### Step 5 — pkg-check rewritten as 3 tiers

**Tier 1 — `publint`** (canonical manifest validator). Runs via `pnpm dlx publint@0.3 --strict --json` (npm/yarn fallbacks). Catches malformed `exports`, missing `types` for TS, deprecated fields, `MAIN_FIELD_NOT_FOUND`, `EXPORTS_GLOB_NO_PATTERN_MATCH`, and ~30 other rules. Severity maps to solo-npm (`error` halts release-path; `warning` surfaced).

**Tier 2 — manual checks** (publint blind spots). Four checks:

1. LICENSE file presence in package dir — auto-fix offers MIT scaffold.
2. README.md exists, non-trivial — auto-fix offers stub from `name` + `description`.
3. `engines.node` ↔ `.nvmrc` consistency.
4. **NEW: `.gitignore` vs `files`/`exports` divergence**. Catches the canonical "I shipped an empty tarball because `dist/` is gitignored AND not in `files` allowlist" gotcha. Parses `.gitignore` (best-effort: simple patterns; v1 skips negation rules). Walks publish-critical paths from `package.json` (`main`, `module`, `types`, `exports.*`, `bin.*`); flags errors when matched + not-allowlisted. Auto-fix offers `files: [<gitignored-but-needed paths>, "package.json", "README.md", "LICENSE"]`.

**Tier 3 — `npm pack --dry-run` tarball-content audit (NEW)**. The actual tarball gets inspected for:

- **Secrets** (`.env`, `*.key`, `*.pem`, `id_rsa*`, `*credentials*`, `secrets*.json`) → **HARD STOP** with verbatim remediation steps (`git rm --cached`, `.gitignore`/`.npmignore`, **rotate any leaked credentials**). NO auto-fix — too dangerous to attempt.
- Lockfiles, test files, editor configs, OS junk → warnings + auto-fix offer for `files` allowlist.
- Missing `LICENSE`/`README.md` from the actual tarball → warnings.
- `unpackedSize > 5 MB` → warning + top-10-largest-files breakdown.

Pattern excludes for false positives: `.env.example`, `.env.sample`, `.env.template` allowed (they're intentional documentation).

### `/init` Phase 1d — pre-flight manifest validation (NEW)

After Phase 1c scaffolding but BEFORE the existing commit gate (renamed to 1e), `/init` invokes `/verify --pkg-check-only`. Auto-fix loop applies fixes; unresolvable errors STOP. This closes the original gap from `docs/npm-coverage.md`: *"/init scaffolds publishConfig but doesn't validate everything"* — now `/init` never claims "complete" while leaving the manifest non-publish-ready.

### `/verify --pkg-check-only` mode (NEW)

Skips lint/typecheck/test/build; runs only Step 5. Used by `/init` Phase 1d and by users wanting fast publish-readiness check without the full suite.

### Hash-based caching

`pkgCheck` cache section in `.solo-npm/state.json` keyed by `{ packageJsonHash, gitignoreHash, distOutputContents }`. Hash invalidation; no TTL. Repeat `/verify` runs hit cache → `PKG_CHECK_OK_CACHED` (near-zero wall-time).

### `docs/npm-coverage.md` comprehensive refresh

Resolved the staleness audit: §2 inventory (9 → 12 skills + new `/dist-tag`/`/deprecate`/`/owner` rows), §3 line 152 ("✗ partial — MED" → ✓ shipped), §4.1–§4.5 past-tensed with shipped callouts, §6.1 README opener history (3 iterations), §6.2 table updated with current statuses, §6.3 capability→skill diagram noted as added-then-removed, §6.4 cosmetic fixes, §8 migration marked complete with historical sequence preserved, NEW §10 explaining the "operator capability" → "npm command" terminology shift in user-facing copy.

The internal contradiction is resolved (TL;DR no longer disagrees with the matrix).

### Migration

No consumer-repo changes. publint and `npm pack --dry-run --json` are invoked ad-hoc from the consumer's package manager (no new runtime dep on solo-npm itself). Existing consumer wrappers (`.claude/skills/release/SKILL.md`, `.claude/skills/verify/SKILL.md`) continue to work — they invoke `/solo-npm:verify` which now runs the enhanced Step 5 transparently.

If a consumer's `.gitignore` excludes their build output AND they don't have a `files` field, the first `/verify` run after upgrading will surface the divergence as an error with auto-fix offer. Single approval restores correctness.

## v0.6.0 — npm operator coverage: three new skills + /verify pkg-check + framing alignment

Major scope expansion driven by the npm operator-capabilities analysis (`docs/npm-coverage.md`). Three new skills close the four real gaps identified there; `/verify` gains a manifest-completeness step; the README narrative is reframed around the npm-operator-capability framing.

### New skill: `/solo-npm:dist-tag` (#10)

Manages `npm dist-tag` post-publish — the lever for channel cleanup, rollback after a botched release, opt-in channels (`@canary`, `@experimental`), and bulk portfolio operations. Five operations:

- **add** — point a tag at a specific version
- **rm** — remove a tag (rejects `@latest` removal; suggests `repoint` instead)
- **ls** — list current dist-tags across the portfolio (read-only, no gate)
- **repoint** — move an existing tag to a different version
- **cleanup-stale** — bulk-remove `@next` where it points at a pre-release whose stable equivalent has shipped to `@latest`

Foolproof `npm login` handoff in Phase A — `npm dist-tag` mutations are NOT covered by OIDC Trusted Publishing (only `npm publish` is). 200ms inter-call backoff for bulk operations to avoid registry rate limits. Phase D verifies each mutation landed.

Composes with `/status` (acts on stale-`@next` warnings via the new explicit hint) and `/prerelease` PROMOTE (optional `AskUserQuestion` to chain into cleanup-stale after stable promote).

Triggers from prompts like *"cleanup stale @next"*, *"repoint @latest to 1.5.2 — 1.6.0 has a bug"*, *"add @canary to 1.6.0-experimental.2"*, *"what dist-tags are set on my packages"*.

### New skill: `/solo-npm:deprecate` (#11)

Marks `npm` package versions deprecated (or undeprecates) with a custom message. Reversible. Safer than `npm unpublish` (which has a 24-hour hard window and breaks consumer lockfiles). Two operations:

- **deprecate** — apply a message to a version, range, or every version satisfying a semver expression
- **undeprecate** — lift the deprecation (set message to `""`)

Range patterns: single version (`1.6.0`), major (`1.x`), comparator (`<2.0.0`, `>=1.0.0 <1.5.0`). Phase A explicitly rejects unbounded ranges (`*`, `x`, empty) for safety — must specify a concrete bound.

Phase 0 prompt-context extraction handles natural language like *"deprecate all 1.x with message 'v1.x is EOL — migrate to v2'"*, *"mark 1.6.0 as do-not-use because of data bug"*, *"undeprecate 1.5.0 of @ncbijs/eutils"*.

Foolproof `npm login` handoff (npm deprecate is not OIDC-covered). 200ms backoff. Already-in-target-state versions are silently skipped.

Composes with `/release` Phase G (post-major auto-chain offer) and `/audit` Phase 5 (CVE response option).

### New skill: `/solo-npm:owner` (#12)

Manages npm package maintainers at portfolio scale. Three operations:

- **ls** — list owners across the portfolio (read-only audit; highlights bus-factor risk where a package has only one owner)
- **add** — add an npm user as a maintainer
- **rm** — remove a maintainer (rejects sole-owner removal; warns on self-removal)

Phase 0 extracts intent from prompts like *"add @backup-maintainer to all my packages"*, *"show me who can publish each package"*, *"remove @old-collaborator from @ncbijs/eutils"*.

Foolproof `npm login` handoff. Bulk add across 43 packages happens with one `AskUserQuestion` gate; manual `npm owner add` × 43 was the previous workflow.

### `/verify` Step 5: pkg-check (manifest completeness validation)

`/verify` now has a 5th step that validates `package.json` + `LICENSE` + `README.md` completeness across the workspace.

**Errors** (publish-critical): `name`, `version`, `license`, `exports` OR `main`, contradictory `private` + `publishConfig.access` combo.

**Warnings**: `description`, `keywords`, `repository.url`, `homepage`, `bugs.url`, `engines.node`, `LICENSE` file, non-empty `README.md`.

**Severity by context**: standalone `/verify` (mid-development) surfaces errors as warnings (don't halt the verify run). Release-path `/verify` (called from `/release` Phase A.2, `/prerelease` Phase A, `/hotfix` Phase A) escalates errors to halt with auto-fix offers before STOP.

**Auto-fix scaffolds**: `repository.url` derivable from `git remote get-url origin`, MIT `LICENSE` scaffold with copyright derivation, minimal `README.md` stub. Each accepted auto-fix becomes a separate `chore(pkg): <fix>` commit.

### Composition wiring

Three new chains across existing skills:

- **`/release` Phase G (NEW)** — fires only on major version bumps. `AskUserQuestion` gate offers to chain into `/solo-npm:deprecate` for the previous major (e.g., after 2.0.0 ships, deprecate 1.x with "v1.x is EOL — migrate to v2"). Default option is "Defer" — user-in-the-loop preserved.
- **`/audit` Phase 5 gate extended** — adds "Deprecate affected versions" option. Useful when the upgrade path is blocked but consumers should be warned off vulnerable versions. Pre-fills `RANGE` from advisories' `vulnerable_versions` and `MESSAGE` from advisory metadata.
- **`/prerelease` PROMOTE Phase E (NEW)** — optional `AskUserQuestion` after promote to chain into `/solo-npm:dist-tag cleanup-stale` and remove the now-superseded `@next`.

`/status`'s stale-`@next` warning now includes an explicit `→ /solo-npm:dist-tag cleanup-stale` hint instead of just suggesting the manual `npm dist-tag rm` command.

### README narrative reframe

The README is reorganized around the **npm-operator-capability framing**:

- **New opener**: *"AI skills for every npm operator capability a solo dev actually uses — plus the safety gates and infrastructure around them."* (Replaces "The full npm publishing lifecycle for AI-driven solo developers.")
- **New "npm coverage" section** with a flat table mapping each npm capability to its skill (or non-goal). Honest about what's in and what isn't.
- **New "Two kinds of skills" subsection** in Architecture distinguishing operator skills (orchestrate npm CLI) from safety + infrastructure skills (`/verify`, `/init`).
- **New capability → skill diagram** (Mermaid, dark-red palette, mirrored to `resources/diagrams/capability-to-skill.mmd`).
- **Lifecycle diagram updated** to add the three new skills in the OPERATE lane, with new edges for the auto-chains (`audit -.-> deprecate`, `status -.-> distTag`, `release -.post-major.-> deprecate`).
- **"Out of scope (deliberate)" extended** with the codified non-goals from the analysis: `npm unpublish`, `npm token`, `npm hook`, `npm org`/`team`, `npm pack`/`search`/`star`/`fund`, `npm sbom`, `npm access` flip post-publish.

The banner SVG's tagline is updated to match: *"AI skills for every npm operator capability solo devs use"*.

### docs

- `docs/npm-coverage.md` updated to reflect shipped status (Tier 1 + 2 items now ✓ shipped).
- `docs/prompts.md` extended with prompt examples for the three new skills + four new compound workflows: major-version transition with auto-chain to /deprecate, CVE-with-blocked-upgrade → /deprecate, botched-release recovery via /dist-tag + /deprecate, bus-factor audit via /owner.

### Migration

No consumer-repo changes needed for v0.6.0. The new skills auto-detect their auth requirements and surface foolproof `npm login` instructions when needed. Existing /release, /verify, /audit invocations continue to work; the new chains and gates are additive.

If you want to take advantage of the auto-chain offers (post-major deprecation, CVE deprecation, post-promote cleanup), no action is needed — they'll fire automatically the next time you hit those scenarios.

## v0.5.5 — Polish iteration: auto-chain consistency, multi-channel awareness, cherry-pick automation, README rework

Closes UX gaps from v0.5.4 across five themes — auto-chain consistency for stale `release.yml`, dist-tag awareness in `/status`, maintenance-lines visibility, cherry-pick automation in `/hotfix`, and changelog aggregation on PROMOTE. Plus a comprehensive README rework with visual Mermaid diagrams.

### Auto-chain `/init --refresh-yml` (#3)

`/solo-npm:prerelease` and `/solo-npm:hotfix` Phase A previously STOPped with a "run /solo-npm:init to refresh" message when `release.yml` lacked the dist-tag detection step. They now auto-chain after a single `AskUserQuestion` gate:

- `Refresh release.yml and continue` → invoke `/solo-npm:init --refresh-yml`, which performs targeted surgery (insert the `Detect dist-tag` step before `pnpm publish`; modify the publish step to use `--tag ${{ steps.dist.outputs.tag }}`), commits as a self-contained `chore: refresh release.yml for dist-tag detection`, then re-enters the caller's Phase A.
- `Abort` → graceful stop.

`/solo-npm:init --refresh-yml` is idempotent — no-op when `release.yml` is already current. STOPs cleanly with manual-edit guidance when the workflow is too divergent for safe surgery.

This restores the auto-chain pattern that v0.5.3 introduced for trust setup; pre-release and hotfix flows no longer break user momentum with manual remediation.

### Pre-release / dist-tag awareness in `/status` (#5)

`/solo-npm:status` now fetches `dist-tags` per package (via `npm view <pkg> --json`) and renders divergent channels:

- **Active divergence** (`@next` is pre-release-shaped, differs from `@latest`): two rows per package — first for `@latest`, continuation row for `@next` with separate Drift count.
- **Stale `@next`** (`@next` points at a version whose stable equivalent has shipped to `@latest`): single row + warning above the table suggesting `npm dist-tag rm` for registry hygiene.
- **Single channel** (typical case): one row, no Channel suffix change needed.

Action hints extended:
- Drift on `@next` row → "→ /solo-npm:prerelease (Bump or Promote)"
- Active `@next` with no drift → "→ /solo-npm:prerelease to bump or promote"

`/solo-npm:audit` gained a one-line note when both channels are affected by an advisory.

### Changelog aggregation on PROMOTE (#6)

`/solo-npm:prerelease` C.1 PROMOTE branch now regenerates the stable changelog entry from `git log <last-stable-tag>..HEAD` — covering all betas in the line + the promote commit — and prepends it to `CHANGELOG.md`. Per-beta entries are preserved below the new stable entry as engineer-facing historical record.

Result: end users on `@latest` get one comprehensive `1.6.0` entry describing all changes since `1.5.0`, instead of seeing only the trivial delta vs `1.6.0-beta.3`. Dedup is automatic — each commit appears once in the range.

START and BUMP paths are unchanged.

### Cherry-pick automation in `/hotfix` (#7)

Two new sub-flows in `/solo-npm:hotfix`, sharing one conflict-handoff helper:

**Backport (`--cherry-pick <sha>` flag or "backport this commit" / "cherry-pick abc1234 to v1" hints)**: Phase 0 detects the SHA. Phase D.1 runs `git cherry-pick <sha>` on the maintenance branch instead of asking for a fix description. On clean merge → `/verify` → Phase E ships the patch.

**Forward-port (NEW Phase F.5)**: after Phase F's `git checkout main`, the skill gates with `AskUserQuestion`:
- `Yes — cherry-pick and ship` → `git cherry-pick <hotfix-sha>` → chain to `/release`.
- `Yes — cherry-pick only` → cherry-pick lands on main; user reviews and runs `/release` separately.
- `No — skip` → final summary with manual instructions.

**Conflict handoff (shared)**: when `git cherry-pick` exits non-zero, the skill surfaces conflicted files and three resolution paths (resolve manually, abort, or describe the resolution). Skill exits cleanly without auto-merging — resolution depends on intent (e.g., which side of a refactor wins), and auto-merging is more dangerous than asking.

This removes the "manual `git cherry-pick` between hotfix and `/release`" friction that broke the AI-driven principle for backport/forward-port workflows.

### Maintenance lines section in `/status` (#8a)

`/solo-npm:status` discovers `<major>.x` branches via `git ls-remote --heads origin '*.x'` and renders a new section (only when ≥1 branch exists):

```
Maintenance lines:
  - 1.x (last patch v1.5.2, 14d ago) — current stable major
  - 0.x (last patch v0.9.5, 90d ago) — legacy, dist-tag @v0
```

Per-line classification compares the major to `LATEST_STABLE_MAJOR`:
- `<major> >= LATEST_STABLE_MAJOR` → "current stable major"
- `<major> < LATEST_STABLE_MAJOR` → "legacy, dist-tag @v<major>"

Action hint added for legacy lines whose last patch is > 365 days old: *"→ Consider archiving `<major>.x` — last patch was 11mo ago"*.

### README rework

The root README is now organized around the AI-skills-as-npm-operator-capabilities philosophy with:

- Visual Mermaid lifecycle diagram (BOOTSTRAP / PER RELEASE / LIFECYCLE TRANSITIONS / OPERATE) using a dark-purple palette.
- Per-skill detail subsections covering purpose, steps it covers, and triggers-from natural-language prompts.
- Visual diagrams for: agent-skills composition (DEVELOPMENT vs RELEASE+MAINTAIN boundary, with explicit composition points), release anatomy (auto-chains visualized), plugin baseline + thin wrapper pattern.
- New "Out of scope (deliberate)" section codifying non-goals (#1, #2, #4, #8b from the v0.5.5 deferred-items analysis) with revisit-trigger criteria.
- Diagnostic prompts table mapping symptoms to skills.

LLM-agent-friendly throughout — structured tables and explicit trigger-phrase mappings.

### Migration

No consumer-repo changes. `/solo-npm:init --refresh-yml` is invoked on-demand from `/prerelease` or `/hotfix` Phase A only when needed; existing repos continue to work without action.

## v0.5.4 — Universal /release entry; pre-release + hotfix as 1st-class skills; agent-skills composition

Major restructure of the release lifecycle. Two new skills (8th and 9th), a friction-minimisation pass on `/release`, dist-tag-aware publish for the workflow templates, and explicit composition with `addyosmani/agent-skills` for development work.

### `/release` simplified + universal entry

- **B.5 prompt simplified** to two options: `Proceed with v{NEXT} / Abort`. Dropped `Override version` (use prompt context — say "release v0.5.4" and the skill picks it up) and `Edit changelog` (follow up with a `docs(changelog):` commit if needed). Pre-release branch removed (now lives in `/solo-npm:prerelease`).
- **First-release flow skips B.5 entirely.** B.1's "First release" pick is the only prompt; changelog renders as visible chat output; Phase C executes directly. Drops first-release from 2 clicks to 1.
- **Auto-chains to `/solo-npm:prerelease`** when `package.json#version` is pre-release-shape. The user types `/release`; skill resolves to the right flow without redirect messages.
- **New Phase 0 prompt-context extraction**: if the user's prompt names a specific version ("release v0.5.4"), pre-fill `NEXT_VERSION`. If auto-bump differs, B.5 surfaces both as a 3-way choice.
- **New Phase A.5 cache-aware audit check**: reads `.solo-npm/state.json#audit`; STOPs if `tier1Count > 0` with directive to fix via `/solo-npm:deps`. Zero-latency on the common path.

### New skill: `/solo-npm:prerelease` (8th)

First-class command for the pre-release lifecycle:

- **START a line** (current version is stable): asks for identifier (alpha/beta/rc) + base bump (patch/minor/major). Phase 0 pre-fills from prompt ("start a beta for v2" → identifier=beta, base=major; both questions skipped). Agent edits package.json to `<bumped>-<id>.0`, commits, tags, publishes to `@next` dist-tag.
- **BUMP counter** (current version is pre-release): one click to `1.1.0-beta.1` → `1.1.0-beta.2`.
- **PROMOTE to stable** (current version is pre-release): one click to `1.1.0-beta.n` → `1.1.0`. Publishes to `@latest`.

User never opens `package.json`, never runs `npm version`, never touches the version string. AskUserQuestion gates with structured options; agent handles all code edits.

### New skill: `/solo-npm:hotfix` (9th)

First-class command for backward maintenance hotfixes on a `<major>.x` branch:

- **Phase 0 prompt-context extraction**: "Hotfix the v1 rate limiter — it crashes on 429" pre-fills target major (`1`) and fix description (`rate limiter crashes on 429`); both questions skipped.
- **Phase B branch management**: agent creates `<major>.x` branch from the most recent `v<major>.*` tag (first hotfix) or checks out + pulls (subsequent hotfixes). Pushes the branch upstream automatically.
- **Phase C dynamic dist-tag**: `@latest` if the maintenance major is still the current stable line; `@v<major>` (legacy tag) if a newer major has already shipped. Sets `package.json#publishConfig.tag` accordingly so the registry doesn't get clobbered.
- **Phase D composition with agent-skills**: when `addyosmani/agent-skills` is installed, the actual code fix delegates to `/agent-skills:debugging-and-error-recovery` (reproduce → localise → reduce → fix → guard methodology). Falls back to general agent code-editing if not installed.
- **Phase F return to main**: after the patch ships, agent switches back. Maintenance branch persists on origin for future hotfixes.

### `release.yml` template gains dist-tag awareness

`init.md` scaffolds `release.yml` with a 3-layer dist-tag detection step: explicit `package.json#publishConfig.tag` (highest, used by hotfix on legacy lines) → version-shape (`*-id.n` → `next`) → default `latest`. Single primitive supports stable releases, pre-releases, and hotfixes correctly without per-skill workflow files.

### `.solo-npm/state.json` cache schema extended

`audit` section added alongside `trust`:

```json
{
  "version": 1,
  "trust": { "configured": [...], "lastFullCheck": "...", "ttlDays": 7 },
  "audit": { "tier1Count": 0, "tier2Count": 0, "lastFullScan": "...", "ttlDays": 1 }
}
```

`/release` Phase A.5 reads `audit`; `/solo-npm:audit` Phase 6 writes it; `/solo-npm:deps` Phase 7 updates it after fixing tier-1 advisories. `/solo-npm:status` reads it for the dashboard.

### Composition with `addyosmani/agent-skills`

solo-npm's scope is **release infrastructure + publishing lifecycle**. Development work (writing code, debugging, code review) is delegated to `addyosmani/agent-skills` when installed:

- `/solo-npm:hotfix` Phase D → `/agent-skills:debugging-and-error-recovery` for the bug fix.
- `/solo-npm:deps` verify-failure handler → `/agent-skills:debugging-and-error-recovery` to triage broken upgrades.

solo-npm works standalone; agent-skills composition is opt-in. Recommended setup: install both. README's new "Scope and partners" section documents the boundary.

### Friction-minimisation: prompt-context extraction

Four skills (release, prerelease, hotfix, deps) now read the user's invoking prompt for hints (version numbers, identifiers, fix descriptions, target majors, dep names, risk-tolerance phrases) and pre-fill subsequent questions. *"Start a beta for v2"* skips both START questions in prerelease. *"Update what's safe"* skips the tier prompt in deps. *"Hotfix the v1 rate limiter"* skips both hotfix prompts. Bare invocations still ask everything.

### Description tuning

All 9 commands' frontmatter `description` fields tuned for natural-language matching — common verbs ("ship", "cut", "ship a beta", "patch the previous major") added to improve Claude's auto-invocation.

### Other changes

- `init.md` scaffolds `.solo-npm/state.json` with new `audit` section.
- New "Diagnostic prompts" section in README covers symptom-based prompts ("why is my CI failing?") and how Claude routes them.
- `marketplace.json` and `plugin.json` descriptions updated to mention 9 commands and the agent-skills composition.

### Migration

No consumer changes needed. Existing `release.yml` files in consumer repos lack the dist-tag detection step but only matter if/when the consumer first invokes `/solo-npm:prerelease` or `/solo-npm:hotfix` — the pre-flight checks surface a clear stop with the remedy ("run `/solo-npm:init` to refresh `release.yml`").

If you previously had `package.json#scripts.npm-trust:setup`, it was already removed in v0.5.3. No re-removal needed.

## v0.5.3 — Auto-chain on trust gap; foolproof manual handoffs; trust-state cache

Three coordinated changes that codify the "skills are operators, not
advisors" principle and dramatically reduce per-release latency for
monorepos with many packages.

### Auto-chain on trust/init gaps

`/release` now auto-invokes `/solo-npm:trust` (or `/solo-npm:init`)
when Phase A.3's check detects a fixable gap, instead of pausing to
ask the user. The agent does NOT surface alternative invocation paths
(e.g., the `pnpm npm-trust:setup` script). One path, the skill,
executes.

- `release.md`: new "Principle: skills are operators, not advisors"
  section. Phase A.3 restructured into Cold path / Targeted path /
  Hot path (cache-aware) and Auto-fix / Hard-stop / Informational
  (issue codes). `WORKFLOWS_NONE` and "trust missing" cases now
  auto-invoke the relevant sibling skill rather than stopping with
  a "see /solo-npm:trust" suggestion.
- `trust.md`: Steps 2 (GitHub repo) and 3 (publish workflow)
  auto-confirm when unambiguous; only prompt when ≥2 plausible
  candidates remain.

Minimum-friction shape codified: **one `AskUserQuestion` per
destructive operation, max** — trust's configure gate + release's
approval gate. Everything fixable auto-chains.

### Foolproof manual handoffs (npm login + web 2FA)

The previous trust run failed because the agent tried to execute
`npm-trust --auto --only-new ...` inside its own non-interactive bash
subprocess. That command needs the npm web 2FA flow which can't be
driven from a subprocess. The agent timed out, fumbled with broken
alternative `npm trust github` invocations, and only after several
failed attempts instructed the user to run the command manually —
buried under failure noise.

`trust.md` Steps 7 and 9 now hand off to the user's terminal **early
and unambiguously** with verbatim numbered step-by-step instructions,
including:

- The exact command to run (`npm login`, then the configure command).
- What text the user will see in their terminal (verbatim code blocks).
- Each click/keystroke to perform in the browser ("Press ENTER",
  "Check the **'Skip 2FA for the next 5 minutes'** checkbox", "Click
  the **'Authenticate'** button").
- The expected end state per step.
- The exact reply the user should give back to the chat (`continue` /
  `done`).
- Notes for common failure modes (`EOTP` errors, "0 configured, N
  already set", per-package failures).

The agent **never spawns a Bash subprocess for the configure command**
— eliminating the previous failure mode entirely.

### Trust-state cache (`.solo-npm/state.json`)

For monorepos with many packages, `npm-trust --doctor` iterates ~3 npm
calls per package on every release — slow and mostly redundant once
trust is established. New cache file at `.solo-npm/state.json`
(committed; non-secret) tracks the trust-configured set with a 7-day
TTL.

Phase A.3 now branches on cache state:

| Branch | Condition | npm calls |
|---|---|---|
| **Hot path** | Cache fresh, no new packages | 0 — env-only quick checks |
| **Targeted path** | Cache fresh, new packages added | ~3 per *new* package |
| **Cold path** | Cache stale or missing | Full `--doctor` run, repopulates cache |

For ncbijs (43 packages), this drops daily release time from ~30s
trust check to ~0s. New packages added since the last release are
auto-detected and configured via `--packages <new>` targeted check;
the rest of the package set is skipped.

To force a full re-verify: delete `.solo-npm/state.json`.

- `release.md`: cache-aware Phase A.3 branching, cache update on
  Phase C.6 (CI green) success.
- `trust.md`: Step 11 (verify) updates the cache after configure.
- `init.md`: scaffolds `.solo-npm/state.json` with empty cache;
  drops the (parallel-but-redundant) `package.json#scripts.npm-trust:setup`
  scaffolding.
- `status.md`: reads the cache for the Trust column instead of
  calling `--doctor`. Surfaces stale-cache warning when applicable.

### Migration

Existing consumers (rfc-bcp47, ncbijs, etc.) should:

1. Pull `solo-npm@v0.5.3` (`/plugin marketplace update gllamas-skills`).
2. Optionally drop `package.json#scripts.npm-trust:setup` (no longer
   needed; `/solo-npm:trust` is the canonical entry point).
3. Create `.solo-npm/state.json` populated with their currently
   trust-configured packages (or run `/release` once to let the
   cold-path doctor populate it).

## v0.5.2 — Strip `name:` frontmatter from commands to match addyosmani

Even after moving entries to `.claude/commands/<name>.md` in v0.5.1,
Claude Code's autocomplete still rendered them with the skill format
(`/<name> (<plugin>)`) instead of the command format
(`/<plugin>:<name>`). Comparing our command files against
addyosmani/agent-skills's `.claude/commands/spec.md`, the difference
was in the frontmatter:

- addyosmani: `description: <one-line>` only.
- ours: `name: <name>` + `description: >` (block scalar, multi-line).

The `name:` field is for skill folders (`skills/<name>/SKILL.md` where
the directory name is the canonical name). For commands (flat .md
files), the filename IS the name and the `name:` field signals "this
is a skill" to Claude Code's parser, triggering the skill display
format even though the file lives in `.claude/commands/`.

This release strips the `name:` field from every command and inlines
the description as a one-line string, matching addyosmani's shape
exactly.

## v0.5.1 — Move skills → commands for namespaced autocomplete

Claude Code's autocomplete renders **plugin commands** (in `.claude/commands/`)
as `/<plugin>:<name>` literally in the dropdown — exactly the
`/agent-skills:spec` style of `addyosmani/agent-skills`. Plugin **skills**
(in `skills/<name>/SKILL.md`) render as `/<name> (<plugin>)` in the
dropdown — same canonical invocation, different display.

This release relocates the 7 entries from `skills/<name>/SKILL.md` to
`.claude/commands/<name>.md` so they show up as `/solo-npm:init`,
`/solo-npm:release`, etc. in the autocomplete dropdown — matching
addyosmani's DX.

- Moved `skills/init/SKILL.md` → `.claude/commands/init.md`
- Moved `skills/release/SKILL.md` → `.claude/commands/release.md`
- Moved `skills/verify/SKILL.md` → `.claude/commands/verify.md`
- Moved `skills/trust/SKILL.md` → `.claude/commands/trust.md`
- Moved `skills/status/SKILL.md` → `.claude/commands/status.md`
- Moved `skills/audit/SKILL.md` → `.claude/commands/audit.md`
- Moved `skills/deps/SKILL.md` → `.claude/commands/deps.md`
- Removed empty `skills/` directory.
- Added `"commands": "./.claude/commands"` to `.claude-plugin/plugin.json`.

The functional behavior is identical (per Claude Code docs, "commands and
skills work the same way" and "support the same frontmatter"). This is a
pure display fix.

## v0.5.0 — Complete lifecycle plugin (7 skills)

**Major restructure: solo-npm now owns the full AI-driven solo-dev npm
publishing lifecycle in seven skills, organized by phase:**

- **Bootstrap**: `init`, `trust`
- **Per-release**: `release`, `verify`
- **Portfolio operations**: `status`, `audit`, `deps`

**Breaking changes:**

- **Renamed `setup` → `trust`.** The OIDC trust skill (previously in
  the [`gagle/npm-trust`](https://github.com/gagle/npm-trust) repo) has
  moved into solo-npm and is renamed for clarity. Invocation:
  `/solo-npm:trust` (was `/npm-trust:setup`). Skill content is
  unchanged.
- **`init` rewritten as umbrella.** Phase 1 scaffolds GitHub-side files
  (release.yml + publishConfig + .nvmrc + consumer wrappers +
  `.claude/settings.json`); Phase 2 gates on first manual publish if
  package isn't on npm yet; Phase 3 chains into `/solo-npm:trust` for
  OIDC config; Phase 4 done. Idempotent — safe to re-invoke.
- **`release` and `verify` reframed as wrappable baselines.** Their
  `description` fields explicitly tell Claude to prefer wrappers when
  present. Consumer repos are expected to use thin wrappers in
  `.claude/skills/<name>/SKILL.md` to add repo-specific narrative.
  Daily DX is `/release` (wrapper) and `/verify` (wrapper); the
  marketplace baselines (`/solo-npm:release`, `/solo-npm:verify`) are
  for direct/unmodified invocation.
- **Three new skills** round out the lifecycle:
  - `/solo-npm:status` — read-only portfolio dashboard with action hints
  - `/solo-npm:audit` — security audit with risk triage (4 tiers); chains to deps for fixes
  - `/solo-npm:deps` — dep upgrade orchestrator with `/verify` gates, batched, rollback on failure, AskUserQuestion gates for major bumps
- **Marketplace.json** drops the `npm-trust` plugin entry. npm-trust
  is a pure CLI as of [`npm-trust@0.9.0`](https://github.com/gagle/npm-trust),
  not a Claude Code plugin. The `/solo-npm:trust` and `/solo-npm:audit`
  skills call it as a tool.

**Migration:**

- Switch invocations: `/npm-trust:setup` → `/solo-npm:trust`. Skill
  content is unchanged.
- If you have committed `.claude/skills/setup/SKILL.md` from the old
  `npm-trust --init-skill setup` flow, delete it; the skill now lives
  only in the marketplace plugin. Invoke as `/solo-npm:trust` directly.
- New consumers: paste the `extraKnownMarketplaces` + `enabledPlugins`
  block into `.claude/settings.json` (see README), accept the install
  prompt, run `/solo-npm:init`.
- Existing consumers with `.claude/skills/release/` or
  `.claude/skills/verify/`: replace with thin wrappers that invoke the
  marketplace baseline. See README's "Composition pattern" section.

## v0.4.0 — Plugin namespacing + drop redundant CI verification

- Renamed plugin: `release-solo-npm` → `solo-npm`. New invocations:
  `/solo-npm:release`, `/solo-npm:init`, `/solo-npm:verify`.
- Skill folders renamed: `release-solo-npm` → `release`,
  `init-solo-npm` → `init`, `verify-solo-npm` → `verify`.
- Removed Phase C.4.5 (ci.yml gating) from the release skill — for
  100% LLM-written code, local `/verify` + tag-triggered `release.yml`
  pre-publish verification is sufficient. CI was redundant.
- README + docs updated to reflect new slash invocations.
- (Brief tag-only release; later superseded by v0.5.0 lifecycle work.)

## v0.3.0 — Init skill

- Added `/solo-npm:init` skill: scaffold release.yml + publishConfig
  + .nvmrc + npm-trust:setup script for fresh repos. Public + private
  registry support.

## v0.2.0 — Marketplace plugin

- Packaged as a Claude Code marketplace plugin
  (`gagle/release-solo-npm`).
- Added `init` and `verify` skills alongside the original `release`
  skill.

## v0.1.0 — Initial release

- The `release` skill (formerly `solo-npm-release-skill`): three-phase
  tag-triggered npm release with OIDC + provenance and one
  `AskUserQuestion` approval gate.
