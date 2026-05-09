---
description: Pre-publish smoke test — pack the tarball, install into a temp fixture project, run an import-and-invoke smoke test against every public export. Catches "the published tarball doesn't actually work" — the bug class no other check catches. Triggers from prompts like "smoke test before publish", "test the tarball", "does the package actually work", "pre-publish sanity check". Use as /verify Tier 7 gate.
---

# Smoke test

Pack the tarball as if publishing now. Install it into a fresh temp fixture project. Import every public export. Invoke a basic operation per export. Surface success/failure.

This catches a class of bugs that survives lint, tests, build, publint, attw, and exports-check: bugs where the published tarball's dist/ paths, exports map, and runtime behavior look right in isolation but break the moment a consumer tries to `import` from the installed package. The fixture-install path exercises the same resolution logic npm/pnpm consumers use — there is no closer substitute for "what happens to a real consumer."

## Phase −0 — Help mode

If the prompt contains `--help`/`-h`/similar, surface a brief summary and STOP. Note: creates a temp fixture; cleans up by default.

## When to use

- Before `/release` (recommended hard gate as `/verify` Tier 7)
- After significant package.json#exports changes
- After build tool upgrade
- When attw flags issues that publint missed (smoke test is the runtime confirmation)

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `KEEP_FIXTURE` | `--keep-fixture` flag | `false` (cleanup by default) |
| `JSON_OUT` | `--json` flag | `false` |
| `PATH` | `--path <dir>` (monorepo) | repo root |
| `DUAL_PACKAGE` | `--dual-package` flag | `false` (ESM-only by default) |

## Phase 1 — Pre-flight

1. Verify `<PATH>/dist/` exists (build first if missing).
2. Verify `<PATH>/package.json` exists.
3. Read public exports via `/solo-npm:exports-check` extraction logic OR fall back to scanning `package.json#exports` for non-`./internal/*` entries.
4. Resolve the tarball pack tool (pnpm pack preferred; npm pack fallback).

## Phase 2 — Pack

```bash
TARBALL=$(pnpm pack --pack-destination /tmp 2>&1 | tail -1)
TARBALL_BASENAME=$(basename "$TARBALL")
```

If pack fails, STOP and surface stderr verbatim.

## Phase 3 — Create fixture

```bash
FIXTURE_DIR=$(mktemp -d -t solo-smoke-XXXXXX)
cd "$FIXTURE_DIR"
pnpm init -y > /dev/null
echo '{"name":"smoke-fixture","version":"0.0.0","type":"module","private":true}' > package.json
```

The fixture's `package.json` mirrors the consumer convention: `type: "module"` to test ESM resolution; `private: true` to prevent accidental publish.

For `--dual-package` mode, also create a sibling fixture with `type: "commonjs"` to test the CJS resolution path.

## Phase 4 — Install tarball

```bash
pnpm add "file:$TARBALL" --silent
```

If install fails, STOP. Common causes: peerDep mismatches, engine-mismatch, integrity hash issues. Surface stderr verbatim.

## Phase 5 — Generate smoke test script

For each public export discovered in Phase 1, generate a single `smoke.mjs` (or `smoke.cjs` for the dual-package path):

```javascript
// smoke.mjs (ESM path)
import { existsSync } from 'node:fs';

const results = [];

try {
  const mod = await import('<package-name>');
  results.push({ name: '<package-name>', kind: 'module', importSucceeded: true });

  // For each named export from the API surface:
  for (const name of <list-of-names>) {
    if (!(name in mod)) {
      results.push({ name, importSucceeded: false, error: 'not exported' });
      continue;
    }
    const value = mod[name];
    const kind = typeof value;

    let invocationSucceeded = null;  // null = not attempted
    let error = null;

    // Probe the value: if function, try calling with no args (catch errors as expected for required-args functions)
    if (kind === 'function') {
      try {
        // for classes (kind: 'function' that's a class): try `new` carefully
        // for plain functions: just invoke
        // wrap in try; many functions need args, but the import itself succeeded which is the primary signal
        ...
      } catch (e) {
        invocationSucceeded = false;
        error = String(e?.message ?? e);
      }
    }

    results.push({
      name,
      kind,
      importSucceeded: true,
      invocationSucceeded,
      error,
    });
  }
} catch (e) {
  results.push({ name: '<package-name>', kind: 'module-import', importSucceeded: false, error: String(e?.message ?? e) });
}

console.log(JSON.stringify(results, null, 2));
```

