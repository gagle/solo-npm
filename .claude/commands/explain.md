---
description: Explain why a release CI run failed in plain English. Fetches gh run logs, surfaces the failing step + likely cause, and suggests next steps mapped to /solo-npm:* skills. Triggers from prompts like "explain the CI failure", "why did the release CI fail?", "what's wrong with the last gh run", "diagnose the failed release". Wraps gh + AI synthesis.
---

# Explain

Diagnose a failed release CI run. Fetches `gh run view --log-failed`, surfaces the failing step + AI-synthesized explanation, suggests remediation as concrete `/solo-npm:*` skill invocations.

This skill is intentionally narrow: it explains the **release CI** specifically (the workflow scaffolded by `/init` and used by `/release`). It does NOT explain arbitrary CI failures (lint runs, PR test runs, etc.) — those are project-build concerns out of solo-npm's scope.

## Phase −0 — Help mode

If `--help`/`-h`/similar, surface brief summary and STOP.

## When to use

- After the release CI fails and you want to know what to fix
- After a failed `/release` run where the user pushed the tag manually and CI is failing
- Investigating a recurring CI failure pattern

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `RUN_ID` | `--run <id>` flag or trigger phrase | last failed run on the release.yml workflow |
| `JSON_OUT` | `--json` | `false` |

## Phase 1 — Locate the failed run

```bash
WORKFLOW=$(detect_release_workflow)   # e.g., "release.yml"

if [ -n "$RUN_ID" ]; then
  TARGET_RUN="$RUN_ID"
else
  TARGET_RUN=$(gh run list --workflow "$WORKFLOW" --status failure --limit 1 --json databaseId | jq -r '.[0].databaseId')
fi

if [ -z "$TARGET_RUN" ]; then
  echo "No failed runs of '$WORKFLOW' found in this repo."
  exit 0
fi
```

## Phase 2 — Fetch logs

```bash
gh run view "$TARGET_RUN" --json jobs > /tmp/run-meta.json
gh run view "$TARGET_RUN" --log-failed > /tmp/run-failed.log
```

Parse `run-meta.json` to identify which jobs failed and which steps within each job. Extract the relevant log slice for each failure.

## Phase 3 — AI synthesis

For each failure, pass the (job name, step name, log excerpt) to Claude (the agent driving the skill) for explanation. The prompt template:

> A solo-dev's npm package release CI failed. The repo uses solo-npm conventions (npm-trust for OIDC trusted publishing, prepare-dist optionally, GitHub Actions). Below is the failing step's log excerpt. Explain in plain English:
> 1. Which step failed and what it was trying to do
> 2. The most likely root cause
> 3. The exact remediation (which solo-npm skill to run, OR a concrete file edit, OR a manual action)
>
> Step: `<step-name>`
> Log:
> ```
> <log excerpt>
> ```

The skill captures Claude's response (which is itself running this skill body) as the synthesized explanation.

## Phase 4 — Map to remediation

Common failure patterns and their canonical remediations:

| Pattern in log | Likely cause | Suggested skill |
|---|---|---|
| `npm error 401` or `EAUTH` | OIDC trust not configured for this package | `/solo-npm:trust` |
| `npm error 403` and `--access` mention | publishConfig.access wrong (private vs public) | manual edit |
| `npm error 403 Forbidden - PUT https://registry.npmjs.org/...` | Trust binding mismatch (workflow file != registered) | `/solo-npm:trust` to re-verify |
| `npm error 409 Conflict` | Version already published | manual: bump version, retry |
| `npm error EOTP` | One-time password required (token auth path) | manual: re-publish locally with 2FA |
| `pnpm: command not found` | pnpm setup step failing | check `pnpm/action-setup@v4` step |
| `tsc: command not found` | typescript not in devDeps | manual: install |
| `Error: Process completed with exit code N` (vague) | Need full log inspection | surface the surrounding context |
| `attw failed` or `arethetypeswrong` problem | Dual-package issue | `/solo-npm:types` to re-run locally |
| `publint warnings` | package.json shape issue | `/solo-npm:verify` Tier 4 to inspect |
| `prepare-dist: Tag version "X" does not match` | Tag mismatch | manual: re-tag with correct version |
| Network/DNS errors (`ETIMEDOUT`, `EAI_AGAIN`) | Transient | wait + retry |
| GHA: `id-token: write` permission missing | release.yml drift | `/solo-npm:init --refresh-yml` |

The skill cross-references the detected pattern with this table; if no match, falls back to the AI-synthesized explanation.

## Phase 5 — Render

### Human (default)

```
explain — gh run #25502822722 (failed 2 hours ago)
Workflow: release.yml • Trigger: push tag v0.19.0

Failed jobs:
  publish (id: 74838534333)

Failed step:
  Publish to npm

Log excerpt:
  npm error 403 403 Forbidden - PUT https://registry.npmjs.org/@scope/foo
  npm error 403 You must verify your email address before publishing.

What happened:
  The publish step rejected the push because the npm account email is not verified.
  The OIDC trust is configured correctly (the request reached the registry), but
  npm's policy requires email verification on the publishing account.

Likely cause:
  Account email verification has lapsed (npm requires this for write operations).

Suggested remediation:
  1. Log into https://www.npmjs.com/settings/<your-username>/profile
  2. Verify the account email address
  3. Re-trigger the workflow: gh run rerun 25502822722
```

When the failure pattern doesn't match the table:

```
What happened:
  The publish workflow attempted to install dependencies but pnpm command was
  not found on the runner. The setup-pnpm action may not have completed
  successfully OR the workflow's pnpm step is misconfigured.

Likely cause:
  release.yml's `pnpm/action-setup@v4` step may be missing or misconfigured.

Suggested remediation:
  Run /solo-npm:init --refresh-yml to re-emit the canonical release.yml
  template (which includes the correct pnpm setup).
```

### --json

```ts
interface ExplainReport {
  schemaVersion: 1;
  run: {
    id: string;
    url: string;
    workflow: string;
    triggeredAt: string;
    conclusion: 'failure' | 'cancelled' | …;
  };
  failures: ReadonlyArray<{
    job: string;
    step: string;
    logExcerpt: string;
    pattern: string | null;     // matched pattern from the table
    summary: string;            // AI-synthesized
    likelyCause: string;        // AI-synthesized
    suggestedAction: {
      kind: 'skill' | 'manual';
      skill?: string;           // e.g., '/solo-npm:init --refresh-yml'
      manualAction?: string;    // human-readable instruction
    };
  }>;
}
```

Exit codes:
- 0 always (read-only diagnostic)
- 1 if gh CLI is missing or auth fails

## Error-handling

- **H6** if gh fails (no auth, network), surface and STOP
- **H8** rate-limit backoff for gh API calls

## Notes

- This skill is read-only: it FETCHES logs and EXPLAINS. It does not apply remediation. The user invokes the suggested skill explicitly.
- AI synthesis quality depends on log clarity. For very large logs, only the failing step's excerpt is sent to AI (with surrounding 50 lines).
- For non-publish CI failures (lint runs, PR test runs), this skill explicitly does NOT diagnose them — those are project-build concerns. Future scope expansion (per PART II) could add a generic `/solo-npm:explain --any-workflow`.
- gh CLI is required. If unauthenticated (`gh auth status` fails), surface install + auth instructions and STOP.
