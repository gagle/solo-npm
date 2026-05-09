---
description: Remove a workspace package from a monorepo — chains /deprecate + /unpublish + filesystem removal + CHANGELOG update. Triggers from prompts like "remove the foo package", "drop @scope/foo from the monorepo", "delete a workspace package", "unpublish and remove a package". Destructive — two-step confirmation. Currently only `remove` operation is implemented; `add` and `deps` are deferred per the v0.19.0 narrowed scope.
---

# Workspace

Monorepo package lifecycle ops. v0.19.0 ships only the `remove` operation (publish-relevant); `add` and `deps` are out of scope (build-time concerns deferred per PART III).

## Operations

| Operation | Trigger | Status |
|---|---|---|
| `remove` | `/solo-npm:workspace remove <pkg-name>` | **IN v0.19.0** — chains /deprecate + optional /unpublish + fs removal |
| `add` | (future) | DEFERRED — adding a package is build-time, not publish-time |
| `deps` | (future) | DEFERRED — cross-package dep sync is build-time |

## Phase −0 — Help mode

If `--help`/`-h`/similar, surface brief summary and STOP.

## When to use (`remove`)

- Retiring a workspace package permanently (the npm tarball stays as `@deprecated`)
- Cleaning up a published-by-accident scratch package within npm's 72h unpublish window
- Reorganizing a monorepo where a package was renamed (deprecate the old name; the renamed package is the new home)

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `OPERATION` | trigger phrase | `remove` (only operation in v0.19.0) |
| `PACKAGE_NAME` | trigger phrase or `--package <name>` | required |
| `MODE` | inferred or `--mode <deprecate-only\|deprecate-and-unpublish>` | ask the user via AskUserQuestion |
| `JSON_OUT` | `--json` | `false` |

## Phase A — Pre-flight

1. Detect monorepo. If single-package, STOP — `/workspace remove` is monorepo-only. (For single-package retirement, use `/deprecate` + `/unpublish` directly.)
2. Verify `<PACKAGE_NAME>` is a workspace package (resides under `packages/<name>` or matches a workspace glob).
3. Verify `<PACKAGE_NAME>` is published on npm (via `npm view <name> version`). If not published, skip Phase B and Phase C; just remove from filesystem.
4. Read `.solo-npm/state.json#trust.configured` to confirm trust was set up; informational only.

## Phase B — Plan + confirm (DESTRUCTIVE — two-step gate per H1 convention)

Render the destruction plan:

```
Workspace remove plan

Package:        @scope/foo
Latest version: 1.5.2 (published 14 days ago)
Workspace dir:  packages/foo (12 files, 4 KB src)

Operations:
  1. Deprecate on npm:    npm deprecate @scope/foo "Removed from monorepo as of 2026-05-09"
  2. (Optional) Unpublish: npm unpublish @scope/foo --force  (only valid in 72h window)
  3. Remove from filesystem: git rm -r packages/foo
  4. Update root CHANGELOG with removal note
```

First AskUserQuestion gate:

- `header`: "Removal plan"
- `question`: "Proceed with workspace package removal? Once on npm, deprecation cannot be undone (you can update the deprecation message but not 'undeprecate'). Unpublishing is even more permanent."
- options:
  1. `Deprecate only` — npm deprecate + fs remove. Tarball stays on npm.
  2. `Deprecate AND unpublish` — both. Tarball is fully removed from npm. Only valid within 72h of publish.
  3. `Abort` — no changes.

If `Deprecate AND unpublish` selected AND outside the 72h window, surface the npm policy and downgrade to `Deprecate only`.

Second AskUserQuestion gate (per H1 destructive double-confirm):

- `header`: "Confirm"
- `question`: "Last chance — confirm removal of @scope/foo from npm + filesystem?"
- options:
  1. `Confirm — proceed`
  2. `Abort`

## Phase C — Execute

### C.1 — Deprecate on npm (chain into /deprecate)

```bash
/solo-npm:deprecate @scope/foo --reason "Removed from monorepo as of <DATE>"
```

Capture exit code; on failure surface verbatim per H6.

### C.2 — Optional: Unpublish (chain into /unpublish)

If user selected `Deprecate AND unpublish` AND within 72h window:

```bash
/solo-npm:unpublish @scope/foo --force
```

Per H1, /unpublish handles the web 2FA gate; this skill just chains.

### C.3 — Filesystem removal

```bash
git rm -rf packages/<name>
```

If the package directory contains uncommitted work, STOP and surface; user must commit or stash first.

### C.4 — Update root CHANGELOG

Prepend a removal note to root `CHANGELOG.md`:

```markdown
## v<version> — workspace package removed

- **Removed**: `@scope/foo` (was at 1.5.2). Deprecated on npm with reason: "Removed from monorepo as of 2026-05-09."
```

### C.5 — Update .solo-npm/state.json

Remove `<PACKAGE_NAME>` from `state.json#trust.configured` (atomic write per H7).

## Phase D — Commit

```bash
git add -A
git commit -m "chore: remove @scope/foo from monorepo

Deprecated on npm. Tarball ${UNPUBLISHED:+'unpublished'} from registry.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>"
```

Do NOT push automatically. The user pushes after reviewing.

## Phase E — Render report

```
Workspace remove — @scope/foo

  ✓ Deprecated on npm with reason: "Removed from monorepo as of 2026-05-09"
  ✓ Removed packages/foo (12 files)
  ✓ Updated CHANGELOG.md
  ✓ Updated .solo-npm/state.json
  ✓ Committed as: chore: remove @scope/foo from monorepo

Next:  Push when ready: git push
```

## --json

```ts
interface WorkspaceRemoveReport {
  schemaVersion: 1;
  package: { name: string; lastVersion: string; dir: string };
  deprecated: boolean;
  unpublished: boolean;
  filesystemRemoved: boolean;
  changelogUpdated: boolean;
  stateUpdated: boolean;
  committed: { sha: string } | null;
  summary: { complete: boolean };
}
```

Exit codes:
- 0 on full success
- 60 (PARTIAL_FAILURE) if some steps complete but others fail
- 21 (OTP_REQUIRED) if /deprecate or /unpublish needed web 2FA
- 10 (CONFIGURATION_ERROR) if not a monorepo, package not found, etc.

## Error-handling

- **H1** OTP/2FA: /deprecate and /unpublish drive their own H1 flows; this skill chains and surfaces their output.
- **H3** auth-window race: if /unpublish is invoked, it re-checks `npm whoami` per H3 conventions.
- **H6** chain-failure recovery: capture each chain target's diagnostic; surface verbatim; STOP on first failure (not "best effort").
- **H7** atomic state.json writes.

## Notes

- This skill is destructive. The two-step AskUserQuestion gate is mandatory; do NOT skip with `--yes` or similar (none implemented for v0.19.0).
- Deprecation is reversible-ish (you can update the message, not un-deprecate). Unpublish is irreversible after the 72h window.
- For renaming a package (vs removing), use `/unpublish rename` (separate skill).
- Solo-npm DOES NOT remove npm-registry-side trust binding — npm's OIDC configuration outlives the package itself. To clean up trust, the user manages the binding via npm's web UI separately.
- For monorepo /workspace add (scaffold a new package), see PART II Theme I.1 — deferred to a future minor; in v0.19.0, add a new package manually then run `/init` (release-side) and `/trust` (OIDC) for the new addition.
