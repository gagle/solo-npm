---
description: Read-only meta-skill that surfaces the cached toolchain descriptors (npm-trust + prepare-dist + future conformant tools) and lets users force-refresh. Triggers from prompts like "what tools does solo-npm see?", "refresh the toolchain cache", "show capabilities", "list installed solo tools", "which features does my npm-trust expose?". Use to verify the toolchain matches what the skills require, or to force a refresh after upgrading a tool.
---

# Toolchain

Read-only meta-skill that exposes solo-npm's view of the installed toolchain. Each conformant tool (npm-trust, prepare-dist) emits a `--capabilities --json` descriptor; solo-npm caches those descriptors in `.solo-npm/state.json#toolchain` (TTL 1 day) and consumes them via feature-presence probes inside other skills.

This skill is the **lens** onto that cache. It does not modify behavior; it describes what's there.

## Phase −0 — Help mode (per `/unpublish` canonical)

If the user's prompt contains `--help` / `-h` / `"how does /solo-npm:toolchain work"` / similar, surface a help summary **INSTEAD** of running.

Synthesize from the **Operations** (read / refresh / per-tool / per-feature), **Phases** (Phase 1 read cache, Phase 2 optional refresh, Phase 3 render), and 2-3 trigger phrases (e.g., *"refresh capabilities"*, *"what tools are installed"*). Note: this skill is read-only by default; only `refresh` mutates state.json. See `/unpublish` Phase −0 for canonical format.

After surfacing, **STOP**. Re-invocation without help triggers runs normally.

## When to use

- Verify the installed toolchain matches what other skills require (e.g., `/release` requires `npm-trust@latest` exposing `validate-only` + `verify-provenance` features).
- Force-refresh after upgrading `npm-trust` or `prepare-dist` so the cached descriptor matches the new binary.
- Diagnose why a skill stopped with "TOO_OLD" — see exactly which feature was missing.
- Inspect the JSON schemas a tool can emit (used by skills that parse those outputs).

## Operations

| Operation | Trigger | Effect |
|---|---|---|
| `read` (default) | `/solo-npm:toolchain` | Render cached descriptors; refresh only if cache is stale or missing |
| `refresh` | `/solo-npm:toolchain refresh` | Force-refresh all probes regardless of TTL |
| `per-tool` | `/solo-npm:toolchain --tool npm-trust` | Render one tool's descriptor only |
| `per-feature` | `/solo-npm:toolchain --feature validate-only` | Show which tool exposes a given feature |

## Phase 0 — Read prompt context

Extract from the user prompt (per /unpublish Phase 0 / E2 from v0.11.0):

| Slot | Source | Example |
|---|---|---|
| `OPERATION` | trigger phrase | `read` (default), `refresh`, `per-tool`, `per-feature` |
| `TOOL` | `--tool <name>` flag or named tool in prompt | `npm-trust`, `prepare-dist` |
| `FEATURE` | `--feature <name>` flag or named feature in prompt | `doctor`, `validate-only` |

If `OPERATION=refresh`, jump to Phase 2 and skip Phase 1's cache-read; rebuild from scratch.

## Phase 1 — Read cache

Read `.solo-npm/state.json` if it exists. Apply H2 corruption guard: wrap `JSON.parse` in try/catch — on parse fail, surface non-fatal warning *".solo-npm/state.json is malformed; treating as empty cache."* and proceed as if `toolchain.lastProbe = null`.

Compute:

- `cacheAgeHours = now - toolchain.lastProbe` (null if never probed)
- `cacheStale = (cache missing) OR (toolchain.lastProbe is null) OR (cacheAgeHours >= toolchain.ttlDays * 24)`

If `cacheStale` is true and `OPERATION != refresh`, **automatically** advance to Phase 2 (refresh). If fresh and `OPERATION = read`, proceed directly to Phase 3 (render).

## Phase 2 — Probe & write

For each known tool resolver (resolved per `/trust` Pre-flight CLI resolution conventions):

```bash
TOOL_BINARY=$(resolve_tool_binary "<tool-name>")     # source / devDep / global / npx fallback
DESCRIPTOR=$("$TOOL_BINARY" --capabilities --json 2>/dev/null) || DESCRIPTOR=""
```