For each subpath export (e.g., `./vite`, `./node`), generate a separate import:

```javascript
const submod = await import('<package-name>/vite');
// ... probe submod's exports ...
```

## Phase 6 — Run

```bash
node smoke.mjs > smoke-results.json 2> smoke-errors.txt
EXIT_CODE=$?
```

If `EXIT_CODE !== 0`, the import itself crashed (catastrophic failure). Surface stderr verbatim.

For `--dual-package` mode, run the CJS script as well:

```bash
node smoke.cjs > smoke-cjs-results.json 2>> smoke-errors.txt
```

## Phase 7 — Cleanup

```bash
if [ "$KEEP_FIXTURE" = "false" ]; then
  rm -rf "$FIXTURE_DIR"
fi
```

If `--keep-fixture`, surface the path so the user can debug:

```
Fixture preserved at: /tmp/solo-smoke-XXXX
   cd /tmp/solo-smoke-XXXX && node smoke.mjs
```

## Phase 8 — Render

### Human (default)

```
smoke-test — sample-pkg@1.5.2 (tarball: /tmp/sample-pkg-1.5.2.tgz)
Fixture: /tmp/solo-smoke-XXXX (cleaned up)

ESM resolution:
  ✓ import 'sample-pkg'                        → module loaded
    ✓ parse           function          (importable)
    ✓ ParseError      class             (importable, instantiable)
    ✓ DEFAULT_OPTIONS const             (importable)

  ✓ import 'sample-pkg/vite'                   → module loaded
    ✓ vitePlugin      function          (importable)

Summary: 4 passed, 0 failed.
```

When failures present:

```
ESM resolution:
  ✓ import 'sample-pkg'                        → module loaded
    ✗ parseLegacy     not exported     (you removed it but exports map still references it?)
    ✓ parse           function          (importable)

  ✗ import 'sample-pkg/cli'                    → ERR_MODULE_NOT_FOUND
    Cannot find module 'sample-pkg/cli'
    → check package.json#exports./cli; expected ./dist/cli.js

Summary: 1 passed, 2 failed.
```

### --json

```ts
interface SmokeTestReport {
  schemaVersion: 1;
  tarball: { path: string; size: number; name: string; version: string };
  fixture: { path: string; cleanedUp: boolean };
  esm: SmokePathResult;
  cjs?: SmokePathResult;        // only present in --dual-package mode
  summary: {
    passCount: number;
    failCount: number;
    ok: boolean;
  };
}

interface SmokePathResult {
  imports: ReadonlyArray<{
    subpath: string;            // 'sample-pkg', 'sample-pkg/vite', etc.
    success: boolean;
    error?: string;
    exports: ReadonlyArray<{
      name: string;
      kind: 'function' | 'class' | 'object' | 'string' | 'number' | 'boolean' | 'symbol' | 'undefined';
      importSucceeded: boolean;
      invocationSucceeded?: boolean | null;
      error?: string;
    }>;
  }>;
}
```

Exit codes:
- 0 if `summary.ok === true`
- 40 (VALIDATION_FAILURE range) if any failure
- 30 (MISSING_INPUTS range) if pack/fixture-create/install failed

## Integration with /verify and /release

- `/verify` Tier 7 — recommended addition; default-on for v0.19.0+
- `/release` Phase A — RECOMMENDED hard gate. STOP if smoke fails.

## Error-handling

- **H6** chain-failure recovery: when /release invokes /smoke-test as a gate and it fails, surface verbatim and STOP.
- **H4** registry propagation: not relevant here (no registry interaction; uses local tarball).
- **H7** fixture cleanup is not technically atomic but is best-effort; partial fixtures are harmless.

## Notes

- The smoke-test invocation phase tries to call functions with no args. Many functions need arguments; the failure of *invocation* is acceptable as long as the *import* succeeded (the primary signal). The skill differentiates these cleanly.
- For `--dual-package`, verifies both `import` and `require()` paths. Solo-dev TS libraries that ship dual-package can use this to confirm both paths work after build.
- Fixture creation requires write access to `/tmp` (or platform equivalent). On constrained CI, set `TMPDIR` to a writable path.
- This skill does not validate runtime behavior beyond "the function can be invoked." Deep correctness testing is the project's own test suite (run via `/verify` Tier 2). Smoke-test is the publish-time integration gate, not a unit-test runner.
- For monorepos, run per-package; aggregate output.
