---
description: Bootstrap a fresh repo for tag-triggered npm publishing with OIDC + provenance — scaffold release.yml + publishConfig + .nvmrc + consumer wrappers + .claude/settings.json, gate on first manual publish, then chain into /solo-npm:trust. Idempotent.
---

# Init

The umbrella skill for fresh-repo bootstrap. Scaffolds the GitHub side,
gates on first publish, then composes with `/solo-npm:trust` for the
npm side. Idempotent — re-invoke any time, only creates what's missing.

## Who this is for

You have a repo (existing or new) and want to add the opinionated
solo-AI-dev release workflow: tag-triggered CI publishing with one
`AskUserQuestion` approval gate, OIDC + SLSA provenance for public npm,
or token-based auth for private/custom registries. Run this skill once;
then `/release` (your project's wrapper) just works.

## How it works

Four phases. Idempotent — never overwrites existing artifacts. Safe to
run multiple times.

- **Phase 1** — detect state, render plan, ONE `AskUserQuestion`,
  generate scaffolds + consumer wrappers + settings.json.
- **Phase 2** — gate on first publish via `npm-trust --doctor`.
- **Phase 3** — chain into `/solo-npm:trust` for OIDC config.
- **Phase 4** — final summary + next-steps.

## Phase 1 — Scaffold

### 1a. Detect existing repo state (silent reads)

| Source | Read | Used for |
|---|---|---|
| `package.json#name` | name | Wrapper templates, scope detection |
| `package.json#version` | version | First-publish (`0.0.0`) vs. ongoing |
| `package.json#private` | private flag | Whether to publish at all |
| `package.json#publishConfig` | registry, access, provenance | Public vs. custom registry detection |
| `package.json#engines.node` | node engine | Skip if already `>=24` |
| `package.json#devDependencies.npm-trust` | devDep presence | Suggest `pnpm add -D npm-trust` if missing |
| `.nvmrc` | content | Skip if already pinning ≥24 |
| `.npmrc` (project) | `registry=`, `@scope:registry=`, `_authToken=` | Detect scope mappings, auth refs |
| `.github/workflows/*.yml` | filenames + content | Skip release.yml if exists |
| `pnpm-workspace.yaml` / `package.json#workspaces` | presence | Single-package vs. monorepo |
| `nx.json` | presence | Nx monorepo flavor |
| `git remote get-url origin` | URL | Repo slug inference |
| `.claude/skills/release/`, `.claude/skills/verify/` | presence | Wrapper templates: skip if exist |
| `.claude/settings.json` | extraKnownMarketplaces, enabledPlugins | Merge if exists |
| `.solo-npm/state.json` | trust-state cache | Create if missing (empty cache) |

### 1b. Render plan + ONE `AskUserQuestion`

Render a summary block to chat (visible in history):

```
Init plan for <PACKAGE_NAME> @ v<VERSION>:

Detected:
  Repo:       <REPO_SLUG>
  Workspace:  <single-package | pnpm-workspace | nx-monorepo>
  Registry:   <public npm | scoped private (@scope at <URL>) | fully private (<URL>)>
  Workflows:  <list of *.yml files>

Will create (only if missing):
  [✓] package.json#engines.node = ">=24"
  [✓] package.json#publishConfig = {access, provenance: true}
  [✓] .nvmrc
  [✓] .github/workflows/release.yml  (public OIDC | private token)
  [✓] .claude/skills/release/SKILL.md  (thin wrapper)
  [✓] .claude/skills/verify/SKILL.md   (thin wrapper)
  [✓] .claude/settings.json            (marketplace + plugin)
  [✓] .solo-npm/state.json             (trust-state cache, empty)
  [—] .npmrc                            (only for scoped private registry)
  [—] CONTRIBUTING.md                   (optional — solo-dev template)

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
  1. `Proceed` (Recommended) — run scaffolding with the plan as shown
  2. `Customize` — open follow-up questions (registry kind override,
     CONTRIBUTING.md include/skip)
  3. `Abort` — no changes; end the skill

If `Customize` is selected and registry kind couldn't be auto-detected,
ask via `AskUserQuestion`:

- header: `"Registry"`
- options: `Public npm` / `Scoped private` / `Fully private` / `Abort`
- For `Scoped private` or `Fully private`, follow up with a free-text
  prompt for the registry URL.

### 1c. Generate / update artifacts

**Idempotent**: only create what's missing. Never overwrite existing
hand-tuned content silently.

#### `package.json` updates

Read, modify the parsed object, write back. Preserve formatting.

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
  }
}
```

For **private** registries, `provenance: true` is **omitted** —
Sigstore is public-npm-only. Including it would trigger
`REGISTRY_PROVENANCE_CONFLICT` in the doctor.

> **Why no `npm-trust:setup` script.** Earlier versions scaffolded a
> `package.json#scripts.npm-trust:setup` shorthand. We removed it in
> v0.5.3: when the user is in Claude Code, the canonical entry point is
> `/solo-npm:trust` (which has the foolproof handoff for the npm web
> 2FA flow). The npm script was a parallel-but-redundant path that the
> agent could surface as an "alternative", causing decision friction.
> Non-Claude users can still add the script manually.

#### `.nvmrc`

If missing, write `24\n`. Otherwise leave alone.

#### `.github/workflows/release.yml`

If missing, pick template based on the registry kind decision in 1b.

Both templates include a **dist-tag detection step** before publish so pre-releases land on `@next`, hotfixes on legacy majors land on `@v<major>`, and stable releases land on `@latest`. Three-layer rule: explicit `package.json#publishConfig.tag` (highest priority — set by `/solo-npm:hotfix` on legacy maintenance branches) → version-shape (`*-id.n` → `next`) → default `latest`.

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
      - name: Detect dist-tag
        id: dist
        run: |
          VERSION=$(node -p "require('./package.json').version")
          EXPLICIT_TAG=$(node -p "require('./package.json').publishConfig?.tag || ''")
          if [ -n "$EXPLICIT_TAG" ]; then
            echo "tag=$EXPLICIT_TAG" >> "$GITHUB_OUTPUT"
          elif [[ "$VERSION" =~ -[a-z]+\.[0-9]+$ ]]; then
            echo "tag=next" >> "$GITHUB_OUTPUT"
          else
            echo "tag=latest" >> "$GITHUB_OUTPUT"
          fi
      - run: pnpm publish --no-git-checks --tag ${{ steps.dist.outputs.tag }}
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
      - name: Detect dist-tag
        id: dist
        run: |
          VERSION=$(node -p "require('./package.json').version")
          EXPLICIT_TAG=$(node -p "require('./package.json').publishConfig?.tag || ''")
          if [ -n "$EXPLICIT_TAG" ]; then
            echo "tag=$EXPLICIT_TAG" >> "$GITHUB_OUTPUT"
          elif [[ "$VERSION" =~ -[a-z]+\.[0-9]+$ ]]; then
            echo "tag=next" >> "$GITHUB_OUTPUT"
          else
            echo "tag=latest" >> "$GITHUB_OUTPUT"
          fi
      - run: pnpm publish --no-git-checks --tag ${{ steps.dist.outputs.tag }}
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

#### `.npmrc` (project root, optional)

**Only generate** for scoped private registry. Mappings only — never
auth tokens.

```
@scope:registry=<PRIVATE_REGISTRY_URL>
```

If `.npmrc` already exists, leave existing lines untouched and append
the scope mapping if not present.

#### `.claude/skills/release/SKILL.md` — thin wrapper

If missing, scaffold a workspace-shape-aware thin wrapper:

**Single-package template:**

```yaml
---
name: release
description: Release <PACKAGE_NAME> — wraps /solo-npm:release for this repo.
---

# Release (<PACKAGE_NAME>)

Composes /solo-npm:release with this repo's specifics.

## Repo context

- Workspace: single package at repo root
- Repo slug: `<REPO_SLUG>`
- Workflow: `release.yml`
- Verification: `/verify` runs <DETECTED_VERIFY_COMMANDS>

## Workflow

Invoke `/solo-npm:release` for the opinionated three-phase baseline.

## Deviations from the baseline

(none)
```

**Monorepo template (pnpm/Nx):**

```yaml
---
name: release
description: Release <PACKAGE_NAME> packages — wraps /solo-npm:release with monorepo specifics.
---

# Release (<PACKAGE_NAME>)

Composes /solo-npm:release with this repo's specifics.

## Repo context

- Workspace: <pnpm | Nx> monorepo at `<WORKSPACE_GLOB>`
- Versioning: <unified across all packages | per-package> — <strategy details>
- Repo slug: `<REPO_SLUG>`
- Workflow: `release.yml`
- Verification: `/verify` runs <DETECTED_VERIFY_COMMANDS>

## Workflow

Invoke `/solo-npm:release` for the opinionated three-phase baseline.
The repo context above tells you what to expect for workspace shape and
verification commands.

## Deviations from the baseline

(none today)
```

Placeholders to substitute from detected state:
- `<PACKAGE_NAME>` from `package.json#name`
- `<REPO_SLUG>` from `git remote get-url origin`
- `<WORKSPACE_GLOB>` from `pnpm-workspace.yaml` or `package.json#workspaces[0]`
- `<DETECTED_VERIFY_COMMANDS>` synthesized from `package.json#scripts`
  (or `pnpm nx run-many -t lint typecheck build test` for Nx)

If `.claude/skills/release/` already exists, **skip** — don't overwrite
the user's wrapper.

#### `.claude/skills/verify/SKILL.md` — thin wrapper

If missing, scaffold:

```yaml
---
name: verify
description: Verify <PACKAGE_NAME> — wraps /solo-npm:verify with this repo's verification commands.
---

# Verify (<PACKAGE_NAME>)

Composes /solo-npm:verify with this repo's specifics.

## Repo context

<DETECTED_VERIFY_LIST>

## Workflow

Invoke `/solo-npm:verify` for the opinionated baseline. Run the
commands above; surface failures with full output.
```

`<DETECTED_VERIFY_LIST>` is a bulleted list of detected scripts:
- Single-package: `- Lint: pnpm lint` / `- Test: pnpm test` / `- Build: pnpm build` (only those present in scripts)
- Nx monorepo: `- Verification: pnpm nx run-many -t lint typecheck build test`

If `.claude/skills/verify/` already exists, **skip**.

#### `.claude/settings.json` — marketplace + plugin pinning

If missing, create with:

```json
{
  "extraKnownMarketplaces": {
    "gllamas-skills": {
      "source": {
        "source": "github",
        "repo": "gagle/solo-npm"
      }
    }
  },
  "enabledPlugins": {
    "solo-npm@gllamas-skills": true
  }
}
```

If existing, **merge keys** — add `extraKnownMarketplaces.gllamas-skills`
and `enabledPlugins["solo-npm@gllamas-skills"]: true` if not already
present. Preserve any other keys in the file.

#### `.solo-npm/state.json` — trust + audit state cache

If missing, create with an empty cache:

```json
{
  "version": 1,
  "trust": {
    "configured": [],
    "lastFullCheck": null,
    "ttlDays": 7
  },
  "audit": {
    "tier1Count": 0,
    "tier2Count": 0,
    "lastFullScan": null,
    "ttlDays": 1
  }
}
```

This file is the per-repo state cache used by:
- `/solo-npm:release` Phase A.3 — skips per-package trust re-checks via the `trust` section.
- `/solo-npm:release` Phase A.5 — skips known-good audit re-runs via the `audit` section; STOPs the release if cached `tier1Count > 0`.
- `/solo-npm:status` — renders the Trust column from cache without live npm calls.

The first `/release` and `/audit` after init populate their respective sections via cold-path scans. Subsequent invocations (within each section's `ttlDays`) take the hot path — zero npm calls.

**Cache TTLs**: trust = 7 days (trust state changes infrequently). Audit = 1 day (CVE landscape changes faster).

The file is **committed** (not gitignored) — package names are public
on npm anyway, and committing gives cross-machine cache sharing for
solo-dev.

If `.solo-npm/state.json` already exists, leave it alone.

#### `CONTRIBUTING.md` (optional)

If user opted in via `Customize` AND no `CONTRIBUTING.md` exists,
write the AI-only solo-dev template (see plugin docs for the canonical
shape).

### 1d. Commit + push (gated)

After scaffolds are written, surface a diff summary and call
`AskUserQuestion`:

- header: `"Commit scaffolds"`
- options: `Commit + push (Recommended)` / `Commit only` / `Stage only` / `Skip`

Default commit message: `chore: bootstrap solo-npm release workflow`.

## Phase 2 — First-publish gate

Run the doctor:

```bash
pnpm exec npm-trust --doctor --json --workflow release.yml
```

If `issues[]` contains `PACKAGE_NOT_PUBLISHED`:

> **Stop here for first publish.**
>
> The following packages need a manual first publish (npm requires the
> name to exist on the registry before OIDC trust can be configured):
>
> - <pkg-1>
> - <pkg-2>
>
> Steps:
>
> 1. Authenticate: `npm login` (interactive, one-time per machine).
> 2. Manually publish each: `npm publish --provenance=false --access public`.
>    (No tag yet; this is the deliberate "claim the name" step.)
> 3. Re-invoke `/solo-npm:init` once published — it'll skip Phase 1
>    (idempotent) and continue to Phase 3.

Then call `AskUserQuestion`:

- header: `"First publish"`
- question: `"Status?"`
- multiSelect: `false`
- options:
  1. `I just published — continue to Phase 3` — proceed to trust config
  2. `Skip Phase 3 — I'll run /solo-npm:trust later` — end gracefully
  3. `Abort` — stop the skill

If no `PACKAGE_NOT_PUBLISHED` issues, skip the gate; proceed to Phase 3
silently.

## Phase 3 — Configure OIDC trust

Invoke `/solo-npm:trust` to configure OIDC Trusted Publishing for all
packages. The trust skill is itself a guided wizard — it'll handle
authentication, dry-run validation, per-package configuration, and
verification.

If the user already has trust configured for some packages and just
added new ones, `/solo-npm:trust` filters via `--only-new` so it
configures only what's needed.

## Phase 4 — Done

Print a final summary:

```
Init complete.
  Scaffolded:           <list of artifacts created>
  Skipped (existing):   <list of artifacts left alone>
  First publish:        <complete | pending — see Phase 2>
  Trust:                <configured for N packages | skipped | pending publish>

Next: /release   (your project's wrapper around /solo-npm:release)
```

## Idempotency contract

Running `/solo-npm:init` against an already-configured repo MUST be a
no-op (zero diff) for these artifacts:

- `package.json#engines.node` (if `>=24`)
- `package.json#publishConfig` (if any value present)
- `.nvmrc` (if exists, regardless of content)
- `.github/workflows/release.yml` (if exists, regardless of content)
- `.npmrc` (existing lines untouched; only append scope mapping if not present)
- `.claude/skills/release/`, `.claude/skills/verify/` (if exists, skip)
- `.solo-npm/state.json` (if exists, leave alone)
- `.claude/settings.json` (existing keys preserved; only merge in marketplace + plugin)
- `CONTRIBUTING.md` (if exists)

The skill never overwrites hand-tuned content. The user sees what's
skipped in Phase 1b's plan summary.

## What this skill does NOT do

- Does NOT do the first publish — chicken-and-egg requires a one-time
  classic publish from local. Phase 2 documents this and waits.
- Does NOT touch `.git/`, `~/.npmrc`, GitHub repo settings, or branch
  protection.
- Does NOT auto-add a GitHub Actions secret — that's a manual step for
  security (token never touches the agent's context).
- Does NOT migrate consumer scripts or change package name.
- Does NOT install the marketplace plugin — that's done by Claude Code
  when the user trusts the folder + accepts the install prompt
  triggered by the committed `.claude/settings.json`.
