# Changelog

## v0.17.0 — Cross-tool integration overhaul (npm-trust ^0.11 + prepare-dist ^1.1)

Pairs with `npm-trust@0.11.0` (released today) and `prepare-dist@1.1.0`
(also released today). Together the three releases close a long-standing
integration gap: solo-npm was parsing text output of npm-trust's `--list`
and configure mid-flight, prepare-dist was disconnected from the
publishing orchestration entirely, and the trust cache was time-only
(blind to workflow file edits).

### Pin bumps

- **npm-trust**: `^0.9` → `^0.11`. Skipping 0.10 — every flag the
  /trust skill exercises is shape-stable across 0.10 → 0.11; bump
  goes straight to the latest minor.
- **prepare-dist**: NEW pin `^1.1`. Used by `/init` Phase 0
  (capabilities probe), `/init` Phase 1c (template variant), and
  documented in `/release` Phase E.0 commentary.

### Changed (deterministic, JSON-driven where possible)

- **`/trust`** — Replaced text-parsing of `--list` (3 sites) and
  configure (Step 9) with `--list --json` and `--json` modes. Now
  branches on `ListReport.packages[].trustConfigured` and
  `ConfigureReport.entries[].result` — no more substring matches.
  Updated the version probe from `--auto` (v0.2.0+) to
  `--capabilities` (v0.11.0+).
- **`/release` Phase A.3** — Hot path now uses
  `npm-trust --validate-only --json` (added in 0.10.0) instead of
  `--doctor --json`. ~95% faster on the hot path: zero per-package
  npm calls. Cold path is unchanged but additionally captures
  `workflowSnapshot.fileHash` to enable content-aware cache
  invalidation.
- **`/release` Phase A.3 cache** — content-aware. State.json schema
  v2 adds `trust.workflowFileHash`; the cold path triggers if the
  hash mismatches the live `release.yml` even within the time-TTL.
  Catches the "I edited release.yml but the time-TTL hasn't expired"
  drift class.
- **`/release` Phase C.7.6** — Bundle-size baseline (used by
  `/verify` Tier 4) now reads from `VerifyProvenanceReport.packages[].unpackedSize`
  (added in npm-trust 0.11.0 / schemaVersion 2). Fallback to the
  per-package `npm view dist.unpackedSize` retained for older CLIs.
- **`/status`** — Discovery + per-package data switched to single
  bulk calls: `--doctor --json` for `latestVersion` /
  `lastSuccessfulPublish` / `unpackedSize` / `perPackageIssueCodes`,
  and `--verify-provenance --json` for `provenancePresent` /
  `attestationCount` / `lastAttestationAt`. Eliminates O(N)
  per-package npm view calls.
- **`/audit`** — Trust audit prefix now uses
  `--verify-provenance --json` instead of N individual
  `npm view dist.attestations` calls.
- **`/init` Phase 1c** — Workflow scaffold now consumes
  `npm-trust --emit-workflow [--with-prepare-dist]` instead of the
  inline YAML heredoc. Single source of truth: when GHA action
  versions or the dist-tag rule change, npm-trust emits the new
  template and solo-npm picks it up via the `^0.11` pin.

### Added

- **`/init` Phase 0 — Toolchain capabilities discovery**. Probes
  `npm-trust --capabilities --json` and
  `prepare-dist --capabilities --json` to detect installed tools at
  runtime instead of hardcoding version assumptions. Drives the
  Phase 1c template-variant choice and the H1-H8 error pattern map.
- **State.json schema v2** with `trust.workflowFileHash`.
  Auto-migrates from v1 on first read; first `/release` post-migrate
  takes the cold path to populate the hash.
- **Structured-exit-code → H1-H8 mapping** in `/trust`. Every
  npm-trust invocation now gets a deterministic recovery path based
  on the exit-code numeric (10/20/21/30/40/50/60) instead of stderr
  grep.
- **Phase E.0 (pre-publish prepare-dist transform)** documented in
  `/release`'s Phase 1c template variant. The `--with-prepare-dist`
  variant wires `gagle/prepare-dist@v1` between
  `pnpm run build` and `pnpm publish`. Phase E.0 within the skill
  body is implicit (the GHA workflow does it); the skill detects via
  the Phase 0 capabilities probe whether the project uses
  prepare-dist and selects the variant accordingly.

### Migration notes

- Existing `.solo-npm/state.json` files at `version: 1` are
  auto-upgraded to `version: 2` on first read by adding
  `trust.workflowFileHash: null`. The next `/release` cold-path run
  populates the hash. No user action required.
- Consumers running `/release` against pinned `npm-trust@^0.9` will
  see the version probe report `TOO_OLD` (because v0.9 lacks
  `--capabilities`). Upgrade to `^0.11` via
  `pnpm add -D npm-trust@^0.11`.

## v0.16.0 — Phase −0 help mode + npm-trust pin bump + TOC for large skills + CHANGELOG condensation + manifest CI

Mixed release: one new feature (Phase −0 help mode), one structural pin bump (npm-trust `^0.4` → `^0.9` after explicit verification), and four polish items (TOC for large skill bodies, CHANGELOG condensation, self-audit pass, manifest CI workflow).

### Phase −0 — Help mode (NEW FEATURE — drives the minor bump)

Every skill now responds to help-mode trigger phrases (`--help`, `-h`, `"how does /solo-npm:<skill> work"`, `"how do I use ..."`, `"explain ..."`) by surfacing a 1-screen summary INSTEAD of running the skill. Format: `OPERATIONS / PHASES / EXAMPLES / HARD STOPS`.

Canonical Phase −0 in `/unpublish` shows the example output template; the 12 sibling skills reference the canonical and provide their own skill-specific synthesis content (operations, key phases, trigger examples, hard stops).

