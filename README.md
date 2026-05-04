<h1 align="center">solo-npm</h1>

<p align="center">
  Tag-triggered npm releases with one approval, for AI-driven solo dev.
</p>

## The problem

You're a solo developer — or running a small group of LLM agents —
shipping an npm package. PRs are disabled in your repo
(issue/discussion contribution model only). There's no committee to
review, no second pair of human eyes.

Existing release tooling is built for teams: PR-based workflows,
multi-stage approvals, complex changelog negotiation. In a solo or
agent-driven context, that overhead becomes friction — and friction
makes you skip steps when you're moving fast. Skipped steps make
unsigned, unverified, opaque releases.

You want releases that are:

- **Boring**: tested, typed, provenanced, every time. No exceptions.
- **One-touch**: not a 15-step runbook, not a commit-message lottery.
- **Provable**: SLSA attestations on every tarball, traceable to git.

## The solution

A Claude Code skill that drives the entire release end-to-end:

| Phase | What happens | Human input |
|---|---|---|
| **A. Pre-flight** | `/solo-npm:verify` (lint + test + build) → `npm-trust --doctor` | none (silent if green) |
| **B. Plan** | Detect bump, render summary, **one** `AskUserQuestion` selector | one click: Proceed / Override / Edit changelog / Abort |
| **C. Execute** | Bump → commit → push → tag → watch CI → verify on registry | none |

Type `/solo-npm:release`. Review the plan once. Click `Proceed`. Get
a notification when the tarball is on npm with provenance. That's it.

## Why this works for solo / agent-driven dev

- **Trust replaces review.** Tag-triggered CI runs the same gates the
  agent ran locally; SLSA provenance proves what was published.
- **One structured approval gate.** Not a free-text "yes/no" prompt
  (those are easy to miss). A clearly-labeled selector — unmissable.
- **Composable.** `/solo-npm:verify` is its own skill. Customize it
  (add e2e, coverage thresholds, license audits) without touching
  `/solo-npm:release`.
- **No special cases.** Every release follows the same path. The
  "small fix" and "big feature" both go through Phase A → B → C.

## Install

```
/plugin marketplace add gagle/solo-npm
/plugin install solo-npm@gllamas-skills
```

This installs three skills into Claude Code: `/solo-npm:init`,
`/solo-npm:release`, and `/solo-npm:verify`.

