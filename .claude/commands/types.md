---
description: Run arethetypeswrong (attw) against a packed tarball to catch dual-package issues, ESM/CJS boundary problems, and broken types pointers — the bugs publint misses. Triggers from prompts like "check types", "verify dual-package", "run attw", "are the types wrong", "check ESM/CJS resolution". Use as a /verify Tier 5 gate or before /release. Wraps the attw CLI.
---

# Types

Run [arethetypeswrong](https://arethetypeswrong.github.io) (`attw`) against the packed tarball. Catches the dual-package class of bugs that publint and tsc don't detect: missing CJS resolution paths, ESM-only packages claiming CJS support, types pointers that don't resolve, declaration files at the wrong location.

Becomes `/verify` Tier 5 when run as part of the standard verify chain; can also be invoked standalone before `/release` for a quick types-only sanity check.

## Phase −0 — Help mode (per `/unpublish` canonical)

If the user's prompt contains `--help` / `-h` / `"how does /solo-npm:types work"` / similar, surface a help summary **INSTEAD** of running.

Synthesize from the **Phases** (build → pack → attw → render), the wrapped tool (attw), and 2-3 trigger phrases. Note: read-only; consumes a built dist/. See `/unpublish` Phase −0.

After surfacing, **STOP**.

## When to use

- Pre-release sanity check on dual-package compatibility
- After refactoring `package.json#exports` to ensure no resolution path broke
- After upgrading TypeScript or build tool
- As a `/verify` Tier 5 gate (default-on for v0.19.0+)

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `STRICT` | `--strict` flag or "strict" in prompt | `false` (warn-only by default) |
| `JSON_OUT` | `--json` flag | `false` |
| `PATH` | `--path <dir>` flag (monorepo: target package) | repo root |

## Phase 1 — Pre-flight

1. Verify `dist/` exists at `<PATH>` (run `pnpm run build` first if missing).
2. Verify `package.json` exists at `<PATH>`.
3. Detect monorepo shape via `npm-trust` cached descriptor (or fallback to `pnpm-workspace.yaml` / `package.json#workspaces` parsing). For monorepo, run per-package; aggregate.

Resolve the `attw` binary:

```
1. ./node_modules/.bin/attw                 (devDep)
2. command -v attw                          (global)
3. npx -y @arethetypeswrong/cli@latest      (registry fallback)
```

## Phase 2 — Pack

```bash
TARBALL=$(pnpm pack --pack-destination /tmp 2>&1 | tail -1)
```

If pack fails (publint-style issues with package.json shape, missing dist, etc.), STOP and surface stderr verbatim. Fix root cause and re-run.

## Phase 3 — Run attw

```bash
attw --pack "$TARBALL" --format json > /tmp/attw-result.json
```

Parse the JSON output. attw emits:

```ts
{
  analysis: {
    types: {
      kind: 'NoResolution' | 'NoTypes' | 'OK' | …,
      ...
    },
    entrypoints: {
      ".": {
        resolutions: {
          'node10':       { resolution, …, problems[] },
          'node16-cjs':   { resolution, …, problems[] },
          'node16-esm':   { resolution, …, problems[] },
          'bundler':      { resolution, …, problems[] },
        }
      }
    },
    problems: ProblemKind[]
  }
}
```

## Phase 4 — Classify findings

Map attw `ProblemKind` to severity:

| attw problem | Severity | What it means |
|---|---|---|
| `NoResolution` | fail | `import 'pkg'` won't resolve in this mode |
| `UntypedResolution` | warn | resolves but no types |
| `FalseExportDefault` | warn | exports `default` exists but the file has no default export |
| `FalseCJS` / `FalseESM` | fail | CJS/ESM mismatch — consumer would crash |
| `CJSResolvesToESM` | fail | requiring this from CJS would crash at runtime |
| `NamedExports` | warn | named exports may not resolve correctly via require() |
| `Unexpected*` | warn | unexpected file extension etc. |
| (other) | warn | default to warn |

In `STRICT` mode, all warns become fails.

## Phase 5 — Render

### Human (default)

```
attw — sample-pkg@1.5.2 (tarball: /tmp/sample-pkg-1.5.2.tgz)

  Resolution mode    Status     Notes
  ─────────────────  ─────────  ───────────────────────────────────
  node10             ✓ pass
  node16-cjs         ✓ pass
  node16-esm         ✓ pass
  bundler            ✓ pass

Problems:  0
Summary:   ✓ all 4 resolution modes pass
```

When problems present:

```
  Resolution mode    Status     Notes
  ─────────────────  ─────────  ───────────────────────────────────
  node10             ✓ pass
  node16-cjs         ✗ FAIL     CJSResolvesToESM at "."
  node16-esm         ✓ pass
  bundler            ✓ pass

Problems:  1 (1 fail, 0 warn)
Summary:   ✗ CJS consumers cannot require this package
           → either ship a CJS build, or set "type": "module" + drop "exports.require"

Remediation hints:
  - Verify package.json#exports paths point at actual files in the tarball
  - Ensure "exports" has a "types" condition before "require"/"import"
  - For ESM-only: drop "main"; set "type": "module"
```

### --json

Emit `TypesReport` JSON to stdout:

```ts
interface TypesReport {
  schemaVersion: 1;
  tarball: { name: string; version: string; path: string; size: number };
  resolutions: {
    'node10':     ResolutionResult;
    'node16-cjs': ResolutionResult;
    'node16-esm': ResolutionResult;
    'bundler':    ResolutionResult;
  };
  problems: ReadonlyArray<{
    kind: string;
    entrypoint: string;
    resolution: string;
    severity: 'warn' | 'fail';
    message: string;
  }>;
  summary: { ok: boolean; failCount: number; warnCount: number };
}
```

Exit code:
- 0 if no problems (or only warns and `STRICT=false`)
- 40 (TAG_MISMATCH/VALIDATION_FAILURE range) if any fail OR (`STRICT=true` AND any warn)

## Integration points

- **`/verify` Tier 5** — recommended addition; default-on when `attw` is available
- **`/release` Phase A** — optional gate via `--types-strict` flag on `/release`; STOPs the release if attw reports a fail
- **`/doctor`** — does NOT run attw (Doctor is read-only against caches); but if a recent `/types` run exists in cache, surfaces the `summary` in the `provenance` or `publish-readiness` domain

## Error-handling patterns (H1–H8)

- **H6** — chain-failure recovery: when `/release` invokes `/types` as a gate and attw fails, surface the verbatim diagnostic and STOP /release per H6.
- **H8** — rate-limit backoff: attw reads `package.json` from the tarball; no network calls; no rate-limit concerns.

## Notes

- attw doesn't emit a `--capabilities` descriptor itself (it's a third-party tool). solo-npm wraps it; the wrapping is what adds the protocol-conformant shape.
- Some packages are deliberately ESM-only — `FalseCJS` for ESM-only packages is expected and harmless. Use `--strict` only when you intend to ship dual-package compatibility.
- For monorepos, run per-package; the aggregated report has one `TypesReport` per package.
- This skill cannot fix issues — it diagnoses. Remediation is a manual `package.json#exports` edit per attw's problem hints.
