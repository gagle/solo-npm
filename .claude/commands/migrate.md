---
description: Upgrade an existing solo-npm-using repo from older conventions to current — handles state.json schema migrations (v1→v2→v3), pin convention bumps (npm-trust@^X → @latest), release.yml drift (chains into /init --refresh-yml), husky/commitlint setup updates. Triggers from prompts like "upgrade my solo-npm setup", "migrate state.json", "fix conventions drift", "bring repo up to current". Idempotent — safe to re-run.
---

# Migrate

Upgrade between solo-npm versions. Each migration is idempotent (safe to re-run); the skill detects the current state and applies only what's needed.

## Phase −0 — Help mode

If `--help`/`-h`/similar, surface brief summary and STOP. Note: idempotent; safe to invoke whenever.

## When to use

- After upgrading the solo-npm plugin (re-installing the marketplace plugin)
- When `/doctor` flags `STATE_JSON_SCHEMA_OUTDATED` or convention drift
- After noticing release.yml uses pre-OIDC patterns or other dated structures
- Initial onboarding of a repo that pre-dates current conventions

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `JSON_OUT` | `--json` | `false` |
| `DRY_RUN` | `--dry-run` | `false` (preview without writing) |
| `MIGRATIONS` | `--only <name1,name2>` | all detected applicable migrations |

## Phase 1 — Detect current state

```bash
SOLO_NPM_VERSION=$(node -p "require('./.claude/settings.json').enabledPlugins['solo-npm@gllamas-skills']" 2>/dev/null || echo "unknown")
STATE_SCHEMA=$(jq -r '.version // 0' .solo-npm/state.json 2>/dev/null || echo 0)
NPMTRUST_PIN=$(grep -o 'npm-trust@[^"]*' .claude/commands/trust.md 2>/dev/null | head -1 || echo "unknown")
WORKFLOW_HASH=$(sha256sum .github/workflows/release.yml 2>/dev/null | cut -d' ' -f1 || echo "")
```

Detect drift:

| Drift | Detection |
|---|---|
| State schema v1 → v2 | `state.json#version === 1` |
| State schema v2 → v3 | `state.json#version === 2` (no toolchain field) |
| State schema v0 (pre-versioning) | no `version` field |
| Pinned npm-trust convention | trust.md mentions `npm-trust@^X.Y` instead of `@latest` |
| Pinned prepare-dist convention | similar to above |
| release.yml drift | `state.json#trust.workflowFileHash` exists AND doesn't match current sha256 |
| Old husky pattern | `.husky/pre-commit` uses husky v8 syntax instead of v9 |
| /init scaffold drift | files written by /init are stale (compared to current /init scaffold output) |

## Phase 2 — Render migration plan

```
solo-npm migrate

Detected:
  solo-npm version:  v0.18.0 (latest: v0.19.0)
  State schema:      v2 (current: v3)
  npm-trust pin:     @latest ✓ (current convention)
  prepare-dist pin:  @^1.1 (current: @latest)
  release.yml:       drift detected (last run was on v0.17.0's template)

Migrations to apply:
  [1] state-v2-to-v3              — adds toolchain cache + pkgCheck section
  [2] prepare-dist-pin-to-latest  — replaces @^1.1 with @latest in skill bodies (none today; user-configured)
  [3] release-yml-refresh         — chain into /init --refresh-yml

  Skipped (already current):
  - state-v0-to-v1
  - state-v1-to-v2
  - npm-trust-pin-to-latest
  - husky-v8-to-v9

Proceed?
```

If `DRY_RUN`, exit here with the report and do NOT apply.

AskUserQuestion gate before applying:

- `header`: "Apply migrations"
- `question`: "Apply N migrations? Each is idempotent and safe to re-run."
- options:
  1. `Apply all` — chain into each migration
  2. `Apply selected` — multiSelect over the list
  3. `Skip — review only`

