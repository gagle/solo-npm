---
description: Run quality gates — lint + typecheck + test + build + pkg-check with auto-detected commands. Triggers from prompts like "verify", "run the gates", "did I break anything", "did tests pass", "make sure everything passes", "quick sanity check before commit", "is my package.json publish-ready". Halts on first failure. Wrap via .claude/skills/verify/SKILL.md for repo-specific commands (e2e, coverage, nx run-many).
---

# Verify

> **This is the opinionated baseline.** Consumer repos typically wrap it
> via `.claude/skills/verify/SKILL.md` to specify the exact commands
> (Nx run-many for monorepos, e2e tests, coverage thresholds). Invoke
> `/solo-npm:verify` directly only when you want the unmodified default.

Run the verification gates for this repo. Halt on the first failure;
surface the error to the user.

## Auto-detection

Detect the package manager and verification commands automatically:

| Value | Source |
|---|---|
| Package manager | `pnpm-lock.yaml` → pnpm; `yarn.lock` → yarn; else npm |
| Lint command | `package.json#scripts.lint` (or omit if absent) |
| Typecheck command | `package.json#scripts.typecheck` (or omit if absent) |
| Test command | `package.json#scripts.test` (or omit if absent) |
| Build command | `package.json#scripts.build` (or omit if absent) |

If a wrapper at `.claude/skills/verify/SKILL.md` provides explicit
commands (e.g., `pnpm nx run-many -t lint typecheck build test`), use
those instead.

## Default sequence

Run the detected commands in this order, sequentially, halting on the
first non-zero exit (per the severity rules in §pkg-check):

1. Lint
2. Typecheck (if separate from lint)
3. Test
4. Build
5. **Pkg-check** — `package.json` completeness validation (see below)

## Step 5 — pkg-check

Validate that each `package.json` in the workspace is publish-ready. For monorepos, validate every package's manifest individually.

### Fields validated

| Field / file | Severity | Auto-fix offer | Reasoning |
|---|---|---|---|
| `name` | error | — | Required by npm |
| `version` | error | — | Required |
| `license` | error | offer MIT scaffold | Publish-critical (consumers need to know) |
| `exports` OR `main` | error | — | Consumers can't import without one |
| `private: true` AND `publishConfig.access: "public"` | error | — | Contradictory — pick one |
| `description` | warning | — | Shown in npm search results |
| `keywords` | warning | — | Discoverability |
| `repository.url` | warning | derive from `git remote get-url origin` | Provenance + GitHub linking |
| `homepage` | warning | derive from repository URL | Landing page |
| `bugs.url` | warning | derive from repository URL + `/issues` | Issue tracker |
| `files` (or `.npmignore`) | warning | — | Controls what gets in the tarball |
| `engines.node` | warning | offer `>=24` (matches `/init` default) | Compatibility hint |
| `LICENSE` file in package dir | warning | offer MIT scaffold | Many tools/CI check the file |
| `README.md` exists, non-empty | warning | offer stub generator | Registry shows it; users expect it |

### Severity by context

`pkg-check` behaves differently depending on **who invoked verify**:

| Invocation context | Errors | Warnings | Auto-fix offers |
|---|---|---|---|
| `/verify` standalone (mid-development) | Surface; **continue** verify (don't halt) | Surface | Yes — one batched `AskUserQuestion` per fixable item |
| `/solo-npm:release` Phase A.2 (pre-flight) | **Halt the release**; offer auto-fixes before STOP | Surface; allow proceed | Yes — fix-and-retry loop before STOP |
| `/solo-npm:release` Phase C.4 (post-bump) | (Errors should already be caught in A.2; if any reach here, halt) | Surface only; no fix offers | No |
| `/solo-npm:deps` after-batch | **Skipped** (deps don't touch manifest) | — | — |
| `/solo-npm:prerelease` Phase A | Halt; offer auto-fixes | Surface | Yes |
| `/solo-npm:hotfix` Phase A | Halt; offer auto-fixes | Surface | Yes |

The skill detects its caller via the invoking context. Standalone is the lenient mode; release-path is the strict mode.

### Auto-fix scaffolds

When a fix offer is accepted via `AskUserQuestion`:

| Fix | Action |
|---|---|
| `repository.url` | Run `git remote get-url origin`; normalize to `git+https://github.com/<owner>/<repo>.git`; write to `package.json#repository.url` (and `repository.type = "git"`). |
| `homepage` | Derive `https://github.com/<owner>/<repo>#readme`. |
| `bugs.url` | Derive `https://github.com/<owner>/<repo>/issues`. |
| `engines.node` | Set `>=24` (matches `/solo-npm:init` default). |
| `LICENSE` file | Generate MIT license text with `{year} {authorName}` from `package.json#author` (fall back to `git config user.name`). Write to `<package-dir>/LICENSE`. |
| `README.md` stub | Minimal: `# <pkg-name>\n\n<description from package.json>\n\n## Installation\n\n\`\`\`bash\nnpm install <pkg-name>\n\`\`\`\n` (or pnpm/yarn). |

Each accepted auto-fix produces a separate commit:

```
chore(pkg): set <field> from <source>
chore(pkg): scaffold MIT LICENSE
chore(pkg): scaffold README.md stub
```

The pkg-check step does NOT batch all auto-fixes into a single mega-commit; separate commits give clean `git log` history.

### `AskUserQuestion` shape

When fixes are available, surface a single batched question:

```
Header:   "Pkg-check fixes"
Question: "Apply <N> auto-fixes? (one commit per fix; review before pushing)"
Options:
  - Apply all <N> fixes (Recommended)
  - Apply selected (sub-prompt with multiSelect)
  - Skip fixes — surface as warnings only
  - Abort
```

If the user picks "Apply selected", show a multiSelect AskUserQuestion listing each fix as a checkbox.

### Output

- **All checks pass**: print `PKG_CHECK_OK` on its own line.
- **Warnings only**: print each warning + `PKG_CHECK_WARN: <count> warnings`.
- **Errors found** (release-path): print each error + `PKG_CHECK_FAIL: <count> errors, <count> warnings`. Halt the caller.
- **Errors found + auto-fix accepted**: apply fixes; re-run pkg-check; if clean, print `PKG_CHECK_OK_AFTER_FIXES` and continue.

## Output

- **Success**: print `VERIFY_OK` on its own line and exit cleanly.
- **Failure**: print the failing command, the last ~50 lines of its output, and `VERIFY_FAIL: <one-line diagnosis>`.

## When this skill is called

- **Manually**: invoke `/verify` (wrapper) or `/solo-npm:verify`
  (baseline) after a change to confirm it's ready.
- **Automatically**: `/solo-npm:release` calls this in Phase A.2
  (pre-flight) and Phase C.4 (post-bump verification).
- **Composed**: `/solo-npm:deps` calls this between dep-upgrade
  batches; rolls back on failure.

As long as the verify skill returns `VERIFY_OK` for a clean state,
downstream skills proceed.
