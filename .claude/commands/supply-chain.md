---
description: Deep supply-chain audit — provenance state of every transitive dep, license compatibility, package-age risk, typosquatting detection, maintainer-count signals. Goes beyond /audit (CVE-only) to surface the broader supply-chain risk surface. Triggers from prompts like "supply chain audit", "check deps risk", "license audit", "are my deps trustworthy?", "deep audit". Use as a release-gate or quarterly review.
---

# Supply chain

Walks the lockfile and produces a per-dep risk score across multiple axes: provenance present, CVE state, package age, maintainer count, license compatibility, typosquatting signals. Far heavier than `/audit` (which is CVE-only) — use periodically (release-gate or quarterly review), not every build.

## Phase −0 — Help mode

If `--help`/`-h`/similar, surface brief summary and STOP. Note: heavy on `npm view` calls; cache aggressively.

## When to use

- Quarterly supply-chain review
- Pre-major-release audit (heavier safety check before a 1.0.0 stamp)
- After adding a new dep (verify it's not high-risk)
- Investigating a suspicious dep flagged by `/lockfile-audit`

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `JSON_OUT` | `--json` | `false` |
| `FRESH` | `--fresh` (skip cache) | `false` |
| `INTERACTIVE` | `--interactive` (triage prompt) | `false` |
| `LICENSE_TARGET` | `--license <spdx>` (target license your package uses) | inferred from `package.json#license` |

## Phase 1 — Read lockfile

```bash
LOCKFILE=pnpm-lock.yaml  # or package-lock.json
```

Parse and produce a deduplicated list of `(name, version)` pairs across the entire transitive graph.

## Phase 2 — Per-dep probes (cached, parallel)

For each `(name, version)` in the dep graph:

### 2.1 Cache lookup

Read `state.json#caches.supplyChain[<name>@<version>]`. If present and not stale (TTL 7 days for static metadata; 1 day for CVEs), use cached.

### 2.2 npm view

```bash
DATA=$(npm view "$NAME@$VERSION" --json 2>/dev/null)
```

Extract:
- `dist.attestations` → provenance present
- `time.created` → publish age (days since `now`)
- `maintainers[]` → maintainer count
- `license` → SPDX license id
- `repository` → home repo URL
- `homepage`

### 2.3 osv-scanner (CVEs)

```bash
osv-scanner --lockfile=pnpm-lock.yaml --json > /tmp/osv-result.json
```

Run once for the whole lockfile (more efficient than per-dep). Parse to map `(name, version) → CVE[]`.

### 2.4 Typosquat detection

For each direct dep (not transitive), check name similarity to common popular packages. Use a simple Levenshtein distance against a hardcoded list of top-1000 npm packages (cached as a static fixture). Distance ≤ 2 raises a typosquat flag.

(Transitive deps are flagged less aggressively — we don't add risk for popular libs depending on each other's lookalikes.)

### 2.5 License compatibility

Compare each dep's license against `LICENSE_TARGET`. Compatibility matrix:

- MIT/Apache-2.0/ISC/BSD-* are mutually compatible (permissive)
- LGPL-2.1+ compatible only if your project is also LGPL+
- GPL-3-only INCOMPATIBLE with MIT-licensed packages
- AGPL INCOMPATIBLE with most commercial uses
- Unknown/missing license → warn (high risk)

## Phase 3 — Risk scoring

Per-dep score (0-10):

| Signal | Weight |
|---|---|
| No provenance | +2 |
| Maintainer count == 1 | +1 |
| Package age < 90 days | +1 |
| Typosquat suspicion (direct deps only) | +3 |
| Any CVE (severity scaled: low=1, medium=2, high=3, critical=4) | +severity*2 |
| License incompatible | +3 |
| License unknown/missing | +2 |

Categorize:
- 0-2: low risk
- 3-5: medium risk
- 6+: high risk

## Phase 4 — Render

### Human

```
supply-chain — 142 deps audited (3 direct, 139 transitive)

High-risk (1):
  ⚠ left-pad@1.0.5
    no provenance | 1 maintainer | typosquat suspicion (similar to 'leftpad')

Medium-risk (3):
  some-dep@2.0.0           CVE GHSA-xxxx-yyyy-zzzz (medium)
  another-dep@0.0.5        no provenance, 1 maintainer
  obscure-dep@1.0.0        license: 'Custom' (unknown SPDX)

Low-risk: 138 deps OK.

License compatibility (target: MIT):
  ✓ All deps compatible with MIT.

Summary: 1 high, 3 medium, 138 low. 0 license incompatible.

Action:
  - Review left-pad@1.0.5 — replace with a trusted alternative
  - Investigate `obscure-dep@1.0.0` license
```

### --json

```ts
interface SupplyChainReport {
  schemaVersion: 1;
  target: { license: string };
  deps: ReadonlyArray<{
    name: string;
    version: string;
    direct: boolean;
    provenance: boolean;
    cves: ReadonlyArray<{ id: string; severity: 'low'|'medium'|'high'|'critical' }>;
    ageDays: number;
    maintainerCount: number;
    license: string;
    licenseCompatible: boolean;
    typosquatSuspect: boolean;
    riskScore: number;
    riskTier: 'low' | 'medium' | 'high';
  }>;
  summary: {
    total: number;
    direct: number;
    transitive: number;
    highRisk: number;
    mediumRisk: number;
    lowRisk: number;
    licenseIncompatible: number;
    typosquatSuspects: number;
    cveCount: number;
  };
}
```

Exit codes:
- 0 if no high-risk
- 40 (VALIDATION_FAILURE) if any high-risk
- 1 if osv-scanner / npm view fails

### --interactive

After rendering, AskUserQuestion gates:
- For each high-risk dep: `Replace / Accept risk / Defer triage`
- The skill does NOT auto-replace; surfaces the suggested alternative + leaves the user to apply

## Cache strategy

Per `(name, version)`:
- Static metadata (provenance, age, maintainers, license, repo): TTL 7 days
- CVEs: TTL 1 day (volatile)

Cache stored at `state.json#caches.supplyChain` with size cap (last 200 deps; LRU evict).

## Integration

- `/audit` is the fast CVE-only path; /supply-chain is the comprehensive path.
- `/doctor`'s `security-gate` domain reads the cached `/supply-chain` summary; surfaces high-risk count without re-running.
- `/release` Phase A optional gate via `--supply-chain-strict`: STOP if high-risk count > 0.

## Error-handling

- **H8** rate-limit backoff for `npm view` calls (parallel; respect 429s).
- **H6** chain-failure recovery if osv-scanner is missing (degrade to "no CVE info available; consider installing osv-scanner").
- **H7** atomic state.json writes for cache updates.

## Notes

- This skill is HEAVY (potentially N×network calls for first scan of a project with hundreds of deps). After first run, the cache makes subsequent runs fast.
- The risk score formula is documented above; tunable via solo-cli-conventions in the future. For now it's hardcoded in the skill body.
- License compatibility is a heuristic — for legal certainty, consult a lawyer. The skill flags risks; it doesn't assert legal correctness.
- Typosquat detection is a best-effort heuristic. False positives possible; treat as "investigate" not "blocked."
