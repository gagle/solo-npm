---
description: Detect suspicious lockfile diffs — defensive against supply-chain attacks where malicious deps slip in via PR. Categorizes lockfile changes (additions, removals, version bumps, integrity hash changes) and flags suspicious patterns. Triggers from prompts like "audit lockfile diff", "is this lockfile safe?", "check for suspicious dep changes", "lockfile review". Use as a release-gate or PR review aid.
---

# Lockfile audit

Compare the current `pnpm-lock.yaml` against a baseline (PR base, last release tag, etc.) and surface suspicious patterns. Defensive against supply-chain attacks where a malicious update is slipped in via a PR or where dependency injection happens between checks.

## Phase −0 — Help mode

If `--help`/`-h`/similar, surface brief summary and STOP.

## When to use

- Reviewing a PR that touches `pnpm-lock.yaml`
- Pre-release safety check ("is the lockfile drift since last release safe to ship?")
- After running `pnpm update` to verify no surprises
- Investigating a specific dep flagged by `/supply-chain`

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `BASE` | `--base <ref>` flag | `HEAD~` for PR review; last release tag for pre-release |
| `HEAD` | `--head <ref>` flag | `HEAD` |
| `JSON_OUT` | `--json` | `false` |
| `STRICT` | `--strict` | `false` (suspicious-only is warn by default) |

## Phase 1 — Diff lockfiles

```bash
git show "$BASE:pnpm-lock.yaml" > /tmp/lockfile-base.yaml 2>/dev/null
git show "$HEAD:pnpm-lock.yaml" > /tmp/lockfile-head.yaml 2>/dev/null
```

If either ref doesn't have a lockfile, surface and STOP.

## Phase 2 — Parse and categorize

Parse both lockfiles into deduplicated `Map<name, { version, integrity }>` per the lockfile format (pnpm uses YAML; npm uses JSON).

Compare:

| Pattern | Detection |
|---|---|
| Addition | Name present in head, not in base |
| Removal | Name present in base, not in head |
| Version bump | Same name; different version |
| Integrity hash change WITHOUT version change | Same name; same version; different integrity (RED FLAG) |
| Direct vs transitive | Direct = listed in `package.json`; transitive = not |

## Phase 3 — Flag suspicious patterns

| Pattern | Severity |
|---|---|
| Integrity hash change without version change | fail (force-push attack signature) |
| New transitive dep NOT in any chain from package.json (orphan) | fail |
| Version downgrade (major or minor) | warn |
| New direct dep added (suggests pkg.json changed too — verify intent) | warn |
| New transitive dep with no maintainer count signal | warn |
| 10+ deps changed in one commit | warn (unusual; verify intentional) |

## Phase 4 — Render

### Human

```
lockfile-audit  base: HEAD~  head: HEAD

Additions:
  + chalk@5.4.1                  (transitive via vitest@4.1.4)
  + node-fetch@3.3.2             (direct dep — was this intentional?)

Removals:
  - left-pad@1.0.5               (direct removal)

Version bumps:
  ~ vitest@4.1.0 → 4.1.4         (transitive)

Suspicious patterns:
  ⚠ NONE detected.

Summary: 4 changes. 0 suspicious.
```

When suspicious patterns present:

```
Suspicious patterns:
  ✗ left-pad@1.0.5 — integrity hash changed without version change
    base:  sha512-aaa...
    head:  sha512-bbb...
    → Possible force-push attack on the registry. Investigate before merging.

  ⚠ orphan-pkg@9.9.9 — added as transitive but not reachable from package.json
    → Could be malicious injection. Verify the chain.
```

### --json

```ts
interface LockfileAuditReport {
  schemaVersion: 1;
  base: string;
  head: string;
  additions: Array<{ name: string; version: string; reason: 'direct'|'transitive'|'orphan' }>;
  removals:  Array<{ name: string; version: string }>;
  bumps:     Array<{ name: string; old: string; new: string; severity: 'patch'|'minor'|'major'|'downgrade' }>;
  suspicious: Array<{
    name: string;
    pattern: 'hash-change-no-version' | 'orphan-dep' | 'version-downgrade' | 'untracked-direct';
    detail: string;
    severity: 'warn' | 'fail';
  }>;
  summary: { totalChanges: number; suspiciousCount: number; failCount: number };
}
```

Exit codes:
- 0 if `failCount === 0` and (no warns OR `STRICT=false`)
- 40 (VALIDATION_FAILURE) if any fail OR (`STRICT=true` AND any warn)

## Integration

- PR-level gate: invoke from a `gh pr` review action with `--base origin/main --head HEAD`
- `/release` Phase A optional gate via `--lockfile-strict`: STOP if any fail
- `/doctor`'s `security-gate` domain surfaces cached lockfile-audit summary
- Cache: write summary to `state.json#caches.lockfileAudit[<base>..<head>]` with TTL 1 hour

## Error-handling

- **H7** atomic state.json writes for cache updates.
- **H6** if base ref doesn't exist (e.g., shallow clone), STOP with diagnostic.

## Notes

- This skill cannot prevent attacks; it surfaces signals. The user must decide.
- Integrity hash changes WITHOUT version changes are the classic supply-chain attack signature — these warrant immediate investigation.
- For first runs (no cached previous diff), this skill compares HEAD vs HEAD~. Useful for "what did this commit change in the lockfile?"
- Works for `pnpm-lock.yaml`. `package-lock.json` parsing is similar but not implemented in v0.19.0; use `--lockfile-format=npm` to opt in (deferred to future).