Why useful: users who type `--help` reflexively get a tight summary instead of triggering a destructive run. Especially valuable for `/unpublish`, `/deprecate`, `/owner`, `/dist-tag` where the skill might otherwise immediately attempt mutation.

### npm-trust pin bump `^0.4` → `^0.9`

After v0.15.0's explicit deferral of this bump pending maintainer verification, the maintainer (also npm-trust's author) verified that npm-trust 0.9.x retains all 10 CLI flags `/solo-npm:trust` exercises (`--auto`, `--doctor`, `--json`, `--list`, `--only-new`, `--dry-run`, `--repo`, `--workflow`, `--scope`, `--packages`). All present and shape-stable in 0.9.1.

Bumped `/trust` Pre-flight pin from `^0.4` → `^0.9`. Removed the v0.15.0 "tested baseline" caveat comment.

In parallel, **shipped npm-trust 0.9.1** (a patch release) to roll up the 2 commits that had accumulated past the v0.9.0 tag (Built with AI badge in README + thin-wrapper adoption of solo-npm marketplace plugin in dogfood skills) and to mark the verified compatibility checkpoint. See [npm-trust v0.9.1 CHANGELOG](https://github.com/gagle/npm-trust/blob/v0.9.1/CHANGELOG.md).

### Tier A polish

- **#2 — Self-audit pass**: explored all 13 skill bodies for cross-skill drift (Composition sections, internal markdown links, phase references, H1–H8 pattern references, state.json schema sections, skill-name references, agent-skills references). **Zero actual drift found** — the audit's flagged items were either hallucinations (a "Phase E of /init" reference that turned out to be `/release`'s OWN Phase A.3) or correctly-documented optional-enhancement patterns (agent-skills with proper fallback). Skill bodies are clean.

- **#3 — Table of Contents** added to the 3 largest skill bodies (`/unpublish` 1250 lines, `/release` 817, `/init` 761). Each TOC links to top-level sections + adds 1-line context so readers can navigate to the right phase quickly. Particularly useful for `/unpublish` where Phase −1 alone is ~500 lines.

### Tier C — CHANGELOG condensation

CHANGELOG was 1028 lines spanning v0.1.0 → v0.15.0 with full per-version detail. Condensed v0.1.0 → v0.9.0 (the foundation era, 16 versions) into a single milestones table with 1-line headlines. v0.10.0+ kept in full detail (the modern surface).

Result: 1028 → ~360 lines (~65% smaller). Per-version full detail still recoverable via `git show v0.X.Y:CHANGELOG.md` (pointer added at the bottom of the condensed table).

### Tier C — Skill `--help` mode (covered above as the minor-bump driver)

### Tier C — Manifest CI workflow

New `.github/workflows/validate.yml` that runs on push to main:

- JSON parse check on `package.json`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`
- Required-field check (name, description, version, author, license, plugins[].source)
- **Version + description alignment check** between `plugin.json` and `package.json` (catches drift like the v0.10.0–v0.14.0 cases where descriptions got out of sync)
- Skill frontmatter validation: every `.claude/commands/*.md` must have `description:` field

Cheap insurance — runs in <30s, no install needed (Node-only, no deps). Catches the kinds of metadata drift that have actually happened during this project.

### Punch list status

Punch list unchanged at 3 items (Maintainer's regression run + wider adoption + drift caught once). v0.16.0 adds `--help` mode + manifest CI as net-new functionality without affecting v1.0.0 readiness.

### Why minor (not patch)

Phase −0 help mode is a real new feature (new observable behavior — skills surface a different output when invoked with `--help`). Per conventional commits → minor bump.

### Upgrading

`/reload-plugins` after marketplace update. The new `--help` mode is opt-in (only fires on the trigger phrases listed above); existing prompts that don't include those triggers behave identically.

The npm-trust pin bump means consumers' `/solo-npm:trust` invocations will now resolve to `npm-trust@^0.9` instead of `^0.4`. If you've manually pinned npm-trust elsewhere, no impact; the pin only governs the npx-fallback path inside the skill.

## v0.15.0 — 6-item polish consolidation (regression drift, prompts refresh, descriptions, CLAUDE.md, outreach, pin status)

Polish-only release. No behavioral change for consumers; targets the 6 nice-to-have items surveyed after v0.14.0 closed the maintainer-owned docs items on the v1.0.0 punch list.

### What shipped

**Item 1 — `docs/regression.md` minor drift fixes** (S3, S4, S6, S8, S12)
- S3: `/prerelease` Phase E cleanup gate is PROMOTE-only conditional (clarified).
- S4: `/hotfix` E.1 publishConfig.tag uses atomic-write per H7; F.5 forward-port is gated even on cherry-pick success.
- S6: `/dist-tag` and `/deprecate` Phase C.0a now apply H3 auth-race re-check + H5 lock + Phase 0.5/0.5b validation; `/deprecate` uses single-gate plan-approve (dual-gate is `/unpublish`-specific).
- S8: bundle-size cache uses atomic-write + 3-version trim; first-run silent-baseline behavior documented.
- S12: Phase A.3 hot/cold/targeted-path conditional logic explained.

Closes the "minor drift" backlog from the v0.14.0 audit.

**Item 2 — `docs/prompts.md` staleness refresh**
- Added a top-of-doc pointer to README's "Hardening + stability" section so the cross-cutting safeguards (Phase −1, H1–H8, Phase 0.5/0.5b) aren't repeated 13 times per skill.
- Day 1 compound workflow now mentions Phase 1d auto-fix loop + secrets HARD STOP + H6 retry gate, plus Phase 2c H3 auth-race re-check before publish.
- Existing scenarios that were already detailed (Wrong-name fast cleanup, Rename after publish) needed no further changes.

**Item 3 — `unpublish` added to comma-separated description in 3 files**
- `package.json`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` descriptions now list 9 npm commands (added `unpublish`). Was: 8 commands. The description is consumer-visible in the marketplace listing.

**Item 4 — Project-level `CLAUDE.md`**
- New `CLAUDE.md` at the repo root orients Claude when working ON solo-npm (vs working IN consumer repos that USE solo-npm). Mostly references CONTRIBUTING.md "Patterns + conventions" rather than duplicating; gives Claude a direct entry point + reading list when contributing.
- Removed the `(when added)` caveat from CONTRIBUTING.md line 14 since CLAUDE.md now exists.

**Item 5 — `docs/outreach.md` v1.0.0 outreach material**
- New maintainer-only artifact with reusable templates: HN-style "Show HN" post, r/node casual post, ~30-second elevator pitch, awesome-claude-code submission, v1.0.0 announcement post for GitHub Discussions/Releases.
- Clearly labeled "maintainer-only artifact, not consumer-facing" at the top.
- Reduces friction at v1.0.0 declaration time — copy + paste + edit, don't write from scratch under deadline pressure.

**Item 6 — npm-trust pin status comment in `/trust`**
- Pre-flight verification revealed npm-trust shipped 0.5.x–0.9.x since the v0.13.0 D3 pin was set to `^0.4`. publint pin (`@0.3`) is still current.
- Conservative path: kept the `^0.4` pin; added an inline comment in `/trust` Pre-flight section naming "0.4.x as tested baseline; newer majors require explicit verification before bumping". Defers the actual bump to a future v0.15.1 patch where the maintainer (also npm-trust's author) verifies the 0.9.x CLI surface.

### Why minor (not patch)

Per user's explicit direction: "do points 1 to 6 in the same minor version bump". Conventional commits would suggest patch (pure docs + config-metadata changes); minor honors the user's preference for visibility.

### Punch list status (unchanged)

v0.15.0 is pure polish — doesn't move v1.0.0 readiness. Same 3 items remain:
1. Run regression checklist S1–S33 end-to-end at least once (maintainer; requires real `npm publish`)
2. Wider adoption signal (external)
3. Skill-spec drift caught once via regression checklist (temporal)

### Upgrading

`/reload-plugins` after marketplace update. No behavioral change. The new `CLAUDE.md` only applies when working IN this repo; consumers don't see it.

## v0.14.0 — docs alignment pass (CONTRIBUTING + README + regression catch-up)

Closes 3 of the 4 maintainer-owned items on the v1.0.0 punch list. No code changes; pure docs alignment.

### `CONTRIBUTING.md` — `+150` lines

Added a comprehensive **"Patterns + conventions (v0.10.0–v0.13.0)"** section that captures the patterns added in five hardening passes which had no contributor-facing documentation:

- **Phase numbering** (`−1` / `0` / `0.5` / `0.5b` / `A`–`G`) explained — what fires when, what each phase is responsible for.
- **H1–H8 error-handling pattern catalog** with 1-line descriptions per pattern + canonical-in-`/unpublish` reference.
- **npm CLI BCL convention** — every `npm <subcmd>` call site applies the comprehensive BCL table from `/unpublish` Phase −1.4b.
- **State.json conventions** — read-modify-write with unknown-field preservation; atomic writes; corruption guard.
- **Lock file convention** — path, content, acquisition, cleanup.
- **AskUserQuestion conventions** — one question per gate; "Recommended" marker; "Abort" always available; semantic header.
- **Where to document new patterns** — universal patterns canonical in `/unpublish`, brief-reference from siblings.
- **Adding a new error-handling test** convention — sequential S-numbering in regression.md.

Also added step 6 to the "Adding a new command" section: "Apply the patterns + conventions in the section below."

### `README.md` — `+25` lines

Added a new **"Hardening + stability"** top-level section after `npm coverage` that surfaces what was previously invisible to consumers: the user-visible behaviors from v0.10.1–v0.13.0 hardening passes (new STOP classes, tag-collision pre-flight, push categorization, SSL/TLS remediation, rate-limit handling, SIGINT cleanup, conventional-commits did-you-mean, concurrent-invocation locks). Lets consumers wondering "is it stable enough" find the answer without digging through 4 versions of CHANGELOG.

### `docs/regression.md` — major drift fixes on S1, S2, S5, S7

Per the audit, these scenarios described pre-v0.10.0 behavior. Fixed:

- **S1** (Daily release happy path): now describes the full Phase −1 / Phase 0 / 0.5 / 0.5b / A.1–A.5 / B.3 / C.0a / C.1–C.7.6 sequence including the load-bearing additions (audit cache pre-flight, tag-collision pre-flight, push rejection categorization, H4 propagation retry, B5 warn-don't-fail on `gh release create`, atomic state.json writes).
- **S2** (Pre-release start): added Phase 0.5 IDENTIFIER validation with did-you-mean + Phase A `release.yml` freshness check with optional `/init --refresh-yml` auto-chain.
- **S5** (CVE response with deprecate): added summary-first parse for >500-dep repos, H6 chain-failure recovery on `/deps` failure, and the cross-skill effect on `/release` Phase A.5 audit cache.
- **S7** (First publish on brand-new repo): now documents the load-bearing Phase 1d pkg-check validation loop (auto-fix loop, secrets HARD STOP, H6 retry gate), Phase 1c .npmrc API, Phase 2c H3 auth-race re-check, and trust chain H6 recovery.

Minor drift items in S3, S4, S6, S8, S12 are noted in the audit but not surgically fixed in this pass — they're noise-level (phase mechanics not described, optional gates not mentioned). May fix opportunistically in a future docs pass.

### v1.0.0 punch list status

Before v0.14.0:
- 6 items: 4 maintainer-owned + 2 external/temporal

After v0.14.0:
- **3 items remaining**: 1 maintainer-owned (run regression S1–S33 end-to-end at least once) + 2 external/temporal (wider adoption signal, drift caught once)

Of the 3 maintainer-owned docs items shipped here, only **the live end-to-end regression run** remains — that requires actual `npm publish` against real npm, which only the maintainer can do.

### Why minor (not patch)

CONTRIBUTING.md additions are substantive (new conventions section); README adds a new top-level section; regression.md scenario rewrites change observable behavior expectations. Per the user's direction, ships as v0.14.0 minor bump.

### Upgrading

`/reload-plugins` after marketplace update. No behavioral change for consumers — pure docs alignment.

## v0.13.0 — Tier-4 polish pass (proactive rate tracker + npm BCL table + shell-safety + CRLF-in-tarball + submodules + audit-summary-first)

Closes the entire `Post-v1.0.0 polish (finalized non-goals)` list from `docs/stability.md`. After v0.13.0 ships, the polish list is empty — every previously-deferred item shipped at scoped depth. The only thing blocking v1.0.0 declaration is the 3 external/temporal items (demo repo, wider adoption, regression-caught-once).

### Sections shipped

- **Section A — H7 extensions**: multi-installation `which -a` reporting in `/unpublish` Phase −1.1 (surface conflicting tool installations); submodule state detection in new Phase −1.6b (HARD STOP if submodules unsynced/conflicted, WARN if uninitialized).
- **Section B — Proactive rate-limit tracker (Tier-4 #1)**: new `gh_with_rate_tracker` helper in `/unpublish` Phase −1.9b that reads X-RateLimit-Remaining via `gh api rate_limit` periodically and proactively sleeps before hitting 429. Falls through to H8 reactive backoff as the safety net. Applied to `/status` Phase 2 gh fan-out.
- **Section C — Comprehensive npm BCL table (Tier-4 #2)**: documents every npm subcommand solo-npm invokes + cross-version variations + defensive-parse convention. Centralized in `/unpublish` Phase −1.4b extension; sibling skills apply the convention at each call site they make.
- **Section D — `gh` GraphQL fan-in for `/deps` (Tier-4 #3)**: extends the v0.12.0 `/status` GraphQL pattern to `/deps` Phase 3 release-notes lookup for major-bump batches. Threshold: >5 unique upstream repos. Saves 5–15s on large dep batches.
- **Section E — Shell-safety hardening Phase 0.5b (Tier-4 #4)**: rejects shell metacharacters (`;`, `&`, `|`, backtick, `$`, parens, redirects, control chars) in extracted name/scope/tag/version slots after Phase 0.5 regex passes. Applied universally; canonical in `/unpublish`. Special handling for `MESSAGE` slots (relaxed; passed via env-var to avoid shell-interpolation context).
- **Section F — `/audit` summary-first parse (Tier-4 #5)**: Phase 1 parses severity counts first, then drills into Tier 1/2 advisory detail only; defers Tier 3/4 detail to opt-in `--full` flag. Avoids token-budget truncation on >500-dep repos that would silently miss advisories.
- **Section G — `/verify` CRLF-in-tarball detection (Tier-4 #8)**: extends v0.12.0 J's `core.autocrlf` warning with detection of **actual CRLF in published bin scripts / shebang files / `.sh` files**. HARD STOP if found — these break Unix consumers (`#!/usr/bin/env node\r` fails to parse). Better implementation than elevating the v0.12.0 J WARN to STOP.

### Tier-4 items shipped (and their stability.md correspondence)

| Tier-4 # | Section | Where |
|---|---|---|
| 1. Concurrent rate-limit tracker | B | unpublish Phase −1.9b + status |
| 2. Full npm BCL | C | unpublish Phase −1.4b extension |
| 3. gh GraphQL beyond /status | D | deps Phase 3 |
| 4. AI-prompt injection (shell-safety) | E | unpublish Phase 0.5b + 5 siblings |
| 5. Audit output truncation | F | audit Phase 1 |
| 6. Submodule state detection | A | unpublish Phase −1.6b |
| 7. Tool version shadowing (multi-install info) | A | unpublish Phase −1.1 |
| 8. CRLF as STOP (re-framed: actual CRLF in bin scripts) | G | verify Tier 3 |

### Why minor (not patch)

New shell-safety check is observably stricter (rejects values with `$` etc.) — some users may see new STOPs. New CRLF-in-tarball check is an entirely new HARD STOP class. Following conventional commits → minor bump.

### v1.0.0 status

After v0.13.0, the entire pre-v1.0.0 code-level work is complete. The `Post-v1.0.0 polish (finalized non-goals)` list in `docs/stability.md` is being **removed entirely** because every item shipped. Only 3 external/temporal items remain blocking v1.0.0:
1. Demo / showcase repo (`solo-npm-example`)
2. Wider adoption signal (external user beyond rfc-bcp47/ncbijs/npm-trust)
3. Skill-spec drift caught once via the regression checklist

Code-side, solo-npm is v1.0.0-ready.

### Upgrading

`/reload-plugins` after marketplace update. No state.json migration. The new shell-safety check rejects more inputs at extraction; if you've been passing values that contain shell metacharacters (rare), the skill will surface a STOP with a diagnostic — fix the input value, no skill-side change required.

## v0.12.0 — Tier-3 stability comprehensive pass (H8 rate-limit + SSL + SIGINT + npm BCL + workspace edges + gh GraphQL)

Closes all 9 Tier-3 stability items previously deferred per `docs/stability.md`, plus 3 adjacent items (J/K/L) that fit the same category. With v0.12.0 shipped, **the code-level v1.0.0 entry criteria are complete**. What remains is external/temporal: demo repo, wider adoption signal, regression catch-once.

### Tier-3 items shipped

- **A — H8 rate-limit backoff helper (NEW)** in `/unpublish` Phase −1.9. Detects 429 / "rate-limited" stderr; exponential 1s/2s/4s/8s with ±20% jitter; max 4 attempts. Applied to portfolio-fan-out paths in `/status` Phase 2, `/deprecate` Phase A.3, `/dist-tag` Phase A.3, `/owner` Phase A.3. Surfaces "rate-limited" per-package on exhaustion rather than collapsing into ambiguous "—".
- **B — SSL/TLS error remediation (NEW)** in `/unpublish` Phase −1.10. Pattern matching on `SSL_ERROR_*`/`cert*`/`unable to get local issuer`/`TLSV1_ALERT` in stderr from curl + git push + npm. Surfaces four-option remediation: transient retry / corporate-proxy CA bundle / OS CA bundle update / `--insecure` last resort. Applied at git push boundaries in `/release`/`/prerelease`/`/hotfix`, curl boundaries in `/status`/`/unpublish`. We hit this on the v0.10.1 push — real failure mode now has clear remediation.
- **C — SIGINT signal handling (NEW)** in `/unpublish` Phase −1.11. State-aware cleanup on Ctrl+C: deletes local-only tag if `LOCAL_TAG_PENDING_PUSH` was set, drops stash if `STASH_PENDING`, reverts `publishConfig.tag` mutation if `PUBLISH_CONFIG_PENDING`. Sibling skills set their own state flags at corresponding mutation points. Scoped (high-value cleanups only), not a full state machine.
- **D — Environment-variable validation** in `/unpublish` Phase −1.3b. `$HOME` writability check (npm requires it for `~/.npmrc` and `~/.npm/_cacache`); `$TMPDIR` fallback to `/tmp`. Surfaces in extreme CI environments where these are unset.
- **E — `.npmrc` ad-hoc parsing → `npm config get` API** in `/init` Phase 1c. Replaces grep-based `.npmrc` reads with `npm config get registry` / `npm config get @<scope>:registry`. npm handles precedence (project < user < global) natively. The "undefined" literal vs empty string ambiguity is normalized via the BCL guard from Phase −1.4b.
- **F — Conventional-commits strictness** in `/release` Phase B.3. Type whitelist (feat/fix/chore/docs/style/refactor/perf/test/build/ci/revert) with did-you-mean hint via Levenshtein-closest match. Special case: `breaking:` (non-canonical typo) is detected and the user is told to use `feat!:` or `BREAKING CHANGE:` footer. Subject length > 72 → warning (not block).
- **G — npm CLI BCL extensions** in `/unpublish` Phase −1.4b. Cross-version cases beyond v0.11.0's coverage: `dist-tags = {}` empty-object vs missing-field; `npm whoami` username with/without `@` prefix; `npm audit` schema differs npm 6 (uses `actions[]`) vs npm 7+ (uses `vulnerabilities{}`); `npm config get` `undefined` literal normalization.
- **H — Workspace symlink edge cases** in `/init` Phase 1a. STOPs if `pnpm-workspace.yaml` glob expands to zero matching dirs (typo or empty workspace) OR matching dirs lack `package.json`. Surfaces "publishing only N of M packages" if mixed `private: true` + public packages exist in the same workspace.
- **I — `gh` GraphQL impl for >20-package portfolios** in `/status` Phase 2. Provides the actual GraphQL query template (one batch call vs N×2 individual calls). Falls back to per-repo individual calls if graphql fails (insufficient token scope, malformed query, rate-limited at global level).
- **J — CRLF / `core.autocrlf` detection (NEW)** in `/unpublish` Phase −1.5b. Detects `git config core.autocrlf=true` and surfaces `core.autocrlf=input` recommendation for npm-shipped repos (CRLF in tarball can break Unix consumers running bin scripts). Non-fatal; warning only.
- **K — `git log` truncation cap (NEW)** in `/release` Phase B.3. Caps at `--max-count=500`; if hit, surfaces "X commits since LATEST_TAG (showing first 500); review CHANGELOG carefully before approving" warning. Keeps conventional-commit parsing bounded.
- **L — Pre-push hook failure rollback (NEW)** in `/release` C.3 (canonical), `/prerelease` C.3, `/hotfix` E.4, `/deps` Phase 5. Distinguishes server-side `pre-receive` rejection from client-side `pre-push` rejection; surfaces case-specific recovery options including `--no-verify` bypass and `rm .git/hooks/pre-push` for the client-side case.

### Why minor (not patch)

The new H8/SIGINT/SSL/CRLF/log-cap behaviors change observable runtime characteristics (e.g., `/status` now retries on 429 instead of failing fast; `/release` now caps `git log` and warns on truncation). Following conventional commits → minor bump.

### Upgrading

`/reload-plugins` after marketplace update. No state.json migration needed. The new SIGINT trap pattern requires sibling skills to set state flags at mutation points; for skills not yet updated, the trap simply has nothing to clean (graceful no-op). The H8 backoff helper is referenced from siblings; if a skill doesn't yet wrap its `npm view` calls, the existing behavior is preserved.

### v1.0.0 status

After v0.12.0, all code-level pre-v1.0.0 entry criteria from `docs/stability.md` are complete. Three external/temporal items remain:
- Demo / showcase repo (`solo-npm-example`)
- Wider adoption signal (external user beyond rfc-bcp47/ncbijs/npm-trust)
- Skill-spec drift caught once via the regression checklist before being noticed in production

These need real-world usage to validate API stability, not code changes. v1.0.0 will ship when those three are met.

## v0.11.0 — Tier-2 strict-safety hardening pass (universal Phase A + concrete C/H + prompt validation)

Major hardening release — closes ~16 Tier-2 strict-safety gaps surfaced by the post-v0.10.1 audit. No new features, no behavioral changes for the happy path. Substantially better diagnostics, schema resilience, concurrent-safety, and prompt-extraction validation.

### Section A — Universal Phase A pre-flight extensions (extend H7)

- **A1 detached-HEAD detection** in every skill that creates commits/tags. `git status --porcelain` passes on detached HEAD; the previous code would create dangling commits/tags. Detection: `git symbolic-ref -q HEAD`. Canonical in `/unpublish` Phase −1.5.
- **A2 worktree detection** in every skill that mutates git or scaffolding state. `.solo-npm/` artifacts are tied to the main worktree; running from a `git worktree` produces confusing cache-state mismatches. Canonical in `/unpublish` Phase −1.6.
- **A3 `gh` Phase A pre-flight** hoisted from per-call detection. Every skill using `gh` (release, prerelease, hotfix, status, unpublish) now reports up-front: `gh: ready (vX.Y.Z)` / `gh: installed but not authenticated` / `gh: not installed`. Skills gracefully skip gh-dependent steps with a clear "what was skipped and why" note instead of failing mid-flow.

### Section B — Skill-specific surgical fixes

- **B1 `gh run watch` 30min timeout cap** in `/release` C.6, `/prerelease` C.6 (already in `/hotfix` E.4 from v0.10.1). Surfaces clear "CI may still be running" message on timeout instead of an indefinite hang.
- **B2 `git stash pop` conflict recovery** in `/deps` Phase 4d (rollback path). When the stash pop fails with merge conflicts, surface the conflicted files + recovery options (resolve-then-drop / discard / re-apply later). Previously left the stash silently on the stack.
- **B3 pre-commit hook failure rollback** in `/release` C.2, `/prerelease` C.2, `/hotfix` E.4, `/deps` Phase 4e. Detects hook rejection vs other commit failures; surfaces the hook output + recovery options including `--no-verify` bypass and `git reset HEAD` to roll back staging.
- **B4 `git fetch` shallow detection** in `/hotfix` Phase B. CI clones with `--depth=N` were silently producing missing-source-tag failures; now detects via `git rev-parse --is-shallow-repository` and surfaces `git fetch --unshallow origin` as the fix.
- **B5 GitHub Release create failure → warn, not block** in `/release` C.7.5. The npm publish already succeeded; the release page is a secondary artifact. Demoted to warning + manual remediation snippet (`gh release create v$V --notes-from-tag`). User can re-run later without re-publishing.
- **B6 `npm pack --dry-run` JSON schema validation** in `/verify` Tier 3. Defensive parsing of `unpackedSize`, `entryCount`, `files[]` — falls back to file-size-sum if `unpackedSize` is missing (npm 7 omits it). Hard warning if `files[]` itself is empty.

### Section C — Actual H3/H4/H5/H6 implementations (the v0.10.0 reference text was skeletal)

- **C1 H3 auth-race re-check (concrete impl)** in `/deprecate` Phase C.0a, `/dist-tag` Phase C.0a, `/owner` Phase C.0a. Re-runs `npm whoami` immediately before the first destructive call; STOPs if session expired or user changed during the AskUserQuestion gate at Phase B. Matches the canonical `/unpublish` Phase C.0 implementation.
- **C2 H4 retry implementation** in `/audit` Phase 5 (`npm view <pkg> dist-tags.next` for stale-@next note). 3-attempt × 5s retry against propagation lag.
- **C3 H5 concurrent-release lock** in `/release` Phase C.0a, `/hotfix` Phase D.0a, `/prerelease` Phase C.0a. Per-repo lock at `.solo-npm/locks/<repo-root-hash>.lock` with stale-PID auto-cleanup. Two parallel releases on the same repo would otherwise race the git tag and double-publish.
- **C4 H6 chain-failure handling (concrete impl)** in `/audit` Phase 5 (chains to `/deps`, `/deprecate`); `/status` `--fresh` flag; `/init` Phase 1d (chain to `/verify --pkg-check-only`); `/init` Phase 3 (chain to `/trust`). Captures child-skill STOP messages verbatim and surfaces them in the parent context with retry/abort/fall-back-to-cache options instead of silently swallowing.

### Section D — Schema resilience (extend H2)

- **D1 state.json schema preservation on writes**. Every write reads + mutates only the relevant keys + writes back the entire object. Forward-compatible: a v0.11.0 skill running against a v0.12.0-shaped cache won't truncate fields it doesn't understand. Canonical in `/unpublish` Phase −1.4.
- **D2 `npm view` defensive schema parsing** for cross-npm-version compatibility. `dist.attestations` may be missing on private registries; `time.<v>` format varies (ISO-8601 vs Unix epoch ms). Documented fallbacks per field. Canonical in `/unpublish` Phase −1.4b.
- **D3 `npm-trust` npx pin** in `/trust` Pre-flight. Replaces `npm-trust@latest` (could fetch any future version) with `npm-trust@^0.4` (pinned major). Bumped explicitly when solo-npm is tested against a newer npm-trust major.
- **D4 `npm pack --dry-run` defensive parsing** in `/verify` Tier 3, `/release` C.7.6 bundle-size cache. Fallback: if `unpackedSize` is missing, sum `files[].size` manually.

### Section E — AI-prompt regex validation framework

- **E1 semver regex validation** in Phase 0.5 of `/unpublish` (canonical), `/release`, `/prerelease`, `/hotfix`, `/dist-tag`, `/deprecate`. Rejects malformed extracted versions BEFORE Phase B's plan rendering. Canonical regex framework documented once in `/unpublish` Phase 0.5; siblings reference it and apply only the slots they extract.
- **E2 identifier whitelist validation** in `/prerelease` Phase 0.5. Rejects typos like "alpa", "betta" with a "did you mean" hint. Whitelist: alpha, beta, rc, canary, experimental, next, preview, edge.
- **E3 scope/name regex validation** across all Phase 0 extractions. npm naming rules enforced at the boundary: `^(@[a-z0-9][a-z0-9._-]*/)?[a-z0-9][a-z0-9._-]*$`. Rejects shell-metacharacter-laden names that could cause confused execution.

### Why minor (not patch)

The Section A and D extensions add new universal pre-flight checks that some users will see as new STOPs (detached HEAD, worktree, missing gitconfig), even though those STOPs were always-correct safety refusals just not previously surfaced. Following conventional commits → minor bump.

### Upgrading

`/reload-plugins` after marketplace update. No state.json migration needed; D1 schema preservation makes the cache forward-compatible automatically. If you've been running solo-npm in a `git worktree` (uncommon), v0.11.0 will surface a STOP — switch to the main working directory or wait for v0.11.1 if worktree support becomes a real ask.

## v0.10.1 — strict-safety hardening pass (H7 + tag/push categorization + curl timeouts)

Patch release that closes the 9 Tier-1 strict-safety gaps surfaced by a follow-up cross-skill audit on every external-tool invocation. No new features; no behavioral changes for the happy path; substantially better diagnostics + remediation when something goes wrong.

### H7 — universal external-tool reliability pattern (NEW)

New canonical pattern in `/solo-npm:unpublish` Phase −1 (referenced from sibling skills). Every external tool the skill invokes is verified upfront with three checks: **presence**, **version detection** (best-effort), and **timeout-wrappable invocation**. Plus two extensions:

- **`.gitconfig` user identity check** — surfaces missing `user.name` / `user.email` upfront with a concrete fix (`git config --global ...`) instead of letting the user hit a cryptic exit-128 from `git commit` mid-skill.
- **Atomic `state.json` writes** — write to `.solo-npm/state.json.tmp`, then `fs.renameSync()`. Single atomic inode swap on POSIX. A killed-mid-write process leaves the original cache intact (the leftover `.tmp` is harmless because the H2 corruption guard already wraps reads in try/catch).
- **Stale-lock auto-cleanup** — the H5 lock check now distinguishes "live PID holding the lock" (STOP — wait or kill) from "dead PID holding a stale lock" (WARN + clean + proceed). Fixes the case where a SIGKILL-ed prior run left the lockfile stuck and blocking forever.

### Per-skill strict-safety fixes

- **`git tag` collision pre-flight** in `/release` C.5, `/prerelease` C.5, `/hotfix` E.4 — `git ls-remote --tags` BEFORE local tag creation. The wrong-name re-release case (`/unpublish` then `/release` v1.0.0 again) was the load-bearing failure mode this fixes; previously the local tag would create successfully but `git push --tags` would fail mid-flow with the local tag dangling.
- **`git push` rejection categorization** in same three skills — non-fast-forward / pre-receive hook / branch protection / auth failure all collapsed into one verbatim stderr dump previously. Now each is detected and surfaces a specific remediation block.
- **`--max-time 10 --connect-timeout 5` on every `curl` call** to deps.dev and the npmjs downloads API — `/unpublish` A.3, `/status` Phase 2. Previously a hung network connection would block the skill indefinitely.
- **`timeout 30 npm view ... --json 2>/dev/null`** on every JSON-parsing npm call — `/dist-tag` A.3, `/status` Phase 2, `/deprecate` A.3. The `2>/dev/null` redirect prevents npm warnings (deprecation notices, registry hints) from corrupting the JSON parse downstream; the `timeout` bounds the call against a slow/stuck registry.
- **Tag push failure rolls back the local tag** in `/release` C.5, `/hotfix` E.4 — if `git push --tags` fails for any reason after the local tag was created, the local tag is now `git tag -d`'d so the next `/release` attempt can re-tag without conflicting.

### Why patch (not minor)

These are pure-additive safety hardenings, no behavioral changes for the happy path. Fix-mode commits per conventional-commits convention → patch bump.

### Upgrading

`/reload-plugins` after marketplace update. No state.json migration needed; the H7 atomic-write pattern is forward-compatible (existing caches read normally; future writes are atomic).

## v0.10.0 — `/solo-npm:unpublish` + cross-skill systemic hardening + auto-cleanup

Big stability-grade release. Adds skill #13 (`/solo-npm:unpublish`) for the rare-but-real wrong-name and rename-after-publish cases, with strict safety gates. Also lands a cross-skill systemic hardening pass (38 patches across 11 sibling skills) for six error-handling patterns surfaced by a comprehensive ~257-error-path audit, plus an auto-cleanup gate for git tags + GitHub Releases after unpublish.

### `/solo-npm:unpublish` — skill #13

Three operations, two-gate destructive confirmation, **defaults to recommending deprecate** in every gate (npm's preferred path):

- `unpublish-version` — single version (e.g., wrong-name typo cleanup within 72h)
- `unpublish-all` — full-package removal (`--force`)
- `rename-redirect` — deprecate old name with redirect message + optionally unpublish eligible old versions; user publishes new name separately via `/init` or `/release`

Eligibility check via [deps.dev](https://docs.deps.dev/api/v3/) `/dependents` API. Differentiates **deps.dev 404 (not indexed yet)** from API-down — a freshly-published wrong-name package isn't blocked just because deps.dev hasn't crawled it yet.

Hard stops with **no override flag**: any version with dependents (would break consumers); post-72h with criteria not met (downloads >300/wk OR multiple owners). The escape hatch is manual `npm unpublish` outside the skill — the skill refuses to participate in unsafe operations, doesn't prevent the user.

After successful unpublish, an **auto-cleanup gate** offers to delete the local + remote git tag and the GitHub Release in one step (or just the git tag, or keep both for audit trail). Git/GitHub artifacts are user-controlled — surfaced as a gate, not an automatic side-effect.

### Cross-skill systemic hardening (H1–H6)

A comprehensive cross-skill error-path audit surveyed about 257 error paths across all 12 skills and surfaced six systemic gaps. v0.10.0 lands the canonical reference patterns in `/solo-npm:unpublish` and references them from the 11 sibling skills:

| Pattern | What | Affected skills |
|---|---|---|
| **H1** OTP / 2FA-on-writes | Detect `EOTP` / `OTP required` in `npm <cmd>` stderr; surface manual `--otp=<code>` handoff so the skill never hangs awaiting stdin OTP | `/deprecate`, `/dist-tag`, `/owner`, `/init` Phase 2c, `/unpublish` |
| **H2** state.json corruption guard | try/catch around every `.solo-npm/state.json` read; non-fatal warning + treat as empty cache on parse fail | `/release`, `/prerelease`, `/hotfix`, `/init`, `/trust`, `/audit`, `/deps`, `/status`, `/verify`, all 4 destructive-CLI skills |
| **H3** Auth-window race | Re-check `npm whoami` immediately before destructive op after any AskUserQuestion gate, in case the npm session expired during the wait | `/init` Phase 2c (canonical), `/release` + `/prerelease` (transitive via Phase G chain to `/deprecate`) |
| **H4** Registry propagation lag retry | 3 attempts × 5s sleep on post-mutation `npm view`; non-fatal note if still inconsistent (npm CDN can take up to 5min) | `/release` C.7, `/prerelease` C.7, `/hotfix` E.5, `/deprecate` D, `/dist-tag` D, `/owner` D, `/deps`, `/unpublish` D.1 |
| **H5** Concurrent invocation lock | File lock at `.solo-npm/locks/<pkg>.lock`; refuse to start if held; PID file with trap cleanup | `/deprecate`, `/dist-tag`, `/owner`, `/unpublish` |
| **H6** Chain-target failure recovery | Capture child STOP messages verbatim and surface in parent context with retry/abort options; don't silently swallow chain failures | All 10 chain edges in the skill graph |

The 11 sibling skills each got an "Error-handling patterns" subsection referencing `/unpublish`'s canonical wording, with skill-specific adaptation per pattern. 38 patches total.

### Other changes

- `/verify` Tier 3 secrets-detection HARD STOP block extended with a **post-publish remediation note**: if the secrets already shipped, *rotate first* (load-bearing fix), then `/solo-npm:unpublish` within the 72h window if available. Outside the window, fall back to `/deprecate` and rely on rotation.
- `docs/regression.md` adds S13–S17 covering the `/unpublish` happy path, blocked-dependents HARD STOP, auth-window race detection, concurrent-lock refusal, and deps.dev 404 (not-indexed) treated as 0 dependents.
- `docs/npm-coverage.md` moves `npm unpublish` from "non-goal" → "✓ shipped (v0.10.0)"; corrects stale "24h window" mentions to "72h" (npm policy changed years ago).
- `docs/stability.md` updates the pre-v1.0.0 punch list — most v1.0.0 entry criteria now closed by v0.10.0.

### Upgrading

`/reload-plugins` after marketplace update. No state.json migration needed; the H2 corruption guard makes existing caches forward-compatible (any malformed entry is now non-fatal).

## v0.1.0–v0.9.0 — Foundation era (condensed)

Pre-v0.10.0 history. Each release's full detail is preserved in git tags (`git show v0.X.Y:CHANGELOG.md`); this table captures the milestones at a glance:

| Version | Headline |
|---|---|
| **v0.9.0** (2026-05-04) | stability roadmap + deps.dev integration + gh detection + audit clarification — `docs/stability.md` and `docs/regression.md` shipped; security-signal section in `/status` |
| **v0.8.0** | polish iteration toward v1.0.0 — de-hardcoded skill counts in user-facing copy; doc-drift fixes; `/release` PACKAGE_NOT_PUBLISHED auto-chain to `/init`; phase-naming convention codified; cache `lastSize` retention policy |
| **v0.7.0** | guided initial-publish flow in `/init` Phase 2 (replacing v0.6.x STOP-and-resume); GitHub Releases creation in `/release` C.7.5; bundle-size regression detection (`/verify` Tier 4) |
| **v0.6.1** | pkg-check enhancement — publint Tier 1, manual Tier 2 checks, `npm pack --dry-run` Tier 3 (tarball-content audit + secrets HARD STOP), `.gitignore`-vs-`files` divergence |
| **v0.6.0** | npm operator coverage — three new skills (`/dist-tag`, `/deprecate`, `/owner`); `/verify` pkg-check Step 5; framing alignment (skills as operators) |
| **v0.5.5** | auto-chain consistency, multi-channel awareness, cherry-pick automation, README rework |
| **v0.5.4** | universal `/release` entry; `/prerelease` and `/hotfix` as 1st-class skills; `agent-skills` composition pattern |
| **v0.5.3** | auto-chain on trust gap; foolproof manual handoffs (`npm login` web 2FA); trust-state cache |
| **v0.5.2** | strip `name:` frontmatter from commands to match `addyosmani` convention |
| **v0.5.1** | move `skills/` → `commands/` for namespaced autocomplete |
| **v0.5.0** | complete lifecycle plugin (7 skills) |
| **v0.4.0** | plugin namespacing + drop redundant CI verification |
| **v0.3.0** | `/solo-npm:init` skill — scaffold `release.yml` + `publishConfig` + `.nvmrc` + `npm-trust:setup` script for fresh repos |
| **v0.2.0** | packaged as Claude Code marketplace plugin (`gagle/release-solo-npm`); `init` and `verify` skills added |
| **v0.1.0** | the `release` skill (formerly `solo-npm-release-skill`): three-phase tag-triggered npm release with OIDC + provenance and one `AskUserQuestion` approval gate |

For full detail of any specific release: `git show v0.X.Y:CHANGELOG.md`.
