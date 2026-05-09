---
description: Scan repo + git history for accidentally committed secrets — wraps gitleaks with structured output and remediation guidance. Triggers from prompts like "scan for secrets", "did I commit a token?", "secret leak audit", "gitleaks check". Use as a release-gate or after a near-miss event. Findings are masked; no actual secret values surface.
---

# Secrets audit

Scan the repository AND its git history for accidentally committed secrets (API keys, tokens, passwords, private keys). Wraps [gitleaks](https://gitleaks.io) with a structured `SecretsAuditReport` shape and remediation guidance.

Critical: **all surfaced findings have masked secret values** — only file:line + secret type are shown. The actual secret content is never echoed.

## Phase −0 — Help mode

If `--help`/`-h`/similar, surface brief summary and STOP. Note: scans the entire git history; can take a while on large repos.

## When to use

- Pre-release: confirm the working tree + history don't expose secrets
- After accidentally committing a `.env` or test fixture
- Quarterly security review
- Before making a private repo public

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `JSON_OUT` | `--json` | `false` |
| `HEAD_ONLY` | `--head-only` flag | `false` (full git history by default) |
| `CONFIG_FILE` | `--config <path>` | `.gitleaks.toml` if exists |

## Phase 1 — Resolve gitleaks

```
1. ./node_modules/.bin/gitleaks                  (devDep)
2. command -v gitleaks                           (global)
3. ~/.local/bin/gitleaks (or homebrew path)      (system)
4. STOP with install hint                         (no fallback — gitleaks is OS-binary, not npm)
```

If unresolved, surface install hint:

> Install gitleaks for solo-npm:secrets-audit:
> - macOS:    brew install gitleaks
> - Linux:    See https://github.com/gitleaks/gitleaks#installation
> - Windows:  See https://github.com/gitleaks/gitleaks#installation
>
> Then re-run /solo-npm:secrets-audit.

## Phase 2 — Run gitleaks

For full history scan:

```bash
gitleaks detect \
  --source . \
  --report-format json \
  --report-path /tmp/gitleaks.json \
  ${CONFIG_FILE:+--config "$CONFIG_FILE"} \
  --no-banner
```

For HEAD-only (faster):

```bash
gitleaks detect \
  --source . \
  --report-format json \
  --report-path /tmp/gitleaks.json \
  --no-git \
  ${CONFIG_FILE:+--config "$CONFIG_FILE"}
```

Parse `/tmp/gitleaks.json` (gitleaks emits an array of findings).

## Phase 3 — Process findings

For each finding:

- Mask the actual secret value: replace with `[REDACTED:<rule-id>]`
- Capture: `file`, `line`, `commit` (SHA), `rule` (e.g., 'aws-access-key', 'github-pat'), `severity` (rule-derived)

Severity mapping (gitleaks-rule → solo-npm severity):
- AWS keys, GCP service account keys, GitHub PATs, Stripe keys → `high`
- Generic API keys, npm tokens → `medium`
- Generic password patterns → `low`
- Private keys (RSA/PEM) → `high`

## Phase 4 — Render

### Human

```
secrets-audit — full history scan (4823 commits)

Findings: 0
✓ No secrets detected.

Scan completed in 3.2s.
```

When findings present:

```
Findings: 2 (1 high, 1 medium)

✗ HIGH    aws-access-key
   File:    examples/test-fixture.json
   Line:    42
   Commit:  abc1234 (2 weeks ago)
   Value:   [REDACTED:aws-access-key]

   Remediation:
     1. ROTATE the key immediately at AWS console
     2. Run: git filter-repo --path examples/test-fixture.json --invert-paths
        (rewrites history to remove the file; coordinate with collaborators)
     3. After history rewrite: git push --force (after coordination)

⚠ MEDIUM  generic-api-key
   File:    src/integration.ts
   Line:    18
   Commit:  HEAD (uncommitted)
   Value:   [REDACTED:generic-api-key]

   Remediation:
     - Move to environment variable: process.env.SERVICE_API_KEY
     - Add to .env.example with placeholder
     - Add the actual key to your secrets manager (1Password, AWS Secrets, etc.)

Action: Address all findings BEFORE the next /release.
```

### --json

```ts
interface SecretsAuditReport {
  schemaVersion: 1;
  scope: { repoRoot: string; historyScanned: boolean; commitsScanned: number };
  findings: ReadonlyArray<{
    file: string;
    line: number;
    commit: string;        // SHA where introduced
    rule: string;
    severity: 'low' | 'medium' | 'high';
    masked: string;        // always [REDACTED:<rule>]
  }>;
  summary: {
    findingsCount: number;
    bySeverity: { high: number; medium: number; low: number };
    ok: boolean;
  };
}
```

Exit codes:
- 0 if no findings
- 40 (VALIDATION_FAILURE) if any high or medium findings
- 1 if gitleaks itself failed

## Integration

- `/release` Phase A optional gate via `--secrets-strict`: STOP if any high finding
- `/doctor`'s `security-gate` domain surfaces cached secrets-audit findings
- Cache: write summary (NOT raw findings — privacy) to `state.json#caches.secretsAudit` with TTL 1 day
- Pre-commit hook (via `/init --refresh-yml` future variant) — runs HEAD-only scan on staged changes

## Error-handling

- **H6** chain-failure recovery if gitleaks crashes
- **H7** atomic state.json writes for cache updates

## Configuration

Optional `.gitleaks.toml` at repo root for project-specific allow rules (test fixtures with intentional fake tokens, etc.). Example:

```toml
[allowlist]
description = "solo-npm allow rules"
paths = [
  '''test/__fixtures__/.+\.(json|env)$''',
]
regexes = [
  '''AKIA[A-Z0-9]{16}EXAMPLE''',  # known dummy AWS key in docs
]
```

If `.gitleaks.toml` is missing, gitleaks uses its default rules.

## Notes

- The skill ALWAYS masks secret values — even in JSON output, only the rule type is exposed. To debug a specific finding, the user runs gitleaks directly with their own logging.
- Full-history scans are SLOW on large repos (4-12 minutes for 100k+ commits). Use `--head-only` for quick checks; full scans for release gates.
- gitleaks is an OS binary (Go), not an npm package. solo-npm doesn't install it; surfaces the install hint and STOPs if absent.
- Findings in git history are HARDER to remediate than working-tree findings — must rewrite history (e.g., `git filter-repo`), which forces a coordinated push for any collaborators.
