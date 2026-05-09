---
description: Verify package.json#exports map points at files that actually exist in dist/. Catches "export points to renamed file" — common after refactors that forget to update exports. Triggers from prompts like "check exports", "validate package.json exports", "are my exports correct?", "exports orphans". Use as /verify Tier 6 gate. Pure read-only fs check.
---

# Exports check

Walk every entry in `package.json#exports` and verify the referenced file exists in `dist/`. Walk `dist/` and find files NOT referenced by any exports entry (orphans). Catches a class of bugs that survives lint, tests, and even a successful `pnpm pack` — the kind that surfaces only when a consumer tries to import a renamed-but-not-re-exported entry.

## Phase −0 — Help mode

If the prompt contains `--help`/`-h`/similar, surface a brief summary and STOP. Note: pure read-only fs check; no external deps.

## When to use

- After refactoring `src/` (renaming files, splitting modules, reorganizing entry points)
- Before `/release` to ensure tarball entries are well-formed
- As `/verify` Tier 6 (default-on for v0.19.0+)
- After upgrading the build tool (some bundlers emit different output paths)

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `STRICT` | `--strict` flag or "strict" in prompt | `false` (warn-only) |
| `JSON_OUT` | `--json` flag | `false` |
| `PATH` | `--path <dir>` | repo root |

## Phase 1 — Pre-flight

1. Verify `<PATH>/package.json` exists; STOP if missing.
2. Verify `<PATH>/dist/` exists; STOP with "build first" hint if missing.

## Phase 2 — Parse package.json#exports

Read `package.json#exports`. The shape can be:

- A single string: `"exports": "./dist/index.js"` (treat as `{".": "./dist/index.js"}`)
- A flat map: `"exports": { "./vite": "./dist/vite.js" }`
- A conditional map: `"exports": { ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" } }`
- Nested: `"exports": { ".": { "import": { "types": "...", "default": "..." }, "require": { ... } } }`

Walk recursively; collect every leaf string value. Each leaf is a relative path that should exist in the tarball publish root.

If `exports` is missing entirely, fall back to `main`/`types` at the package.json root level.

## Phase 3 — Verify each export entry

For each `(pattern, condition, target)` triple:

```bash
RESOLVED=$(realpath "<PATH>/$target" 2>/dev/null)
if [ -f "$RESOLVED" ]; then
  STATUS=ok
else
  STATUS=missing
fi
```

For `types` conditions, verify the file ends in `.d.ts` (or `.d.mts` / `.d.cts`).

For non-types conditions, verify the file ends in `.js` (or `.mjs` / `.cjs`).

Type-mismatch (e.g., types pointer to `.js`) is a `fail`-severity issue.

## Phase 4 — Detect orphans

Walk `<PATH>/dist/` recursively. For every `.js`/`.mjs`/`.cjs`/`.d.ts`/`.d.mts`/`.d.cts` file, check if it's referenced by any exports entry. If not, mark as orphan.

Exclude these from orphan detection:
- `*.map` files (sourcemaps)
- Files that match a `subpath pattern` in exports (e.g., `"./subpath/*": "./dist/subpath/*.js"` matches anything under that path)

## Phase 5 — Render

### Human (default)

```
exports-check — sample-pkg@1.5.2 (./dist)

Exports map:
  "."              types     ./dist/index.d.ts        ✓
                   default   ./dist/index.js          ✓
  "./vite"         types     ./dist/vite.d.ts         ✓
                   default   ./dist/vite.js           ✓

Orphans (built but not referenced):
  ./dist/internal-helper.js              (consider adding to exports or excluding from build)

Summary: 4 entries OK, 0 missing, 1 orphan.
```

When entries are missing:

```
  "./bin"          default   ./dist/bin.js            ✗ MISSING
                   types     ./dist/bin.d.ts          ✗ MISSING

Summary: 2 entries OK, 2 missing, 0 orphans.
Action: rebuild OR remove these entries from package.json#exports.
```

### --json

```ts
interface ExportsCheckReport {
  schemaVersion: 1;
  exports: ReadonlyArray<{
    pattern: string;
    conditions: ReadonlyArray<{
      condition: string;
      target: string;
      exists: boolean;
      typeMismatch: boolean;
    }>;
  }>;
  orphans: ReadonlyArray<{
    file: string;
    suggestedExport: string | null;
  }>;
  summary: {
    entries: number;
    missing: number;
    orphans: number;
    ok: boolean;
  };
}
```

Exit code:
- 0 if `summary.missing === 0` and (no orphans OR `STRICT=false`)
- 40 (validation failure range) otherwise

## Integration points

- `/verify` Tier 6
- `/release` Phase A optional gate via `--exports-strict`
- `/doctor` — does NOT run this directly; but consumes cached results when present

## Error-handling

- **H7** — atomic state.json updates if results are cached (current implementation: cache only the summary block per-version).

## Notes

- Orphans are warn-only because some legitimate patterns (internal helpers, subpath wildcards expanded) intentionally produce dist files not referenced by exports. `--strict` makes them fail.
- For monorepos, run per-package; aggregate output.
- This skill complements `/types` (attw): exports-check verifies *the targets exist*; attw verifies *the resolution semantics are coherent*. Both are needed.
