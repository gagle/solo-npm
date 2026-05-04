<h1 align="center">solo-npm</h1>

<p align="center">
  The full npm publishing lifecycle for AI-driven solo dev.<br>
  Seven skills covering bootstrap, release, and portfolio operations.
</p>

## The problem

You're a solo developer — or running a small group of LLM agents —
shipping npm packages. PRs are disabled in your repos
(issue/discussion contribution model only). There's no committee to
review, no second pair of human eyes.

Existing release tooling is built for teams: PR-based workflows,
multi-stage approvals, complex changelog negotiation. In a solo or
agent-driven context that overhead becomes friction — and friction
makes you skip steps when you're moving fast. Skipped steps make
unsigned, unverified, opaque releases.

You want releases that are:

- **Boring**: tested, typed, provenanced, every time. No exceptions.
- **One-touch**: not a 15-step runbook, not a commit-message lottery.
- **Provable**: SLSA attestations on every tarball, traceable to git.

But also: you want to *operate* a portfolio of packages without
context-switching across five browser tabs every morning. You want to
catch CVEs before they bite. You want to upgrade deps without it being
a chore.

## The solution

A Claude Code marketplace plugin with **seven skills**, organized by
lifecycle phase:

| Phase | Skills |
|---|---|
| **Bootstrap** (one-time per repo) | `/solo-npm:init`, `/solo-npm:trust` |
| **Per-release** | `/release` (your wrapper) → `/solo-npm:release`, `/verify` (your wrapper) → `/solo-npm:verify` |
| **Portfolio operations** | `/solo-npm:status`, `/solo-npm:audit`, `/solo-npm:deps` |

Each skill has one responsibility:

- `init` — bootstrap a fresh repo (scaffold + first-publish gate + chain into `trust`).
- `trust` — configure npm OIDC Trusted Publishing across the repo's packages.
- `release` — three-phase tag-triggered release with one approval gate.
- `verify` — lint + typecheck + test + build.
- `status` — one-page portfolio dashboard (read-only).
- `audit` — security audit with risk triage (read-only).
- `deps` — dep upgrade orchestrator with `/verify` gates.

Type `/release` (your wrapper) for daily releases. Type
`/solo-npm:status` to see your portfolio at a glance.
`/solo-npm:audit` and `/solo-npm:deps` round out the maintenance
loop. `/solo-npm:init` and `/solo-npm:trust` are one-off bootstrap.

## Composition pattern

The marketplace skills are **opinionated baselines**. Consumer repos
keep **thin wrappers** in `.claude/skills/<name>/SKILL.md` that invoke
the baseline + add repo-specific narrative. The wrapper pattern
preserves per-repo customization (Nx monorepo specifics, prepare-dist
usage, custom verification commands) without forking the whole skill.

| Skill | Has consumer wrapper? | Why |
|---|---|---|
| `release` | YES (daily use) | Per-repo narrative: workspace shape, prepare-dist usage, custom verification |
| `verify` | YES (daily use) | Per-repo verification commands |
| `init`, `trust`, `status`, `audit`, `deps` | NO | One-off or auto-detected; no per-repo customization |

`/solo-npm:init` scaffolds the wrappers automatically when you bootstrap
a fresh repo.

### What a wrapper looks like

`.claude/skills/release/SKILL.md` in a consumer repo:

