---
name: init-solo-npm
description: >
  Initialize an existing repo with the opinionated solo-AI-dev release
  workflow. Detects current state, asks ONE structured question, then
  scaffolds release.yml + publishConfig + .nvmrc + npm-trust:setup
  script (idempotent — only creates what's missing). Supports both
  public npm (OIDC + provenance) and private/custom registries
  (token auth via GitHub Actions secret). Composes with
  /release-solo-npm and the npm-trust CLI.
---

# Init Solo NPM

## Who this is for

You have an existing repo and want to add the opinionated solo-AI-dev
release workflow: tag-triggered CI publishing with one
`AskUserQuestion` approval gate, OIDC + SLSA provenance for public
npm, or token-based auth for private/custom registries. Run this skill
once; then `/release-solo-npm` works out of the box.

## How it works

Three-phase flow with **one** human approval gate. Idempotent — never
overwrites existing artifacts. Safe to run multiple times.

- **Phase A** detects existing repo state (silent reads).
- **Phase B** renders the plan and asks ONE `AskUserQuestion`.
- **Phase C** generates / updates artifacts. Skip what's already there.
- **Phase D** runs `npm-trust --doctor` to verify the result, then
  prints the next-steps checklist.

## Phase A — Detect existing repo state

Read every signal that affects the plan. No writes.

| Source | Read | Used for |
|---|---|---|
| `package.json#name` | name | Placeholders, scope detection |
| `package.json#version` | version | First-publish (`0.0.0`) vs. ongoing |
| `package.json#private` | private flag | Whether to publish at all |
| `package.json#publishConfig` | registry, access, provenance | Public vs. custom registry detection |
| `package.json#engines.node` | node engine | Skip if already `>=24` |
| `package.json#scripts.npm-trust:setup` | script presence | Skip if already there |
| `package.json#devDependencies.npm-trust` | devDep presence | Suggest `pnpm add -D npm-trust` if missing |
| `.nvmrc` | content | Skip if already pinning ≥24 |
| `.npmrc` (project) | `registry=`, `@scope:registry=`, `_authToken=` | Detect scope mappings, auth refs |
| `.github/workflows/*.yml` | filenames + content | Skip release.yml if exists; ci.yml awareness |
| `pnpm-workspace.yaml` / `package.json#workspaces` | presence | Single-package vs. monorepo |
| `git remote get-url origin` | URL | `<REPO_SLUG>` placeholder inference |
| `.claude/skills/release-solo-npm/` | presence | Whether to suggest re-install |
| `.claude/skills/npm-trust-setup/` | presence | Whether to suggest install |

## Phase B — Plan + ONE `AskUserQuestion`

Render a summary block to chat (visible in history):

```
Init plan for <PACKAGE_NAME> @ v<VERSION>:

Detected:
  Repo:       <REPO_SLUG>
  Workspace:  <single-package | pnpm-workspace | npm-workspace>
  Registry:   <public npm | scoped private (@scope at <URL>) | fully private (<URL>)>
  Workflows:  <list of *.yml files>

Will create (only if missing):
  [✓] package.json#engines.node = ">=24"
  [✓] package.json#publishConfig = {access, provenance: true}
  [✓] package.json#scripts.npm-trust:setup
  [✓] .nvmrc
  [✓] .github/workflows/release.yml  (public OIDC | private token)
  [—] .npmrc                          (only for scoped private registry)
  [—] CONTRIBUTING.md                 (optional — solo-dev template)

Will skip (already present):
  [-] <list of skipped artifacts with reasons>
```

Then call the `AskUserQuestion` tool (load via
`ToolSearch query="select:AskUserQuestion"` if its schema isn't loaded
yet).

- header: `"Init"`
- question: `"Apply the plan above?"`
- multiSelect: `false`
- options:
  1. `Proceed` (Recommended) — run Phase C with the plan as shown
  2. `Customize` — open follow-up questions (registry kind override,
     monorepo block override, CONTRIBUTING.md include/skip)
  3. `Abort` — no changes; end the skill

If `Customize` is selected:

- If registry kind couldn't be auto-detected (no existing
  `publishConfig.registry`, no `.npmrc#registry` or `@scope:registry`),
  ask via `AskUserQuestion`:
  - header: `"Registry"`
  - options: `Public npm` / `Scoped private` / `Fully private` / `Abort`
  - For `Scoped private` or `Fully private`, follow up with a free-text
    prompt for the registry URL.

## Phase C — Generate / update artifacts

**Idempotent**: only create what's missing. Never overwrite existing
hand-tuned content silently. The user sees what's untouched in the
plan.

### `package.json` updates

Read, modify the parsed object, write back. Preserve formatting where
possible (use the same indent the file already uses).

**Fields to add ONLY if missing**:

```jsonc
{
  // engines.node:
  "engines": { "node": ">=24" },

  // PUBLIC path — publishConfig:
  "publishConfig": {
    "access": "public",
    "provenance": true
  },

  // PRIVATE path — publishConfig (NO provenance):
  "publishConfig": {
    "access": "public",
    "registry": "<PRIVATE_REGISTRY_URL>"
  },

  // scripts.npm-trust:setup:
  "scripts": {
    "npm-trust:setup": "npm-trust --auto --repo <REPO_SLUG> --workflow release.yml"
  }
}
```

For **private** registries, `provenance: true` is **omitted** —
Sigstore is public-npm-only. Including it would trigger
`REGISTRY_PROVENANCE_CONFLICT` in the doctor.

### `.nvmrc`

If missing, write `24\n`. Otherwise leave alone.

### `.github/workflows/release.yml`

If missing, pick template based on the registry kind decision in Phase B.

**Public template** (OIDC + provenance):

