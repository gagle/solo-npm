---
description: Explain the published tarball's composition — which sources contribute size, which deps are bundled vs externalized, what's actually in the dist/. Triggers from prompts like "what's in my tarball?", "bundle composition", "why is my package so big?", "size breakdown by file". Pure read-only; consumes built dist/ + bundler metafile if available.
---

# Bundle analyze

Explain the published tarball's composition. Useful when the unpacked size is unexpectedly large, when wondering which dep contributes most to bundle size, or as a routine pre-publish check.

## Phase −0 — Help mode

If `--help`/`-h`/similar, surface brief summary and STOP.

## When to use

- Pre-publish to understand what's shipping
- Investigating a size regression flagged by `/verify` Tier 4
- Comparing bundle composition between versions
- Diagnosing "why is my package 5 MB?"

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `JSON_OUT` | `--json` | `false` |
| `TOP_N` | `--top <N>` | 20 (top contributors to surface) |
| `PATH` | `--path <dir>` (monorepo) | repo root |
| `MODE` | `--mode <pack\|metafile>` | auto-detect |

## Phase 1 — Pre-flight

1. Verify `<PATH>/dist/` exists. STOP if missing.
2. Verify `<PATH>/package.json` exists.
3. Detect bundler metafile presence: `dist/.metafile.json` (esbuild/tsdown) OR `dist/stats.json` (rollup) OR none. This determines `MODE` if auto.

## Phase 2 — Pack and measure

```bash
TARBALL=$(pnpm pack --pack-destination /tmp 2>&1 | tail -1)
TARBALL_SIZE=$(stat -f '%z' "$TARBALL" 2>/dev/null || stat -c '%s' "$TARBALL")
```

Extract tarball into a temp dir to walk contents:

```bash
TEMP_DIR=$(mktemp -d)
tar -xzf "$TARBALL" -C "$TEMP_DIR"
PACKAGE_DIR="$TEMP_DIR/package"
```

Walk every file in `$PACKAGE_DIR`:

```bash
find "$PACKAGE_DIR" -type f -exec wc -c {} \;
```

Aggregate per directory + per file extension.

## Phase 3 — Bundler metafile analysis (when available)

If `MODE=metafile` and `.metafile.json` exists:

```javascript
const meta = JSON.parse(readFileSync('dist/.metafile.json'));
// meta.outputs[<output-file>].inputs[<input-file>].bytesInOutput
// meta.outputs[<output-file>].imports[] (external deps)
```

Aggregate per source-file contribution to each output. Map back to original `src/` files where possible.

## Phase 4 — Render

### Human

```
bundle-analyze — sample-pkg@1.5.3
Tarball:   /tmp/sample-pkg-1.5.3.tgz (12.3 KB packed)
Unpacked:  51.2 KB (4 files)

Top contributors (unpacked size):
  Source                                    Size       %
  ─────────────────────────────────────────  ─────────  ─────
  dist/index.js                              22.4 KB    43.7%
  dist/index.d.ts                            12.1 KB    23.6%
  dist/cli.js                                10.8 KB    21.1%
  dist/internal-helper.js                    5.9 KB     11.5%

External deps (imported, not bundled):
  - typescript@^6.0   (peer)
  - @scope/runtime@^1.0

Tarball contents:
  package.json
  README.md
  LICENSE
  dist/index.js
  dist/index.d.ts
  dist/cli.js
  dist/internal-helper.js

Compared to last published version (1.5.2): +2.1 KB (+4.3%)
```

If a metafile is available, surface a deeper breakdown:

```
Bundler metafile breakdown:
  dist/index.js (22.4 KB)
    src/index.ts             16.8 KB  75.0%
    src/parse.ts              4.2 KB  18.7%
    src/utils.ts              1.4 KB   6.3%
```

### --json

```ts
interface BundleAnalyzeReport {
  schemaVersion: 1;
  tarball: { path: string; packedSize: number; unpackedSize: number; fileCount: number };
  files: ReadonlyArray<{
    path: string;
    size: number;
    percentage: number;
    extension: string;
  }>;
  metafile: {
    available: boolean;
    outputs: ReadonlyArray<{
      file: string;
      size: number;
      contributors: ReadonlyArray<{ source: string; size: number; percentage: number }>;
    }>;
  } | null;
  externalDeps: ReadonlyArray<string>;
  comparison: {
    lastVersion: string | null;
    deltaPacked: number | null;        // bytes
    deltaUnpacked: number | null;
    deltaPercentage: number | null;
  };
}
```

Exit code:
- Always 0 (read-only; no failure mode beyond "missing dist/" which STOPs in Phase 1)

## Cleanup

```bash
rm -rf "$TEMP_DIR"
```

## Integration

- `/verify` Tier 4 already does pack-audit at a high level (publint output); bundle-analyze is the deeper analysis tier.
- `/release` Phase A optional — surface bundle composition diff vs last release as informational.
- `/doctor` does NOT run bundle-analyze (heavy). Cached summary via `state.json#caches.bundleAnalyze[<pkg>@<version>]` (TTL 1 day).

## Error-handling

- **H7** atomic state.json writes for cache updates
- **H8** rate-limit not relevant (local-only)

## Notes

- Bundle analyze does not run the bundler — it consumes existing build outputs. To force a fresh analyze, run `pnpm run build` first.
- The metafile analysis is the most precise; without it, the skill falls back to file-size aggregation (still useful but less granular about WHICH src/ files contribute to which output).
- For monorepos, run per-package; aggregate output.
- The `comparison` block requires the package to be published already; for first releases, `lastVersion` is null and deltas are absent.
