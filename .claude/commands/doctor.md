---
description: Pure read-only diagnose-and-report skill for the publish/release health of the repo. Triggers from prompts like "is my repo healthy?", "diagnose the repo", "what's wrong with this setup?", "publish health check", "show me the publish-side issues". Emits a unified DoctorReport across 5 publish-domains (trust, provenance, config-publish, security-gate, publish-readiness) with each issue mapped to a remediation skill. Use for cold checks, pre-release dry-runs, or to see what /doctor --fix would do. Use --fix to apply auto-remediation.
---

# Doctor

The umbrella health-check skill for solo-npm. Pure read-only by default; `--fix` applies auto-remediation by chaining into other skills.

Doctor's domain is **publish-side health only** — solo-npm is the AI-driven release wizard for TypeScript npm packages, and Doctor reports only on what affects releasing safely. It does NOT diagnose build tooling, lint config, daily-dev hygiene, README quality, or other project-meta concerns.

## Phase −0 — Help mode (per `/unpublish` canonical)

If the user's prompt contains `--help` / `-h` / `"how does /solo-npm:doctor work"` / similar, surface a help summary **INSTEAD** of running.

Synthesize from the **Operations** (read / fix), **5 domains** (trust / provenance / config-publish / security-gate / publish-readiness), and 2-3 trigger phrases. Note: read-only by default; `--fix` mutates files via chained skills. See `/unpublish` Phase −0 for canonical format.

After surfacing, **STOP**.

## When to use

- Pre-release dry-run: see what /release Phase A would gate on
- After modifying release.yml: confirm structure and trust binding still align
- After upgrading toolchain: re-probe capabilities and detect drift
- Cold check on an unfamiliar repo: get a one-shot health snapshot
- CI gate: `/solo-npm:doctor --json` exit codes + summary integrate into pipelines

## Operations

| Operation | Trigger | Effect |
|---|---|---|
| `read` (default) | `/solo-npm:doctor` | Run all 5 domain probes; render report; exit code 0 (ok), 1 (warn), or per-domain failure code |
| `fix` | `/solo-npm:doctor --fix` | Run probes, then chain into remediation skills for `autoFixable` issues |
| `json` | `/solo-npm:doctor --json` | Same as read; emit `DoctorReport` JSON to stdout |
| `fresh` | `/solo-npm:doctor --fresh` | Skip cache; force-refresh every probe |

## Phase 0 — Read prompt context

Extract from the user prompt (per /unpublish Phase 0 / E2 from v0.11.0):

| Slot | Source | Default |
|---|---|---|
| `OPERATION` | trigger phrase / flags | `read` |
| `JSON_OUT` | `--json` flag | `false` |
| `FRESH` | `--fresh` flag or "force refresh" / "without cache" in prompt | `false` |
| `FIX_MODE` | `--fix` flag | `false` |
| `STRICT` | `--strict` flag | `false` (treats warn as fail; used by CI gates) |

## Phase 1 — Toolchain probe

Read `.solo-npm/state.json#toolchain` (per `/solo-npm:toolchain` Phase 1). If `cacheStale` or `FRESH`, refresh via the toolchain skill's Phase 2 probe.

The Doctor's domain probes consume the cached descriptors:

- `npm-trust` capability descriptor → required features: `doctor`, `validate-only`, `verify-provenance` (for the trust + provenance domains)
- `prepare-dist` capability descriptor → required if `package.json#devDependencies.prepare-dist` is present (for prepare-dist usage detection)

If `npm-trust` is missing or the required features are absent, **STOP** with the precise diagnostic (`--capabilities` output mismatch). Doctor cannot probe trust/provenance domains without npm-trust.

## Phase 2 — Domain probes (parallel)

Run the 5 domain probes in parallel. Each emits a `DomainReport`.

### 2.1 — `trust` domain

Source: `npm-trust --doctor --json --workflow <RELEASE_WORKFLOW>` (use cached output if `/release` ran in last hour).

Map npm-trust's `issues[]` directly into the trust domain's `DoctorIssue[]`. Each issue retains `code`, `message`, `remedy` from npm-trust; add `domain: 'trust'`, `remediationSkill` per the table below.

