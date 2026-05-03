# Installing solo-npm-release-skill

Step-by-step install + customization guide.

## 1. Add the marketplace and install the plugin

In Claude Code:

```
/plugin marketplace add gagle/solo-npm-release-skill
/plugin install solo-npm-release-skill@gllamas-skills
```

This installs two skills into your repo's `.claude/skills/`:

- `release/SKILL.md` — the canonical `/release` skill
- `verify/SKILL.md` — a default `/verify` skill

## 2. Edit the placeholders in `release/SKILL.md`

Open `.claude/skills/release/SKILL.md` and find the **Placeholders**
table near the top. Replace each placeholder throughout the file:

| Placeholder | What to put |
|---|---|
| `<PACKAGE_NAME>` | Your npm package name (e.g., `rfc-bcp47`) |
| `<REPO_SLUG>` | `<owner>/<repo>` (e.g., `gagle/rfc-bcp47`) |
| `<RELEASE_WORKFLOW>` | The filename of your publish workflow (e.g., `release.yml`) |
| `<LINT_CMD>`, `<TEST_CMD>`, `<BUILD_CMD>` | The actual commands for your stack |

If your repo is **single-package**, delete the monorepo block (it's
clearly marked).

If your repo is a **monorepo**, keep the monorepo block and replace
`<MAIN_PACKAGE_DIR>` with the path to the package whose version is
canonical (e.g., `packages/core`).

## 3. Customize `verify/SKILL.md` if needed

The default `/verify` skill runs lint + test + build. Edit
`.claude/skills/verify/SKILL.md` to add anything else your stack
needs:

- `pnpm test:e2e` for e2e tests
- `pnpm typecheck` if separate from lint
- Coverage gates, license audits, bundle size limits

## 4. Verify prerequisites

Make sure you have:

- A tag-triggered `.github/workflows/release.yml` that publishes via
  `pnpm publish --no-git-checks`. The publish step relies on
  `package.json#publishConfig` (set `access: "public"` and
  `provenance: true` for public npm).
- OIDC trust configured for your package. If not yet set up, install
  the trust setup wizard:
  ```
  pnpm add -D npm-trust
  pnpm exec npm-trust --init-skill setup-npm-trust
  ```
  Then invoke `/setup-npm-trust` in Claude Code.

## 5. Run the skill

In Claude Code, type:

```
/release
```

The skill walks through:

1. **Phase A**: pre-flight (silent if green)
2. **Phase B**: shows the plan, asks **one** structured question
3. **Phase C**: executes (commit, tag, push, watch CI, verify on registry)

## Troubleshooting

- **`PACKAGE_NOT_PUBLISHED` from doctor**: your package isn't on npm
  yet. See the
  [first-publish ceremony](https://github.com/gagle/npm-trust/blob/main/docs/bootstrap.md#5-first-publish--the-chicken-and-egg)
  in the npm-trust docs.
- **`REGISTRY_PROVENANCE_CONFLICT`**: `publishConfig.registry` points
  at a custom URL but `provenance: true` is set. Either remove
  provenance (custom registries can't sign with Sigstore) or change
  the registry back to public npm.
- **CI fails on tag push**: the publish workflow likely needs OIDC
  trust to be configured. Run the `/setup-npm-trust` wizard first.