## Phase 3 — Apply migrations

Each migration is implemented as a small, idempotent change:

### state-v0-to-v1
Add `version: 1` field. Add empty `trust` and `audit` sections.

### state-v1-to-v2
Add `trust.workflowFileHash: null`. Bump version to 2. First /release after this migration takes the cold path to populate the hash.

### state-v2-to-v3
Add `pkgCheck: { lastSize: {}, lastFullScan: null, ttlDays: 1 }`. Add `toolchain: { lastProbe: null, ttlDays: 1, tools: {} }`. Bump version to 3. First /doctor after this migration takes the cold path to probe the toolchain.

### npm-trust-pin-to-latest
Replace `npm-trust@^X.Y` with `npm-trust@latest` in `.claude/commands/trust.md` (or other skill bodies that reference the pin). Per the v0.18.0 convention, solo-npm tracks @latest with runtime capability probes.

### prepare-dist-pin-to-latest
Same as above for prepare-dist references.

### release-yml-refresh
Chain into `/solo-npm:init --refresh-yml`. The init skill detects drift and offers to refresh the yml.

### husky-v8-to-v9
Update `.husky/pre-commit` to v9 syntax: drop the `#!/bin/sh` shebang and `. "$(dirname "$0")/_/husky.sh"` line; just leave the bare command. Detect via grep for `husky.sh` source.

Each migration:
1. Backs up the file (write to `.tmp` first per H7)
2. Applies the change
3. Validates the result (parses JSON; runs sanity check)
4. Commits via atomic rename
5. Records in `state.json#migrations: { applied: ['state-v2-to-v3', ...] }` so re-runs skip already-applied migrations

## Phase 4 — Render report

```
solo-npm migrate — applied 3 migrations

  ✓ state-v2-to-v3              state.json bumped from v2 to v3
  ✓ prepare-dist-pin-to-latest  trust.md updated (1 substitution)
  ✓ release-yml-refresh         release.yml refreshed via /init --refresh-yml

Skipped (already current): 5
Failed: 0

Next: review changes via `git diff`; commit when ready.
```

## --json

```ts
interface MigrateReport {
  schemaVersion: 1;
  detected: {
    soloNpmVersion: string;
    stateSchemaVersion: number;
    pinConventions: { 'npm-trust': string; 'prepare-dist': string };
    releaseYmlDrift: boolean;
  };
  migrations: ReadonlyArray<{
    name: string;
    status: 'applied' | 'skipped-already-current' | 'failed';
    detail?: string;
  }>;
  summary: { applied: number; skippedAlreadyCurrent: number; failed: number };
}
```

Exit codes:
- 0 if all selected migrations applied or already-current
- 60 (PARTIAL_FAILURE) if some applied but others failed
- 1 if any catastrophic failure during application (typically state.json corruption — H2)

## Migration registry

The list of known migrations lives inline in this skill (for v0.19.0). When v0.20.0 introduces new conventions, this skill body gains additional migration entries. Each entry is a small idempotent function — additions are append-only.

For very-future-self: extracting this into `solo-cli-conventions/migrations/*.ts` is on the deferred list; for now inline.

## Error-handling

- **H2** state.json corruption guard: parse failures wrap in try/catch; on fail, surface diagnostic and STOP (don't lose data by writing over a corrupt file).
- **H6** chain-failure recovery: when migrations chain into other skills (`/init --refresh-yml`), capture diagnostics and surface verbatim.
- **H7** atomic writes: every file edit goes through `.tmp` rename.

## Notes

- Idempotency: re-running migrate is always safe. State is checked at each step; already-applied migrations are skipped.
- The migration history is persisted in `state.json#migrations.applied` so the skill knows what's been done. This is the only mutable state introduced by /migrate beyond the migrations themselves.
- For the rare case where a migration goes wrong, recovery is manual: revert the file via `git restore`, run `/migrate` again with `--only <other-migrations>`.