| npm-trust issue code | severity | remediationSkill |
|---|---|---|
| `NODE_TOO_OLD` | fail | manual (install Node ≥24) |
| `NPM_TOO_OLD` | warn | manual |
| `NPM_UNREACHABLE` | warn | manual (check PATH) |
| `AUTH_NOT_LOGGED_IN` | warn | manual (`npm login`) — H1 web 2FA |
| `AUTH_REGISTRY_UNUSUAL` | warn | manual |
| `WORKSPACE_NOT_DETECTED` | warn | manual or `/init` to scaffold a single-package layout |
| `WORKSPACE_EMPTY` | warn | manual (mark a package public, or use `--packages`) |
| `REPO_NO_REMOTE` | warn | manual (`git remote add origin <url>`) |
| `REPO_REMOTE_NOT_GITHUB` | warn | manual (OIDC requires GitHub today) |
| `WORKFLOWS_NONE` | warn | `/init` (scaffold release.yml via `npm-trust --emit-workflow`) — autoFixable |
| `WORKFLOWS_AMBIGUOUS` | warn | manual (pass `--workflow <file>`) |
| `WORKFLOW_NOT_FOUND` | warn | `/init --refresh-yml` or fix typo manually |
| `PACKAGE_TRUST_DISCREPANCY` | warn | informational (web-UI configured) |
| `PACKAGE_NOT_PUBLISHED` | warn | `/release` for first publish |
| `REGISTRY_UNREACHABLE` | warn | manual / retry |
| `REGISTRY_PROVENANCE_CONFLICT` | warn | manual (drop `provenance: true` OR change registry) |
| `WORKFLOW_AUTH_MISMATCH` | warn | `/init --refresh-yml` |
| `NPMRC_REGISTRY_DIVERGES` | warn | manual (align .npmrc and publishConfig) |
| `NPMRC_LITERAL_TOKEN` | fail | manual (replace with `${ENV_VAR}` reference) |
| `WORKFLOW_MISSING_AUTH_SECRET` | warn | manual (set GitHub Actions secret) |
| `DOCTOR_FLAG_IGNORED` | warn | manual |

### 2.2 — `provenance` domain

Source: `npm-trust --verify-provenance --auto --json` (cached if recent).

For each package in the report's `packages[]`:

