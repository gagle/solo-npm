---
description: Extract the public API surface from current dist/, diff against the last released version's surface, classify the diff as patch/minor/major. Detects accidental breaking changes BEFORE /release tags. Triggers from prompts like "check API surface", "did I break consumers?", "validate version bump", "API diff vs last release", "what changed publicly?". Use as /release Phase A hard gate. The single highest-leverage skill for solo-dev breaking-change prevention.
---

# Public API

Extract the public API surface from the built `dist/` (declarations), diff against the last released version's surface, classify the diff. Surfaces breaking changes BEFORE `/release` tags an incorrectly-versioned bump — the key safety net for solo devs without a PR reviewer to catch accidental breaks.

This is the **single highest-leverage** new skill in v0.19.0: a wrong semver bump is silently published and breaks every consumer who picks it up. /public-api detects the mismatch before the tag goes out.

## Phase −0 — Help mode

If the prompt contains `--help`/`-h`/similar, surface a brief summary and STOP. Note: this skill consumes built dist/ + downloads the last released tarball from npm; no other state is mutated.

## When to use

- Before `/release` (the recommended hard gate; default-on in v0.19.0+ via `/release` Phase A)
- After significant refactor to confirm the public surface is intact
- To suggest the right version bump given source changes
- CI gate via `--json` exit codes

## Operations

| Operation | Trigger | Effect |
|---|---|---|
| `check` (default) | `/solo-npm:public-api` | Extract → fetch baseline → diff → classify → report |
| `extract` | `/solo-npm:public-api extract` | Just extract current surface; emit JSON; no baseline |
| `compare-bump` | `/solo-npm:public-api --against <version>` | Compare current vs specific older version (not just latest) |

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `OPERATION` | trigger | `check` |
| `BASELINE_VERSION` | `--against <version>` | last released (per `npm view <pkg>@latest`) |
| `REQUESTED_BUMP` | `--bump <patch\|minor\|major>` | inferred from package.json#version diff vs registry |
| `STRICT` | `--strict` | `true` by default for /release; `false` for ad-hoc |
| `JSON_OUT` | `--json` | `false` |
| `PATH` | `--path <dir>` (monorepo) | repo root |

## Phase 1 — Build & locate current surface

1. Verify `<PATH>/dist/` exists. If missing, **STOP** with "run `pnpm run build` first."
2. Locate the `.d.ts` entry: read `package.json#exports["."].types` (preferred) OR `package.json#types` OR `dist/index.d.ts` (last resort).
3. If types entry is missing entirely, surface `NO_TYPES_ENTRY` warning. The skill can still partially extract (fall back to walking `dist/*.d.ts`) but will be less reliable.

## Phase 2 — Extract current surface

The extractor walks the entry `.d.ts` and follows `export` re-exports recursively. For each exported symbol, capture:

- `name` — symbol name as exported (handle `export { X as Y }` → publishedName=Y)
- `kind` — function | class | interface | type | const | enum | namespace
- `signature` — text of the declaration (function: `(args) => return`; class: full body; type: aliased shape; const: typed value)
- `members` — for class/interface/namespace: nested public symbols
- `optional` flag — for function args, type members
- `readonly` flag — for properties
- `genericParams` — for generic types

**Implementation approach** — use a custom AST walker via TypeScript's compiler API (`typescript` is a peer-deepDep in solo-dev TS projects):

```typescript
import ts from 'typescript';

function extractAPISurface(entryDts: string): APISurface {
  const program = ts.createProgram([entryDts], { /* read-only declaration parse */ });
  const sourceFile = program.getSourceFile(entryDts);
  const checker = program.getTypeChecker();
  const symbols: APISymbol[] = [];

  ts.forEachChild(sourceFile, (node) => {
    if (ts.isExportDeclaration(node) || ts.isExportAssignment(node) || ts.isFunctionDeclaration(node) ...) {
      // walk and extract; follow re-exports across files
    }
  });

  return { symbols };
}
```

Fallback: if the project doesn't have `typescript` available (rare), fall back to a regex-based walker over the `.d.ts` text. Less precise but works.

## Phase 3 — Fetch baseline surface

```bash
PKG=$(node -p "require('./package.json').name")
LATEST=$(npm view "$PKG" version 2>/dev/null)
if [ -z "$LATEST" ]; then
  # Never released
  BASELINE_SURFACE=null
else
  TARBALL_URL=$(npm view "$PKG@$LATEST" dist.tarball)
  curl -sL "$TARBALL_URL" -o /tmp/baseline.tgz
  tar -xzf /tmp/baseline.tgz -C /tmp/baseline
  # find the d.ts entry the same way as Phase 2 (read /tmp/baseline/package/package.json#exports etc.)
  BASELINE_SURFACE=$(extractAPISurface /tmp/baseline/package/<entry>.d.ts)
fi
```

If `BASELINE_SURFACE === null` (first release), skip the diff entirely; surface "first release — no baseline" and classify as patch (anything is fine for first publish).

If `BASELINE_VERSION` is a pre-release (semver `-suffix`), fetch the latest STABLE tag instead via `npm view "$PKG" dist-tags.latest`. Pre-releases break compat by definition; comparing against them is misleading.

## Phase 4 — Diff

Compare `currentSurface.symbols[]` against `baselineSurface.symbols[]`:

```ts
interface APIDiff {
  added:   APISymbol[];
  removed: APISymbol[];
  changed: { current: APISymbol; baseline: APISymbol; reason: string }[];
}
```

For each symbol in baseline:
- Not in current → **REMOVED** (BREAKING)
- In current with different signature → **CHANGED** (BREAKING if narrowing/removing args/return type changes)
- In current with same signature → unchanged

For each symbol in current not in baseline → **ADDED** (non-breaking; minor)

