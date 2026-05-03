---
name: verify
description: >
  Run lint, test, and build to confirm a change is ready to ship.
  Default commands target a typical pnpm-based TypeScript repo.
  Customize for your stack — add e2e steps, coverage thresholds, etc.
  Other skills (especially /release) call this in their pre-flight phase.
---

# Verify

Run the verification gates for this repo. Halt on the first failure;
surface the error to the user.

## Default commands

```bash
<LINT_CMD>      # default: pnpm run lint
<TEST_CMD>      # default: pnpm test
<BUILD_CMD>     # default: pnpm run build
```

Run sequentially. Halt on the first non-zero exit.

## Output

- **Success**: print `VERIFY_OK` on its own line and exit cleanly.
- **Failure**: print the failing command, the last ~50 lines of its
  output, and `VERIFY_FAIL: <one-line diagnosis>`.

## Customization

This skill ships with sensible defaults. Edit it to match your stack:

- Add `pnpm test:e2e` for end-to-end tests.
- Add `pnpm typecheck` if your project separates lint and typecheck.
- Add `pnpm test --coverage --threshold 90` for coverage gates.
- Replace `pnpm` with `npm` or `yarn` if needed.
- Add domain-specific checks (schema validation, license audit,
  accessibility, bundle size limits).

## When this skill is called

- **Manually**: invoke `/verify` after a change to confirm it's ready.
- **Automatically**: `/release` calls this in Phase A.2 (pre-flight)
  and Phase C.4 (post-bump verification).

As long as `/verify` returns `VERIFY_OK` for a clean state, downstream
skills proceed.