- `provenancePresent: false` AND `published: true` → emit `PROVENANCE_MISSING` (warn) issue with remedy "publish via the OIDC release.yml; verify trust is configured for this package via /trust"
- `attestationCount === 0` AND `provenancePresent: true` → emit `ATTESTATION_COUNT_ZERO` (warn — should be at least 1 if provenance present)
- `published: false` → not an issue here (covered by trust domain's `PACKAGE_NOT_PUBLISHED`)

Surface the `summary` block (total / withProvenance / withoutProvenance / unpublished) for the `ok/warn/fail` counts.

### 2.3 — `config-publish` domain

Read-only checks against the repo's filesystem. No external calls.

| Check | Issue code | Severity | Auto-fixable | Remediation |
|---|---|---|---|---|
| `package.json#publishConfig` missing | `PUBLISHCONFIG_MISSING` | warn | yes | `/init` |
| `package.json#publishConfig.access` not "public" or "restricted" | `PUBLISHCONFIG_ACCESS_INVALID` | fail | no | manual edit |
| `package.json#publishConfig.provenance: true` AND `package.json#publishConfig.registry` is not the public npm registry | `REGISTRY_PROVENANCE_CONFLICT` | fail | no | manual (mirrors npm-trust's check) |
| `.github/workflows/release.yml` missing | `RELEASE_WORKFLOW_MISSING` | fail | yes | `/init` |
| `.github/workflows/release.yml` content drift (sha256 ≠ cached `trust.workflowFileHash`) | `WORKFLOW_DRIFT` | warn | yes | `/init --refresh-yml` |
| `.solo-npm/state.json` schema below v3 | `STATE_JSON_SCHEMA_OUTDATED` | warn | yes | `/migrate` |
| `LICENSE` missing AND `package.json#private != true` AND `package.json#publishConfig.access == "public"` | `LICENSE_MISSING` | warn | no | manual (publishing without one is a real concern) |
| `CHANGELOG.md` missing | `CHANGELOG_MISSING` | warn | no | manual |
| `prepare-dist` declared in devDeps but not wired in release.yml | `PREPARE_DIST_NOT_WIRED` | warn | yes | `/init --refresh-yml --with-prepare-dist` |
| `prepare-dist` wired in release.yml but not in devDeps | `PREPARE_DIST_NOT_INSTALLED` | warn | no | manual (`pnpm add -D prepare-dist@latest`) |
| `package.json#name` doesn't match git remote slug pattern | `NAME_REMOTE_MISMATCH` | warn | no | informational |

Workflow file-hash cache: if the cached `trust.workflowFileHash` exists, recompute sha256 of the live file and compare. Emit `WORKFLOW_DRIFT` on mismatch.

### 2.4 — `security-gate` domain

Read-only checks against caches. Do NOT run `npm audit` / `gitleaks` / `osv-scanner` here — that's `/audit`, `/secrets-audit`, `/supply-chain` respectively. Doctor surfaces the cached state and counts.

| Check | Issue code | Severity | Auto-fixable | Remediation |
|---|---|---|---|---|
| `state.json#audit.tier1Count > 0` | `AUDIT_TIER1_PRESENT` | fail | no | `/audit --interactive` to triage; chain into `/deps` |
| `state.json#audit.lastFullScan` older than 7 days | `AUDIT_STALE` | warn | no | `/audit` |
| `state.json#audit.lastFullScan` is null | `AUDIT_NEVER_RUN` | warn | no | `/audit` |
| Cached `/secrets-audit` shows any high-severity findings | `SECRETS_HIGH_FINDING` | fail | no | `/secrets-audit` for re-scan + remediation |
| Cached `/supply-chain` shows any high-risk dep | `SUPPLY_CHAIN_HIGH_RISK` | warn | no | `/supply-chain --interactive` for triage |
| Cached `/lockfile-audit` shows suspicious diff | `LOCKFILE_SUSPICIOUS_DIFF` | warn | no | `/lockfile-audit` for re-scan |

If any `*_NEVER_RUN` or stale-cache issue is present, suggest running the corresponding skill before /release.

### 2.5 — `publish-readiness` domain

Per-package state. Source: `npm-trust --doctor --json` (reused from 2.1) for `packages[]` with `latestVersion` / `lastSuccessfulPublish` / `unpackedSize`.

| Check | Issue code | Severity | Auto-fixable | Remediation |
|---|---|---|---|---|
| Package has `published: false` | `PACKAGE_NOT_PUBLISHED` | warn | no | `/release` (first publish) |
| Package has `lastSuccessfulPublish` older than 365 days | `PACKAGE_STALE_RELEASE` | warn | no | informational (consider /deprecate or fresh release) |
| Package's `dist/` directory does not exist (per fs check) | `DIST_MISSING` | warn | no | manual (`pnpm run build`) |
| Package's `dist/` directory's mtime is older than its src/ mtime | `DIST_OLDER_THAN_SRC` | warn | no | manual (`pnpm run build`) |
| Workspace shape mismatch with last `/release`'s expectation (e.g., monorepo grew a new package not in `trust.configured`) | `WORKSPACE_NEW_PACKAGE` | warn | yes | `/trust --packages <newpackages>` |
| `dist-tag.latest` points at a pre-release version (semver `-suffix`) | `DIST_TAG_LATEST_PRERELEASE` | warn | no | `/dist-tag` to repoint |

## Phase 3 — Aggregate

Compose the `DoctorReport`:

```ts
interface DoctorReport {
  schemaVersion: 1;
  cli: { soloNpmVersion: string };          // from .claude/settings.json or hardcoded
  scope: {
    workspace: 'single-package' | 'pnpm-workspace' | 'npm-workspace';
    packageCount: number;
  };
  toolchain: {
    npmTrust: { version: string; level: 1|2|3 } | null;
    prepareDist: { version: string; level: 1|2|3 } | null;
  };
  domains: {
    trust:             DomainReport;
    provenance:        DomainReport;
    configPublish:     DomainReport;
    securityGate:      DomainReport;
    publishReadiness:  DomainReport;
  };
  summary: { ok: number; warn: number; fail: number; total: number };
  remediation: {
    autoFixable: Array<{ issueCode: string; skill: string; safe: true }>;
    manual:      Array<{ issueCode: string; skill: string; hint: string }>;
  };
}

interface DomainReport {
  ok: number;
  warn: number;
  fail: number;
  issues: ReadonlyArray<DoctorIssue>;
}

interface DoctorIssue {
  domain: 'trust' | 'provenance' | 'configPublish' | 'securityGate' | 'publishReadiness';
  severity: 'ok' | 'warn' | 'fail';
  code: string;
  message: string;
  remedy?: string;
  remediationSkill?: string;
  remediationArgs?: string[];
  autoFixable: boolean;
  relatedField?: string;
}
```

## Phase 4 — Render

### Default (human-readable)

```
solo-npm doctor — npm-trust@0.11.0 + prepare-dist@1.1.0  (toolchain probed 2h ago)

Repo: gagle/my-pkg @ pnpm-workspace (3 packages)

✓ Trust              all 3 packages trust-configured; OIDC binding intact
✓ Provenance         3 of 3 packages have SLSA attestations; latest 2026-05-08
✗ Config (publish)   2 issues:
                       ✗ RELEASE_WORKFLOW_MISSING   .github/workflows/release.yml not found
                                                    → run: /solo-npm:init
                       ⚠ LICENSE_MISSING            no LICENSE at repo root (publishing public)
                                                    → manual: pick MIT/Apache-2.0/ISC and add
⚠ Security gate      1 issue:
                       ⚠ AUDIT_STALE                last full audit 14 days ago
                                                    → run: /solo-npm:audit
✓ Publish readiness  all packages have current dist/ + recent successful publish

Summary: 7 ok, 2 warn, 1 fail.
Remediation:
  Auto-fixable (1):  RELEASE_WORKFLOW_MISSING → /solo-npm:init
  Manual (2):        LICENSE_MISSING, AUDIT_STALE
```

### --json

Emit the full `DoctorReport` JSON to stdout. Exit code:
- 0 if `summary.fail === 0`
- 1 otherwise (or `--strict` mode: 1 if any warn or fail)

### --fix mode (chains to remediation)

After Phase 3 aggregation, partition issues by `autoFixable`. For the auto-fixable ones, render:

```
Auto-fixable issues found:
  [1] RELEASE_WORKFLOW_MISSING (config-publish) → /solo-npm:init
  [2] WORKFLOW_DRIFT          (config-publish) → /solo-npm:init --refresh-yml
  [3] STATE_JSON_SCHEMA_OUTDATED (config-publish) → /solo-npm:migrate

Apply all? Apply selected? Skip?
```

Use `AskUserQuestion` (load via `ToolSearch query="select:AskUserQuestion"` if needed) to gate:
- `header`: "Apply auto-fixes"
- `question`: "Which auto-fixable issues to apply?"
- options:
  1. `Apply all` — chain into each remediation skill in order
  2. `Apply selected` — second AskUserQuestion with multiSelect over the issues
  3. `Skip` — no changes; exit with the original report

For each selected fix:
1. Chain into `remediationSkill` with `remediationArgs`.
2. Capture the chain target's exit code; on failure, surface the diagnostic and STOP per H6.
3. After all selected fixes apply, re-run Phase 2 probes to verify resolution.

Final render shows:
- Original issue list
- Applied fixes (with chain-target exit codes)
- Remaining issues (manual + any that didn't auto-resolve)

## Error-handling patterns (H1–H8 from `/unpublish` reference)

- **H2** — `.solo-npm/state.json` corruption guard at every read.
- **H6** — Chain-target failure recovery in `--fix` mode: surface the chained skill's diagnostic verbatim before continuing or stopping per the `STOP_ON_FIRST_FAILURE` flag (default true).
- **H7** — Atomic state.json writes when `--fix` chains call into skills that update caches.
- **H8** — Rate-limit backoff if probes (npm view via npm-trust) hit 429.

## Notes

- Doctor is **read-only by default**. Only `--fix` mode mutates state; even then, mutations happen via chained skills, not Doctor directly.
- Doctor does NOT run quality gates (lint/test/build) — that's `/verify`. Doctor checks **CONFIG-time** state of these tools (e.g., release.yml structure) but doesn't EXECUTE them.
- Doctor does NOT diagnose project-build concerns (tsconfig strictness, vitest coverage thresholds, eslint config, README quality, daily-dev tooling) — those are out of solo-npm's narrowed scope (per the v0.19.0 PART III decision).
- Cache TTL: Doctor reuses the `state.json#toolchain` cache (1 day), the `state.json#trust` cache (7 days), the `state.json#audit` cache (1 day). For accurate one-shot reports, use `--fresh` to bypass all caches.
- The H1-H8 pattern catalog is available at `/unpublish` Phase −1.5+. Doctor's `--fix` mode applies H6 (chain-failure recovery) and H7 (atomic writes) when chained skills mutate state.json.