```yaml
name: Release

on:
  push:
    tags: ['v*']

permissions:
  contents: read
  id-token: write   # OIDC for npm trusted publishing

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: pnpm
          registry-url: https://registry.npmjs.org
      - run: pnpm install --frozen-lockfile
      - run: pnpm run lint
      - run: pnpm test
      - run: pnpm run build
      - run: pnpm publish --no-git-checks
```

**Private template** (token auth, no provenance):

```yaml
name: Release

on:
  push:
    tags: ['v*']

# No id-token: write — private registries don't use OIDC
permissions:
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: pnpm
          registry-url: <PRIVATE_REGISTRY_URL>
          always-auth: true
      - run: pnpm install --frozen-lockfile
      - run: pnpm run lint
      - run: pnpm test
      - run: pnpm run build
      - run: pnpm publish --no-git-checks
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

`actions/setup-node@v4` with `registry-url` + `always-auth: true`
writes a temporary `.npmrc` at job runtime that pulls auth from
`NODE_AUTH_TOKEN`. No committed `.npmrc` needed for auth.

### `.npmrc` (project root, optional)

**Only generate** for scoped private registry. Mappings only — never
auth tokens (those live in `NODE_AUTH_TOKEN` env / GitHub secrets).

```
@scope:registry=<PRIVATE_REGISTRY_URL>
```

If `.npmrc` already exists, leave existing lines untouched and append
the scope mapping if not present.

### `release-solo-npm` skill placeholders

If `.claude/skills/release-solo-npm/SKILL.md` exists with
placeholders, fill them with detected values:

- `<PACKAGE_NAME>` → from `package.json#name`
- `<REPO_SLUG>` → from `git remote get-url origin`
- `<RELEASE_WORKFLOW>` → `release.yml`

If the skill is not yet installed (no
`.claude/skills/release-solo-npm/`), Phase D's next-steps prints the
`/plugin install` command.

### Strip blocks per detected layout

After filling placeholders, strip blocks that don't apply:

- **Single-package**: remove the monorepo block from
  `release-solo-npm/SKILL.md` (clearly delimited in the template).
- **No separate `ci.yml`**: remove the Phase C.4.5 ci.yml gating block.

For **monorepos**: keep the monorepo block. For repos that have a
separate `ci.yml` workflow with verification gates not in
`release.yml`: keep Phase C.4.5.

### `CONTRIBUTING.md` (optional)

If user opted in via `Customize` AND no `CONTRIBUTING.md` exists,
write the AI-only solo-dev template (see
`docs/contributing-template.md` in this plugin for the canonical
shape).

## Phase D — Run doctor + print next steps

### Run the doctor

```bash
pnpm exec npm-trust --doctor --json --workflow release.yml
```

Surface any issues. The new `npm-trust@0.7.0` checks
(`WORKFLOW_AUTH_MISMATCH`, `NPMRC_REGISTRY_DIVERGES`,
`NPMRC_LITERAL_TOKEN`, `WORKFLOW_MISSING_AUTH_SECRET`) catch most
misconfigs.

If doctor reports `WORKFLOW_MISSING_AUTH_SECRET` (private path),
that's expected — surface the secret name to the user as a reminder.

### Next-steps checklist

**Public path**:

```
Init done. Next steps:

1. pnpm add -D npm-trust   (if not already a devDep)
2. pnpm exec npm-trust --init-skill npm-trust-setup
3. If <PACKAGE_NAME> isn't on npm yet (first publish):
   - npm publish --provenance=false --access public   (one-time, web 2FA)
   - Then run /npm-trust-setup to enable OIDC trust
4. From here on: /release-solo-npm
```

**Private path**:

```
Init done. Next steps:

1. pnpm add -D npm-trust   (optional — for the doctor)
2. In GitHub Settings → Secrets, add `NPM_TOKEN` with your registry
   auth token. The release workflow reads it as NODE_AUTH_TOKEN.
3. If <PACKAGE_NAME> isn't on the registry yet:
   - Configure registry auth locally (~/.npmrc or env)
   - npm publish   (first publish from local; provenance N/A)
4. From here on: /release-solo-npm
```

## Phase E (optional) — run common next steps

After Phase D's checklist, optionally call `AskUserQuestion`:

- header: `"Continue"`
- options:
  1. `Run pnpm add -D npm-trust` — execute the install now
  2. `Install npm-trust-setup skill` — run `pnpm exec npm-trust
     --init-skill npm-trust-setup`
  3. `Both` — run both
  4. `Just print, I'll run them` — finish silently

## Idempotency contract

Running `/init-solo-npm` against an already-configured repo MUST be a
no-op (zero diff) for these artifacts:

- `package.json#engines.node` (if `>=24`)
- `package.json#publishConfig` (if any value present)
- `package.json#scripts.npm-trust:setup` (if any script with that key)
- `.nvmrc` (if exists, regardless of content)
- `.github/workflows/release.yml` (if exists, regardless of content)
- `.npmrc` (existing lines untouched; only append scope mapping if not present)
- `CONTRIBUTING.md` (if exists)

The skill never overwrites hand-tuned content. The user sees what's
skipped in Phase B's plan summary.

## What this skill does NOT do

- Does NOT enable OIDC trust — that's `/npm-trust-setup`'s job.
- Does NOT do the first publish — chicken-and-egg requires a one-time
  classic publish from local. Phase D's checklist documents this.
- Does NOT touch `.git/`, `~/.npmrc`, GitHub repo settings, or
  branch protection.
- Does NOT auto-add a GitHub Actions secret — that's a manual step
  for security (token never touches the agent's context).
- Does NOT migrate consumer scripts or change package name — pure
  scaffold of missing scaffolding.
