---
description: Bootstrap a fresh repo for tag-triggered npm publishing with OIDC + provenance — scaffold release.yml + publishConfig + .nvmrc + consumer wrappers + .claude/settings.json, gate on first manual publish, then chain into /solo-npm:trust. Idempotent. Also supports `--refresh-yml` mode for surgical updates to an existing release.yml (used by /solo-npm:prerelease and /solo-npm:hotfix when they detect drift).
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

Two modes:

**Default mode** — full bootstrap. Four phases. Idempotent — never overwrites existing artifacts. Safe to run multiple times.

- **Phase 1** — detect state, render plan, ONE `AskUserQuestion`,
  generate scaffolds + consumer wrappers + settings.json.
- **Phase 2** — gate on first publish via `npm-trust --doctor`.
- **Phase 3** — chain into `/solo-npm:trust` for OIDC config.
- **Phase 4** — final summary + next-steps.

**Refresh mode** (`--refresh-yml`) — surgical updates to existing `release.yml` without re-running the full plan. Used by `/solo-npm:prerelease` and `/solo-npm:hotfix` Phase A when they detect a stale workflow that lacks the dist-tag detection step. Idempotent — no-op if `release.yml` is already current. See [Refresh-only mode](#refresh-only-mode---refresh-yml) below.

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
| `which gh` (PATH lookup) | binary present + version | Plan summary recommendation |
| `gh auth status` (if `gh` present) | authenticated state | Plan summary auth state |

**Tooling check** — `gh` (GitHub CLI) is **optional but recommended**. solo-npm uses `gh` for:
- GitHub Release auto-creation in `/release`, `/prerelease`, `/hotfix` (CHANGELOG entry as release notes body)
- `/status` portfolio enrichment (open issues, CI run results)
- `gh run watch` for CI tracking

If `which gh` returns nothing → record `gh: not installed` in the plan summary. If `gh` is installed but `gh auth status` fails → record `gh: installed, not authenticated`. Otherwise → record `gh: ready`.

These are surfaced in Phase 1b's plan; the user can install `gh` from [cli.github.com](https://cli.github.com) (the canonical install page covers macOS, Linux, Windows, and other platforms) before proceeding, OR continue without it (skills gracefully skip the gh-dependent steps).

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

### 1d. Pkg-check validation (NEW)

After scaffolds are written but BEFORE committing them, validate the package manifest end-to-end so /init never claims "complete" while leaving the manifest non-publish-ready.

Invoke `/solo-npm:verify --pkg-check-only`. This runs Step 5's three tiers (publint + manual checks + tarball-content audit) without re-running lint/typecheck/test/build (those are dev-time, not publish-time concerns).

Auto-fix loop:

1. If pkg-check returns `PKG_CHECK_OK` → continue to 1e.
2. If `PKG_CHECK_FAIL` (errors found) with auto-fix offers → AskUserQuestion → apply selected fixes → re-run pkg-check.
3. If errors persist after auto-fixes (or auto-fixes weren't applicable) → STOP with: *"Init scaffolded the workflow files but `package.json` has unresolvable issues. Fix manually then re-run `/solo-npm:init` (idempotent — won't re-scaffold)."*
4. If `PKG_CHECK_WARN` (warnings only) → log warnings; continue to 1e.
5. **If secrets detected in Tier 3** → HARD STOP with the verbatim secrets-detection block. Do NOT proceed to commit.

This validation closes the original gap from `docs/npm-coverage.md`: *"/init scaffolds publishConfig but doesn't validate everything"*.

### 1e. Commit + push (gated)

After scaffolds are written and pkg-check is clean (or warnings-only), surface a diff summary and call `AskUserQuestion`:

- header: `"Commit scaffolds"`
- options: `Commit + push (Recommended)` / `Commit only` / `Stage only` / `Skip`

Default commit message: `chore: bootstrap solo-npm release workflow`.

If pkg-check made any auto-fix commits during 1d, the diff summary surfaces them as separate commits already (so the user sees `chore(pkg): set repository.url from git remote` etc. alongside the scaffolding commit).

## Phase 2 — First-publish gate (guided in v0.7.0)

Run the doctor:

```bash
pnpm exec npm-trust --doctor --json --workflow release.yml
```

If `issues[]` does NOT contain `PACKAGE_NOT_PUBLISHED`: skip the gate; proceed to Phase 3 silently.

If `issues[]` contains `PACKAGE_NOT_PUBLISHED`, run the **guided initial-publish flow** (replaces the v0.6.x STOP-and-resume pattern):

### 2a. Auth check + foolproof handoff

Run `npm whoami`. If exit code != 0 (not authenticated), surface foolproof handoff:

> The following packages need a first publish (npm requires the name to exist on the registry before OIDC trust can be configured):
>
> - `<pkg-1>`
> - `<pkg-2>`
>
> First, authenticate with npm:
>
> 1. Type `npm login` and press Enter.
> 2. It prints a URL — open it in your browser.
> 3. Sign in to npmjs.com.
> 4. Approve the login via your authenticator app (web 2FA).
> 5. Return here when the terminal shows your username.

Then `AskUserQuestion`:

```
Header:   "Authenticated"
Question: "Done with `npm login`?"
Options:
  - Yes, continue (Recommended) — re-runs `npm whoami` to verify
  - Abort
```

If `npm whoami` succeeds (already authenticated), skip 2a.

### 2b. Per-package publish gate

For each package in `PACKAGE_NOT_PUBLISHED`, surface an `AskUserQuestion`:

```
Header:   "First publish"
Question: "Run `npm publish --provenance=false --access public` now to claim the package name `<NAME>`?"
Options:
  - Yes — publish now (Recommended) — agent runs the command
  - I'll publish manually — agent stops; user re-invokes /init after
  - Abort — end the skill
```

The question text shows the **resolved package name** so the user can sanity-check before claiming. No `--provenance` (chicken-and-egg: trust isn't configured yet, so OIDC can't sign this initial publish; provenance will be added on subsequent releases via `release.yml`'s `--provenance=true`).

### 2c. Execute on Yes

```bash
cd <package-dir>
npm publish --provenance=false --access public
```

Capture stdout + stderr. On success, surface:

> ✓ Published `<NAME>@<VERSION>` to npm. Continuing to Phase 3 (trust config).

On failure (e.g., name already taken by someone else, registry rejection), surface npm error verbatim and STOP:

> ❌ Publish failed: `<npm error>`
>
> Common causes:
>   - Package name already taken (check with `npm view <NAME>`).
>   - Authentication expired (re-run `npm login`).
>   - Registry rejection (rare; check npm status page).
>
> Fix the underlying issue, then re-invoke `/solo-npm:init`.

For monorepos: iterate per package needing first publish, asking once per package. User can pick "publish now" for some and "manually" for others.

### 2d. Manual path

If user picks "I'll publish manually" for any package, surface the manual instructions verbatim (preserved from v0.6.x):

> Manual publish steps:
>
> 1. `cd <package-dir>`
> 2. `npm publish --provenance=false --access public`
> 3. Re-invoke `/solo-npm:init` when done — it skips Phase 1 (idempotent) and continues from here.

Then end the skill (no Phase 3 — trust config requires the package to exist on the registry first).

### 2e. After all packages published

Continue to Phase 3 trust config without requiring re-invocation. The user-in-the-loop AskUserQuestion gate at 2b is the publish-decision; once they confirm, the rest is automatic.

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

## Refresh-only mode (`--refresh-yml`)

Triggered by:

- Explicit invocation: `/solo-npm:init --refresh-yml`
- Auto-chained from `/solo-npm:prerelease` Phase A or `/solo-npm:hotfix` Phase A when those skills detect that `release.yml` is missing the dist-tag detection step.

**Scope**: only edits `.github/workflows/release.yml`. Skips Phases 1c (everything else), 2, 3, 4. No package.json changes, no wrappers, no trust chain, no first-publish gate. Single-purpose: bring `release.yml` up to the current template's dist-tag awareness.

### Algorithm

1. **Pre-flight**: working tree clean (`git status --porcelain`); if dirty, STOP with the standard "commit or stash first" message. (Pre-flight runs in this mode too because the refresh produces its own commit.)

2. **Read** `.github/workflows/release.yml`. If missing, STOP with: *"No release.yml found. Run `/solo-npm:init` (full bootstrap) first."*

3. **Detect drift**:
   - Grep for `EXPLICIT_TAG=` → dist-tag detection step present?
   - Grep for `--tag ${{ steps.dist.outputs.tag }}` → publish step uses the computed tag?
   - If both present → no-op. Print *"release.yml is up to date — dist-tag detection step already present."* and exit (no commit).

4. **Locate insertion point**: find the `pnpm publish` (or `npm publish`) step in the YAML. The dist-tag detection step must be inserted as the immediately-preceding step in the same job.

5. **Detect divergence**: if any of the following, the workflow is too customized for safe surgery — STOP with the manual-edit message (see step 7):
   - More than one publish step in the workflow
   - The publish step has additional commands chained (e.g., `pnpm publish && ...`)
   - The publish step is inside a custom script block
   - YAML structure is unparseable

6. **Surgical insert**:
   - Insert the `Detect dist-tag` step (verbatim from the template the workflow most closely matches — public OIDC if `id-token: write` is present, private token otherwise) immediately before the publish step.
   - Modify the publish step's `run:` line to append `--tag ${{ steps.dist.outputs.tag }}` (idempotent — only append if not already present).
   - Preserve all other content (formatting, comments, custom env vars, additional steps before/after).

7. **STOP message** (when surgery isn't safe):

   ```
   release.yml looks customized — manual edit needed.

   The dist-tag detection step belongs immediately before your publish step.
   Copy this snippet from the canonical template:

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

   And modify your publish step to append: --tag ${{ steps.dist.outputs.tag }}

   Then re-run /solo-npm:prerelease (or /solo-npm:hotfix) to continue.
   ```

8. **Commit**:

   ```bash
   git add .github/workflows/release.yml
   git commit -m "chore: refresh release.yml for dist-tag detection"
   git push
   ```

   The commit is self-contained and descriptive — separate from any release commit so `git log` clearly shows the workflow evolution.

9. **Return**: print a one-line success summary and exit. The caller skill (/prerelease or /hotfix) re-enters its own Phase A to continue.

### Auto-chain behavior

When `/solo-npm:prerelease` or `/solo-npm:hotfix` invokes this mode via auto-chain, the user has already approved the refresh via an `AskUserQuestion` gate in the caller skill. This mode does NOT re-prompt — it just executes.

When invoked directly (`/solo-npm:init --refresh-yml`), this mode also does NOT prompt — it's a focused operation; the user invoking it explicitly has already decided.

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
