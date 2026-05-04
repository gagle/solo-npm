# Installing solo-npm

Step-by-step install + bootstrap guide.

## 1. Pin the marketplace and plugin in your repo

In your repo, create or merge `.claude/settings.json`:

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

Commit this file. Anyone who opens the repo on a fresh machine gets
prompted to install the marketplace + plugin on first folder trust.

## 2. Open the repo in Claude Code

On first folder trust, you'll see two prompts:

1. *Install marketplace `gllamas-skills`?* → **Yes**
2. *Install plugin `solo-npm@gllamas-skills`?* → **Yes**

After accepting, all seven `/solo-npm:*` invocations resolve.

If you skip the prompts, you can install manually any time:

```
/plugin marketplace add gagle/solo-npm
/plugin install solo-npm@gllamas-skills
```

## 3. Bootstrap a fresh repo with `/solo-npm:init`

```
/solo-npm:init
```

Phase 1 scaffolds:

- `.github/workflows/release.yml` (public OIDC or private token, depending on your registry config)
- `package.json` updates (`engines.node`, `publishConfig`, `npm-trust:setup` script)
- `.nvmrc`
- `.claude/skills/release/SKILL.md` (thin wrapper, workspace-shape-aware)
- `.claude/skills/verify/SKILL.md` (thin wrapper)

Idempotent — only creates what's missing. Safe to re-invoke.

Phase 2 gates on first manual publish if your package isn't on npm yet
(npm requires the name to exist before OIDC trust can be configured —
this is unavoidable).

Phase 3 chains into `/solo-npm:trust` to configure OIDC Trusted
Publishing.

Phase 4 prints a summary.

## 4. (Manual path) If you want to skip `/solo-npm:init`

If you already have `release.yml`, `publishConfig`, etc., you can skip
init and just configure OIDC trust:

```
/solo-npm:trust
```

The trust skill handles authentication, dry-run validation, per-package
configuration, and verification. Uses the
[`npm-trust`](https://github.com/gagle/npm-trust) CLI under the hood —
install it as a devDep for fast invocation:

```bash
pnpm add -D npm-trust
```

(Optional — the skill falls back to `npx -y npm-trust@latest` if the
CLI isn't a devDep.)

## 5. Customize the consumer wrappers

`/solo-npm:init` scaffolds workspace-shape-aware wrapper templates.
Edit `.claude/skills/release/SKILL.md` and `.claude/skills/verify/SKILL.md`
to add repo-specific narrative:

- For monorepos: note the package layout, versioning strategy
  (unified vs per-package), any `gagle/prepare-dist@v1` usage.
- For private registries: the registry URL, the `NPM_TOKEN` secret
  reference.
- Custom verification commands (e.g., `pnpm nx run-many -t lint
  typecheck build test` for Nx).

The wrappers are thin — they invoke the marketplace baseline and add
context. Don't reimplement the workflow logic.

## 6. Run a release

```
/release
```

This loads your project's wrapper, which invokes `/solo-npm:release`
internally. Both bodies sit in context; Claude reasons over the
merged guidance.

The release skill walks through:

1. **Phase A**: pre-flight (`/verify` + `npm-trust --doctor`; silent if green)
2. **Phase B**: plan + one `AskUserQuestion`
3. **Phase C**: execute (commit, tag, push, watch CI, verify on registry)

## 7. Operate the portfolio

After your first release, the portfolio operations skills become
useful:

- **Daily**: `/solo-npm:status` for a quick dashboard.
- **Monthly or on CVE alert**: `/solo-npm:audit` for security triage,
  then `/solo-npm:deps` to apply upgrades.

Both are read-friendly entry points — they don't mutate state without
an `AskUserQuestion` gate.

## Troubleshooting

- **`PACKAGE_NOT_PUBLISHED` from doctor**: your package isn't on npm
  yet. See Phase 2 of `/solo-npm:init` — first publish must be manual
  via `npm publish --provenance=false --access public`. After that,
  re-invoke `/solo-npm:init` and it'll continue to Phase 3 (trust).
- **`REGISTRY_PROVENANCE_CONFLICT`**: `publishConfig.registry` points
  at a custom URL but `provenance: true` is set. Either remove
  provenance (custom registries can't sign with Sigstore) or change
  the registry back to public npm.
- **CI fails on tag push**: OIDC trust likely isn't configured. Run
  `/solo-npm:trust`.
- **`/help` doesn't show `/solo-npm:*` invocations**: the marketplace
  + plugin install may not have completed. Run
  `/plugin marketplace list` to confirm `gllamas-skills` is added,
  then `/plugin install solo-npm@gllamas-skills`. Then `/reload-plugins`.
- **`/release` (wrapper) and `/solo-npm:release` (baseline) both
  appear**: that's expected — the wrapper is your customized entry
  point; the baseline is invokable directly to bypass the wrapper.
  Daily DX is `/release`.