Signature-equality rules (NORMATIVE):
- Function signature changes:
  - Adding optional parameter → non-breaking
  - Adding required parameter → BREAKING
  - Removing parameter → BREAKING
  - Changing parameter type to subtype/supertype → BREAKING (subtype narrows; supertype widens which still breaks consumers writing the broader type)
  - Changing return type → BREAKING (consumers may rely on the old shape)
- Interface member changes:
  - Adding optional member → non-breaking
  - Adding required member → BREAKING (consumers' implementations break)
  - Removing member → BREAKING
  - Changing member type → BREAKING
- Type alias changes: any change → BREAKING (consumers may have inferred the type)
- Class changes: same as interface for instance shape; constructor signature follows function rules

## Phase 5 — Classify

Map diff to bump:

| Diff state | Classification |
|---|---|
| Any `removed` OR any `changed` | **major** |
| Only `added` (no removed, no changed) | **minor** |
| No diffs at all | **patch** |
| First release (no baseline) | **patch** (any version is fine) |

Compare against `REQUESTED_BUMP` (inferred from `package.json#version` diff vs `npm view <pkg>@latest version`):

- `requested === classification` → ✓ correct bump
- `requested > classification` → ⚠ over-bumping (allowed; warn only)
- `requested < classification` → ✗ UNDER-BUMPING (BLOCKING in `/release` Phase A)

## Phase 6 — Render

### Human (default)

```
public-api — sample-pkg

Current   : 1.5.2 → 1.5.3 (proposed)
Baseline  : 1.5.2 (npm registry)

Diff:
  + 0 added
  ~ 0 changed
  - 0 removed

Classification: patch
Requested:      patch
Verdict:        ✓ correct
```

When breaking:

```
Diff:
  + 1 added       (new export `parseExtended`)
  ~ 1 changed     (function `parse` — added required parameter)
  - 1 removed     (function `parseLegacy` — was deprecated)

Classification: MAJOR (any removed or signature-narrowed)
Requested:      patch (1.5.2 → 1.5.3)
Verdict:        ✗ UNDER-BUMPING — bump version to 2.0.0 before /release

Breaking changes:
  - parse(input: string) → parse(input: string, options: ParseOptions)
                          (added required argument 2: options)
  - parseLegacy() removed entirely
                          (consumers must migrate to parse with options.legacy=true)
```

Suggest the next version: `1.5.2 → 2.0.0`.

### --json

```ts
interface PublicApiReport {
  schemaVersion: 1;
  current:  { version: string; surface: APISurface };
  baseline: { version: string | null; surface: APISurface | null };
  diff: {
    added:   ReadonlyArray<APIChange>;
    removed: ReadonlyArray<APIChange>;
    changed: ReadonlyArray<APIChange & {
      oldSignature: string;
      newSignature: string;
      reason: 'arg-added-required' | 'arg-removed' | 'return-type-changed' | 'member-removed' | …;
    }>;
  };
  classification: 'patch' | 'minor' | 'major';
  requestedBump: 'patch' | 'minor' | 'major' | null;
  verdict: 'ok' | 'over-bump' | 'under-bump';
  suggestedNextVersion: string;
  summary: { breaking: number; nonBreaking: number };
}

interface APISymbol {
  name: string;
  kind: 'function' | 'class' | 'interface' | 'type' | 'const' | 'enum' | 'namespace';
  signature: string;
  exported: boolean;
  members?: ReadonlyArray<APISymbol>;
  genericParams?: ReadonlyArray<string>;
  optional?: boolean;
  readonly?: boolean;
}
```

Exit codes:
- 0 if `verdict === 'ok'` or `verdict === 'over-bump'`
- 40 (VALIDATION_FAILURE range) if `verdict === 'under-bump'` AND `STRICT=true` (the `/release` default)
- 1 if extraction itself failed (couldn't locate dist or baseline)

## Integration with /release

`/release` Phase A invokes this skill BEFORE the version bump is staged. If `verdict === 'under-bump'` AND no `--api-strict=false` flag, /release STOPS with the precise bump suggestion. The user must either:
1. Bump version higher (the recommended path)
2. Pass `--api-strict=false` to override (rare, e.g., when a "removed" symbol is provably unused by anyone)

The convention: TS-library breaking changes always require a major bump. /public-api enforces this by default.

## Cache

Per-version surfaces are CACHED in `state.json#publicApi.surfaces[<pkg>@<version>]` (immutable; baseline never changes for a published version). First fetch downloads the tarball and extracts; subsequent compares are local-only.

Cache invalidation: never (versions are immutable). Cache size: bound to last 5 versions per package; LRU evict.

## Error-handling

- **H6** chain-failure recovery: when `/release` STOPS due to under-bump, the diagnostic is surfaced verbatim with the suggested version.
- **H7** atomic cache writes for `state.json#publicApi.surfaces`.
- **H8** rate-limit backoff if `npm view`/tarball fetch hits 429.

## Notes

- The custom AST walker is intentionally narrow: function/class/interface/type/const/enum/namespace cover ~99% of solo-dev library surfaces. Edge cases (Symbol exports, computed property names, complex conditional types) may produce warnings; the skill surfaces them as `EXTRACT_PARTIAL` issues but proceeds with what it could extract.
- For first releases (no baseline), the skill exits 0 with a "first release" note. The /release flow continues normally.
- This skill does NOT mutate any files. It reads from `dist/` and the npm registry; emits a report.
- Pre-release versions (semver `-suffix`) are compared against the last STABLE release per dist-tags.latest. To compare pre-release vs pre-release, pass `--against <prerelease-version>` explicitly.
- For monorepos, `--path <package-dir>` runs per-package. Aggregate report has one `PublicApiReport` per package.
