---
description: Post-publish verification — fetch the just-published version's attestations from the registry and confirm the SLSA provenance is valid. Triggers from prompts like "verify provenance", "check the published version's attestations", "did provenance work?", "confirm the SLSA signature". Wraps npm-trust --verify-provenance.
---

# Provenance verify

Post-publish: confirm the just-published tarball has valid SLSA provenance attestations on the npm registry. Mostly a thin wrapper around `npm-trust --verify-provenance --json`; surfaces a human-friendly report and integrates with `/release` Phase C.7 verification.

## Phase −0 — Help mode

If the prompt contains `--help`/`-h`/similar, surface a brief summary and STOP.

## When to use

- After a `/release` to confirm the published artifacts have provenance
- After fixing a broken release.yml (`/init --refresh-yml`) to verify next publish works
- Auditing existing packages: confirm the entire portfolio has consistent provenance state
- CI gate: `--json` exit codes integrate

## Phase 0 — Read prompt context

| Slot | Source | Default |
|---|---|---|
| `PACKAGES` | `--packages <name...>` flag | inferred via `--auto` workspace detection |
| `JSON_OUT` | `--json` | `false` |
| `FRESH` | `--fresh` (skip cache) | `false` |

## Phase 1 — Resolve packages

If `--packages` is set, use the explicit list. Otherwise call npm-trust to detect:

```bash
PACKAGES=$(npx -y npm-trust@latest --auto --doctor --json 2>/dev/null | jq -r '.workspace.packages[]')
```

For monorepos this surfaces every workspace package.

## Phase 2 — Run npm-trust --verify-provenance

```bash
npx -y npm-trust@latest --packages "${PACKAGES[@]}" --verify-provenance --json > /tmp/provenance.json
```

Returns a `VerifyProvenanceReport` (schemaVersion 2 since npm-trust v0.11.0):

```jsonc
{
  "schemaVersion": 2,
  "packages": [
    {
      "pkg": "@scope/foo",
      "latestVersion": "1.5.2",
      "provenancePresent": true,
      "attestationCount": 2,
      "lastAttestationAt": "2026-05-08T10:00:00Z",
      "unpackedSize": 51234
    }
  ],
  "summary": { "total": 1, "withProvenance": 1, "withoutProvenance": 0, "unpublished": 0 }
}
```

## Phase 3 — Render

### Human

```
provenance-verify — 3 packages

  @scope/foo        ✓ 1.5.2  (2 attestations, last: 2026-05-08T10:00:00Z, 51 KB)
  @scope/bar        ✓ 0.9.0  (1 attestation, last: 2026-04-22T08:00:00Z, 28 KB)
  @scope/baz        ✗ 0.1.0  (no provenance — was this published before OIDC trust was set up?)

Summary: 2 with provenance, 1 without, 0 unpublished.

Action:
  - @scope/baz needs a re-publish via the OIDC release.yml to get provenance.
    Run /solo-npm:trust to verify trust is configured, then /solo-npm:release.
```

### --json

Pass through `VerifyProvenanceReport` directly (already conformant JSON).

Exit codes:
- 0 if every published package has `provenancePresent: true`
- 40 (VALIDATION_FAILURE) if any published package lacks provenance
- 0 with warning if `unpublished > 0` (not a failure; just absence)

## Integration with /release

`/release` Phase C.7 already invokes provenance verification inline. This skill is the standalone analog for ad-hoc auditing AND the orchestration target if `/release` Phase C.7 is refactored to chain into this skill rather than inlining the logic.

For the v0.19.0 release, /release C.7 stays inline (no behavior change); /provenance-verify exists as the standalone skill for auditing.

## Error-handling

- **H4** registry propagation: just-published versions may take a few seconds to appear with attestations. If `attestationCount === 0` immediately post-publish, retry 3× with 5s backoff.
- **H8** rate-limit backoff if npm view hits 429.

## Notes

- Provenance is npm-side; this skill surfaces the registry's view. To verify the LOCAL build matches the registry's attestation, additional cryptographic checks would be needed (out of scope for v0.19.0).
- `unpackedSize` was added to the report in npm-trust v0.11.0 / VerifyProvenanceReport schemaVersion 2; v0.10.x versions don't include it (graceful degradation).
- For monorepos, all workspace packages are checked in a single bulk call.