The marketplace also hosts the sibling [`npm-trust`](https://github.com/gagle/npm-trust)
plugin (OIDC trust setup wizard). Install it the same way:

```
/plugin install npm-trust@gllamas-skills
```

This makes `/npm-trust:setup` available without needing to clone the
skill into your repo's `.claude/skills/`.

## Initializing a fresh repo with `/solo-npm:init`

Going from "I have a repo" to "the release skill works out of the
box" used to mean manually scaffolding `release.yml`, `publishConfig`,
`engines.node`, `.nvmrc`, the `npm-trust:setup` script, etc.
`/solo-npm:init` does all that with one approval gate.

```
cd path/to/your/repo
# in Claude Code:
/solo-npm:init
```

The skill detects existing repo state, shows you the diff it'll
apply, asks one `AskUserQuestion`, then scaffolds only what's
missing. Idempotent — safe to run multiple times. Supports both
public npm (OIDC + provenance) and private/custom registries (token
auth via GitHub Actions secret).

After `/solo-npm:init` finishes, the printed checklist covers the
remaining manual steps (devDep install, OIDC trust setup or GitHub
secret config). Then `/solo-npm:release` works.

## Prerequisites

If you used `/solo-npm:init`, prerequisites are taken care of. If
you're configuring manually:

- A tag-triggered `release.yml` workflow that publishes via
  `pnpm publish --no-git-checks` and reads `publishConfig` from
  `package.json`.
- OIDC trust configured for the package (one-time setup via
  [`gagle/npm-trust`](https://github.com/gagle/npm-trust) and the
  bundled `/npm-trust:setup` skill). Skip this for private/custom
  registries — they use token auth instead.
- Conventional commit messages (`feat:`, `fix:`, etc.) since the
  release skill parses them to compute the version bump.

## Configuration

After install, edit `.claude/skills/release/SKILL.md` to fill
in the placeholders for your repo:

| Placeholder | Example | What it controls |
|---|---|---|
| `<PACKAGE_NAME>` | `rfc-bcp47` | npm view target |
| `<REPO_SLUG>` | `gagle/rfc-bcp47` | compare URLs in CHANGELOG |
| `<RELEASE_WORKFLOW>` | `release.yml` | doctor's `--workflow` filter |
| Monorepo block | (delete if single-package) | iterate `packages/*` |
| `<LINT_CMD>`, `<TEST_CMD>`, `<BUILD_CMD>` | `pnpm run lint`, `pnpm test`, `pnpm run build` | Fallbacks if `/solo-npm:verify` isn't installed |

Customize `.claude/skills/verify/SKILL.md` for your stack —
add e2e, coverage gates, typecheck, whatever.

## Pre-release versions

The skill detects `x.y.z-pre.n` shape automatically. Phase B's
`AskUserQuestion` adapts to:

- `Bump pre-release counter` (e.g., `1.3.0-beta.1` → `1.3.0-beta.2`)
- `Promote to stable` (`1.3.0-beta.n` → `1.3.0`)
- `Override` / `Abort`

To start a pre-release line from stable, use `Override version` and
type something like `1.3.0-beta.1` — the skill accepts it verbatim.

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
| Phase C.7 verify on registry | `npm view <pkg> dist.attestations` | Skipped (no provenance to verify) |

### Configuration shape

**Fully private** (all packages go to your registry):

```json
"publishConfig": {
  "access": "public",
  "registry": "https://npm.your-internal-registry.example.com/"
}
```

**Scoped private** (only `@my-org/*` goes private; everything else
resolves from public npm):

```jsonc
// package.json
"publishConfig": {
  "access": "public",
  "registry": "https://npm.your-internal-registry.example.com/"
}

// .npmrc (committed at repo root, scope mappings only — NEVER auth)
@my-org:registry=https://npm.your-internal-registry.example.com/
```

For either shape, **omit `"provenance": true`** from `publishConfig`.
`npm-trust --doctor` flags this as `REGISTRY_PROVENANCE_CONFLICT` if
both are set.

### Auth in CI

`actions/setup-node@v4` with `registry-url` + `always-auth: true`
writes a temporary `.npmrc` at job runtime that pulls auth from
`NODE_AUTH_TOKEN`. The token comes from a GitHub Actions repo secret
(commonly named `NPM_TOKEN`):

```yaml
- uses: actions/setup-node@v4
  with:
    node-version-file: .nvmrc
    cache: pnpm
    registry-url: https://npm.your-internal-registry.example.com/
    always-auth: true
- run: pnpm publish --no-git-checks
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

NEVER commit a literal `_authToken=...` line in `.npmrc`. The token
belongs in env / repo secrets only. `npm-trust --doctor` flags this
as `NPMRC_LITERAL_TOKEN`.

### `/solo-npm:init` handles both

If you start with `/solo-npm:init`, the skill detects whether
`publishConfig.registry` (or `.npmrc`) already points at a custom
registry and picks the right `release.yml` template + omits
`provenance: true`. You don't have to remember any of the above.

## Monorepos

The bundled `release/SKILL.md` ships with a monorepo block.
Single-package repos delete it after install. Monorepos keep it;
Phase C iterates `packages/*` for the version bump and per-package
registry verification.

See [`gagle/ncbijs`](https://github.com/gagle/ncbijs) as a working
example of the matrix-publish pattern.

### Publishing from `dist/` with `gagle/prepare-dist@v1`

Monorepos that publish from `<package>/dist/` (rather than from
`<package>/`) commonly pair with the
[`gagle/prepare-dist@v1`](https://github.com/gagle/prepare-dist)
GitHub Action — it cleans up `<dist>/package.json` (strips `dist/`
prefix from paths, drops dev fields like `scripts` /
`devDependencies` / `files`, copies `README.md` and `LICENSE` into
`dist/`) and verifies that `package.json#version` matches the pushed
tag. The two layers compose cleanly:

| Layer | Owner |
|---|---|
| Bump source `package.json#version` | this skill (Phase C.1) |
| Tag + push tag | this skill (Phase C.5) |
| Translate source `package.json` → `dist/package.json` | `prepare-dist` action in `release.yml` |
| `npm publish` from `<package>/dist/` | `release.yml` publish step |
| Registry verify | this skill (Phase C.7) |

See [`gagle/ncbijs`'s `release.yml`](https://github.com/gagle/ncbijs/blob/main/.github/workflows/release.yml)
for the full pattern.

## Contributing

This project follows an AI-only contribution model — see
[`CONTRIBUTING.md`](./CONTRIBUTING.md). PRs are disabled. Open an issue
or discussion for change requests.