Known tools (initial set):

- `npm-trust` — resolved via `node ./bin/npm-trust.js` → `./node_modules/.bin/npm-trust` → `command -v npm-trust` → `npx -y npm-trust@latest`
- `prepare-dist` — resolved via `./node_modules/.bin/prepare-dist` → `command -v prepare-dist` → `npx -y prepare-dist@latest`

For each non-empty `DESCRIPTOR`:

1. Validate JSON shape: must contain `schemaVersion`, `name`, `version`. Skip + warn on malformed.
2. Detect conformance level from populated fields: L1 (just required) / L2 (+ features/flags/exitCodes) / L3 (+ jsonSchemas populated).
3. Write to `state.json#toolchain.tools[<name>]` with `version`, `level`, `features`, `flags` (names only), `jsonSchemas`, `exitCodes`.

Atomic write per H7: write to `.solo-npm/state.json.tmp`, then `renameSync()` to prevent corruption on mid-flight kill.

Set `state.json#toolchain.lastProbe = now`.

Save.

## Phase 3 — Render

### Default render (operation=read)

```
Toolchain (last probed 2 hours ago):

  npm-trust       0.11.0  L3  ✓
                          features: doctor, validate-only, verify-provenance,
                                    emit-workflow, with-prepare-dist, list,
                                    configure, json-output
                          flags: 16 documented
                          json schemas: 6 documented
                          exit codes: 9 documented (0, 1, 10, 20, 21, 30, 40, 50, 60)

  prepare-dist    1.1.0   L3  ✓
                          features: transform, copy-metadata, verify-tag,
                                    json-output, plugins
                          flags: 6 documented
                          json schemas: 2 documented
                          exit codes: 6 documented (0, 1, 10, 30, 40, 50)

Skills require:
  /trust:       doctor, validate-only, verify-provenance, list, configure, with-prepare-dist  ✓
  /release:     validate-only, doctor                                                          ✓
  /status:      doctor, verify-provenance                                                      ✓
  /audit:       verify-provenance, doctor                                                      ✓
  /verify:      verify-provenance, transform                                                   ✓
  /doctor:      doctor, validate-only, verify-provenance                                       ✓
  /smoke-test:  transform                                                                      ✓
  /workspace remove: deprecate-on-npm                                                          ⚠ inferred (no capability)
```

When all required features are present, surface ✓; when missing, surface ⚠ with the missing feature names listed inline.

### Per-tool render (operation=per-tool)

Full descriptor JSON for the named tool:

```json
{
  "name": "npm-trust",
  "version": "0.11.0",
  "level": 3,
  ...
}
```

### Per-feature render (operation=per-feature)

Lists which tool exposes the named feature:

```
Feature `validate-only` is exposed by:
  npm-trust 0.11.0 (L3)

Feature `transform` is exposed by:
  prepare-dist 1.1.0 (L3)

Feature `bundle-analyze` is exposed by:
  (no installed tool exposes this feature)
```

## Error-handling patterns (H1–H8 from `/unpublish` reference)

- **H2** — `.solo-npm/state.json` corruption guard (Phase 1 reads it).
- **H7** — Atomic writes (Phase 2 writes back).
- **H8** — Rate-limit backoff if probe involves `npx -y` registry fetch and registry returns 429; retry 1s/2s/4s/8s + jitter.
- **H6** — Chain-failure recovery: this skill is consumed by `/doctor`'s capability-domain probe. If toolchain refresh fails (e.g., binary missing), `/doctor` surfaces the diagnostic instead of crashing.

## Notes

- This skill does NOT install missing tools. If a tool is missing, the descriptor is absent from the cache and skills consuming the cache will surface "missing tool" diagnostics.
- The TTL is intentionally aggressive (1 day) because tool versions don't change often. Override per repo by editing `state.json#toolchain.ttlDays`.
- Capability descriptors are SOURCE OF TRUTH for what features each tool exposes. solo-npm skills probe by feature, not by version, per the v0.18.0 `@latest` convention (see `/unpublish` −1.4d).
- The cached descriptor for solo-npm itself is conceptual (solo-npm is a markdown plugin, not a CLI binary). Skills know the solo-npm version via `.claude/settings.json` and the `cli.version` field of `/doctor`'s output.
