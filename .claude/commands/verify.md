---
description: Run quality gates — lint + typecheck + test + build with auto-detected commands. Triggers from prompts like "verify", "run the gates", "did I break anything", "did tests pass", "make sure everything passes", "quick sanity check before commit". Halts on first failure. Wrap via .claude/skills/verify/SKILL.md for repo-specific commands (e2e, coverage, nx run-many).
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
first non-zero exit:

1. Lint
2. Typecheck (if separate from lint)
3. Test
4. Build

## Output

- **Success**: print `VERIFY_OK` on its own line and exit cleanly.
- **Failure**: print the failing command, the last ~50 lines of its
  output, and `VERIFY_FAIL: <one-line diagnosis>`.

## When this skill is called

- **Manually**: invoke `/verify` (wrapper) or `/solo-npm:verify`
  (baseline) after a change to confirm it's ready.
- **Automatically**: `/solo-npm:release` calls this in Phase A.2
  (pre-flight) and Phase C.4 (post-bump verification).
- **Composed**: `/solo-npm:deps` calls this between dep-upgrade
  batches; rolls back on failure.

As long as the verify skill returns `VERIFY_OK` for a clean state,
downstream skills proceed.
