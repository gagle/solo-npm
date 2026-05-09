---
description: Adopt a publish-related convention or feature on an existing repo. Triggers from prompts like "opt in to prepare-dist", "enable capabilities probe", "adopt prepare-dist transform". Narrowed in v0.19.0 to publish-relevant features only — `prepare-dist` and `capabilities-protocol` (the latter for repos that themselves expose a CLI). Build-time and project-meta opt-ins (commitlint, husky, lint-staged, eslint preset, tsconfig base, dependabot, branch-protection) are out of scope per PART III.
---

# Opt in

Adopt a new publish-related convention on an existing repo. Each feature is implemented as a small, idempotent chain into other skills.

Per PART III's narrowed scope, only publish-relevant features are wired in v0.19.0. Build-time / project-meta features are intentionally absent — they live outside solo-npm's scope.

## Phase −0 — Help mode

If `--help`/`-h`/similar, surface brief summary and STOP.

## Available features (v0.19.0)

| Feature | Description |
|---|---|
| `prepare-dist` | Switch to publish-from-dist via `gagle/prepare-dist@v1` GHA step. Modifies release.yml to insert the prepare-dist step between build and publish. |
| `capabilities-protocol` | If the repo's own package exposes a CLI binary, scaffold a `--capabilities --json` implementation conforming to the (implicit, v0.19.0) cli-capability-protocol. Adds devDep on `cli-capability-protocol` (when that family package eventually ships) OR inlines the shape today. |

(Future features available via PART II's plan, deferred per PART III: `commitlint`, `husky`, `lint-staged`, `dependabot`, `branch-protection`, `solo-eslint-config`, `solo-tsconfig-base`, etc.)

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `FEATURE` | trigger phrase or `--feature <name>` | required |
| `JSON_OUT` | `--json` | `false` |

## Phase 1 — Validate feature

Look up `FEATURE` in the registry. If not found:

```
Unknown feature: <name>

Available features in v0.19.0 (publish-related only):
  - prepare-dist
  - capabilities-protocol

For other conventions (commitlint, husky, eslint config, tsconfig, etc.),
those are deferred per the v0.19.0 narrowed scope (PART III). Manage
them via your project's existing tooling.
```

STOP.

## Phase 2 — Detect prerequisites

Each feature has prerequisites. Check before applying:

### `prepare-dist`

- Repo must build to `dist/` (publish-from-dist convention)
- Repo's `package.json#main`/`exports` must reference paths under `dist/`
- `release.yml` must exist (chain into /init if missing)

### `capabilities-protocol`

- Repo's `package.json` must have a `bin` entry (it's a CLI)
- (When `cli-capability-protocol` ships) — devDep added to package.json

If prerequisites unmet, surface and STOP. Suggest the prereq path.

## Phase 3 — Apply

### `prepare-dist`

```bash
# 1. Verify the dep is available (or suggest install)
if ! grep -q '"prepare-dist"' package.json; then
  pnpm add -D prepare-dist@latest
fi

# 2. Refresh release.yml with the variant template
npx -y npm-trust@latest --emit-workflow --with-prepare-dist > .github/workflows/release.yml

# 3. Update state.json: cache.trust.workflowFileHash = sha256 of new release.yml
NEW_HASH=$(sha256sum .github/workflows/release.yml | cut -d' ' -f1)
jq --arg h "$NEW_HASH" '.trust.workflowFileHash = $h' .solo-npm/state.json > .solo-npm/state.json.tmp
mv .solo-npm/state.json.tmp .solo-npm/state.json

# 4. Optionally update package.json#publishConfig per the dist publish pattern
# (handled by /init scaffold — chain if needed)
```

### `capabilities-protocol`

For now (until the substrate package ships), this scaffolds an inline `--capabilities` implementation:

```bash
# 1. Create or edit src/capabilities.ts with the inline CapabilitiesReport shape
# 2. Wire src/cli.ts's parseArgs to handle --capabilities
# 3. Add a self-test: pnpm dlx cli-capability-protocol-tck (when the substrate ships)
```

The actual implementation of this scaffolding requires solo-npm to know the project's CLI structure — best handled with AskUserQuestion to confirm which entry file to wire.

## Phase 4 — Render report

```
opt-in prepare-dist

  ✓ Verified prerequisites
  ✓ devDep prepare-dist@latest already installed (was: ^1.1)
  ✓ release.yml refreshed with --with-prepare-dist variant
  ✓ state.json#trust.workflowFileHash updated (was: abc..., now: def...)

Next:
  - Verify the build still produces dist/ correctly: pnpm run build
  - Run /solo-npm:smoke-test to confirm the new tarball shape works
  - Commit + push, then test the GHA on the next /release
```

## --json

```ts
interface OptInReport {
  schemaVersion: 1;
  feature: string;
  prerequisites: ReadonlyArray<{ name: string; satisfied: boolean }>;
  applied: boolean;
  changes: ReadonlyArray<{ description: string; filesAffected: string[] }>;
  summary: { ok: boolean };
}
```

Exit codes:
- 0 on success
- 10 (CONFIGURATION_ERROR) if feature unknown or prerequisites unmet
- 60 (PARTIAL_FAILURE) if some chain steps succeeded but others failed

## Error-handling

- **H6** chain-failure recovery for the chained skills (/init, scaffold steps)
- **H7** atomic state.json writes
- **H1** if the feature triggers an OIDC re-binding (rare; only `capabilities-protocol` could relate)

## Notes

- This skill is a registry + dispatcher. Each feature is a small, well-defined chain. Adding a new feature = adding an entry to the registry + a Phase-3 application sub-section in this body.
- Per PART III, the v0.19.0 registry is intentionally TWO features (`prepare-dist`, `capabilities-protocol`). Future minor bumps may add more (e.g., when family packages ship in a later minor, `solo-cli-conventions` opt-in could appear).
- For build-time conventions outside solo-npm's scope (commitlint, husky, eslint, tsconfig presets), use the project's own tooling. solo-npm doesn't impose these.