```yaml
---
name: release
description: Release ncbijs packages — wraps /solo-npm:release with monorepo specifics.
---

# Release (ncbijs)

Composes /solo-npm:release with this repo's specifics.

## Repo context

- Workspace: pnpm + Nx monorepo at `packages/*`
- Versioning: unified across all packages
- Publish: per-package matrix in `release.yml`; uses `gagle/prepare-dist@v1`
- Repo slug: `gagle/ncbijs`
- Workflow: `release.yml`
- Verification: `/verify` runs `pnpm nx run-many -t lint typecheck build test`

## Workflow

Invoke `/solo-npm:release` for the opinionated three-phase baseline.

## Deviations from the baseline

(none today)
```

When you type `/release`, Claude Code loads the wrapper, the wrapper
instructs the agent to invoke `/solo-npm:release`, and both bodies
sit in context. Claude reasons over the merged guidance — your
overrides + the opinionated baseline.

## Install

### Bootstrap a fresh repo

Paste this into your repo's `.claude/settings.json` (create the file
if it doesn't exist):

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

Then open the repo in Claude Code. On first folder trust, you'll see
prompts:

1. *Install marketplace `gllamas-skills`?* → Yes
2. *Install plugin `solo-npm@gllamas-skills`?* → Yes

After accepting, all seven `/solo-npm:*` invocations are available.

Then run `/solo-npm:init` once. The skill scaffolds:

- `.github/workflows/release.yml` (public OIDC or private token, depending on your registry)
- `package.json` updates (`engines.node`, `publishConfig`, `npm-trust:setup` script)
- `.nvmrc`
- `.claude/skills/release/SKILL.md` thin wrapper (workspace-shape-aware)
- `.claude/skills/verify/SKILL.md` thin wrapper

After Phase 1 it gates on first manual publish (npm requires the name
to exist on the registry before OIDC trust can be configured), then
chains into `/solo-npm:trust` for the npm-side OIDC config.

### Manual install (one-off testing)

If you want to try the plugin without committing the settings.json
block:

```
/plugin marketplace add gagle/solo-npm
/plugin install solo-npm@gllamas-skills
```

## The seven skills

### Bootstrap

#### `/solo-npm:init`

Bootstrap a fresh repo. Idempotent — safe to re-invoke. Phase 1
scaffolds files, Phase 2 gates on first publish, Phase 3 chains into
`/solo-npm:trust`, Phase 4 done.

Supports public npm (OIDC + provenance) and private/custom registries
(token auth via GitHub Actions secret).

#### `/solo-npm:trust`

Interactive wizard for configuring npm OIDC Trusted Publishing. The
skill orchestrates the [`npm-trust`](https://github.com/gagle/npm-trust)
CLI: detects workspace, walks through any required manual steps
(`npm login`, web 2FA), runs `npm trust github` per package, verifies.

Use on first OIDC setup or when adding new packages.

### Per-release

#### `/release` (wrapper) → `/solo-npm:release`

Three phases. Halts on the first failure.

| Phase | What happens | Human input |
|---|---|---|
| **A. Pre-flight** | `/verify` (lint + test + build) → `npm-trust --doctor` | none (silent if green) |
| **B. Plan** | Detect bump, render summary, **one** `AskUserQuestion` selector | one click: Proceed / Override / Edit changelog / Abort |
| **C. Execute** | Bump → commit → push → tag → watch CI → verify on registry | none |

Type `/release` (or `/solo-npm:release` for the bare baseline). Review
the plan once. Click `Proceed`. Get a notification when the tarball
is on npm with provenance.

#### `/verify` (wrapper) → `/solo-npm:verify`

Lint + typecheck + test + build, halting on first failure. Auto-detects
the package manager and verification scripts; consumer wrapper specifies
the exact commands.

Composes with `/release` (Phase A.2 + C.4) and `/solo-npm:deps` (after
each upgrade batch).

### Portfolio operations

#### `/solo-npm:status`

One-page dashboard. Read-only snapshot of all your published packages:
latest version, last publish date, weekly downloads, OIDC trust state,
recent commits, open issues, CI health. Daily/weekly cadence.

Replaces "open 5 browser tabs to check on my packages" with one
command.

#### `/solo-npm:audit`

Security audit with risk triage. Runs `pnpm audit` and classifies
advisories into 4 tiers (Fix today / Plan upgrade / Lower priority /
Note). Surfaces only the actionable subset. Read-only.

When Tier 1 or 2 has entries, gates with `AskUserQuestion` to chain
into `/solo-npm:deps` for the fix.

#### `/solo-npm:deps`

Curated dep upgrade orchestrator with `/verify` gates. Detects
outdated + vulnerable deps, classifies into tiers (trivial/safe/
major/CVE-driven), batches upgrades in dependency-graph order, runs
`/verify` after each batch, rolls back on failure with
`AskUserQuestion` gate. One commit per batch.

Major upgrades NEVER auto-applied — surfaces upstream release notes
URL for human review. AI-orchestration ≠ AI-decision for breaking
changes.

## Pre-release versions

The release skill detects `x.y.z-pre.n` shape automatically. Phase B's
`AskUserQuestion` adapts:

- `Bump pre-release counter` (e.g., `1.3.0-beta.1` → `1.3.0-beta.2`)
- `Promote to stable` (`1.3.0-beta.n` → `1.3.0`)
- `Override` / `Abort`

To start a pre-release line from stable, use `Override version` and
type something like `1.3.0-beta.1` — accepted verbatim.

## Custom / private registries

The release skill works against public npm (default) and any custom /
private registry that respects npm token auth (Verdaccio, JFrog
Artifactory, GitHub Packages, GitLab Package Registry, etc.).

### What changes for custom registries

| Concern | Public npm | Custom / private |
|---|---|---|
| Auth in CI | OIDC (no secrets needed) | `NODE_AUTH_TOKEN` env from GitHub Actions secret |
| `release.yml` permissions | `id-token: write` | (no `id-token`) |
| `package.json#publishConfig` | `{ access, provenance: true }` | `{ access, registry: "<URL>" }` (no provenance) |
| SLSA provenance attestation | ✓ | ✗ (Sigstore is npmjs.com-only) |
| Phase C.7 verify on registry | `npm view <pkg> dist.attestations` | skipped |
| `/solo-npm:trust` | configures OIDC | not applicable (token auth) |

`/solo-npm:init` detects whether `publishConfig.registry` (or `.npmrc`)
already points at a custom registry and picks the right `release.yml`
template + omits `provenance: true`.

NEVER commit a literal `_authToken=...` line in `.npmrc`. The token
belongs in env / repo secrets only. `npm-trust --doctor` flags this
as `NPMRC_LITERAL_TOKEN`.

## Monorepos

The release skill auto-detects pnpm + Nx monorepos and iterates
`packages/*` for the version bump + per-package registry verification.
The init skill scaffolds a monorepo-flavored release wrapper template.

See [`gagle/ncbijs`](https://github.com/gagle/ncbijs) as a working
example of the matrix-publish pattern.

### Publishing from `dist/` with `gagle/prepare-dist@v1`

Monorepos that publish from `<package>/dist/` (rather than from
`<package>/`) commonly pair with the
[`gagle/prepare-dist@v1`](https://github.com/gagle/prepare-dist)
GitHub Action — it cleans up `<dist>/package.json` (strips `dist/`
prefix from paths, drops dev fields, copies README + LICENSE) and
verifies that `package.json#version` matches the pushed tag.

| Layer | Owner |
|---|---|
| Bump source `package.json#version` | release skill (Phase C.1) |
| Tag + push tag | release skill (Phase C.5) |
| Translate source → `dist/package.json` | `prepare-dist` action in `release.yml` |
| `npm publish` from `<package>/dist/` | `release.yml` publish step |
| Registry verify | release skill (Phase C.7) |

The consumer's release wrapper notes "uses `gagle/prepare-dist@v1`"
so the agent expects the dist-translation phase in CI.

## Contributing

This project follows an AI-only contribution model — see
[`CONTRIBUTING.md`](./CONTRIBUTING.md). PRs are disabled. Open an issue
or discussion for change requests.

### Dogfooding

To test changes locally without going through the marketplace install:

```bash
cd ~/projects/solo-npm
claude --plugin-dir .
```

Inside that session, all `/solo-npm:*` invocations resolve to your
local source. Each new skill or change in `skills/<name>/SKILL.md`
takes effect after `/reload-plugins`.

## See also

- [`gagle/npm-trust`](https://github.com/gagle/npm-trust) — pure CLI for
  configuring npm OIDC Trusted Publishing. The `/solo-npm:trust` skill
  orchestrates this CLI.
