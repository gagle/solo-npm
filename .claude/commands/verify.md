---
description: Run quality gates — lint + typecheck + test + build + pkg-check (publint + tarball audit + manual checks). Triggers from prompts like "verify", "run the gates", "did I break anything", "did tests pass", "is my package.json publish-ready", "will my tarball ship correctly", "are there any secrets in my package", "check my exports map". Halts on first failure. Wrap via .claude/skills/verify/SKILL.md for repo-specific commands. Supports `--pkg-check-only` mode to skip lint/test/build and run only Step 5.
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

Run in order, halting on the first non-zero exit (per the severity rules in §pkg-check):

1. Lint
2. Typecheck (if separate from lint)
3. Test
4. Build
5. **Pkg-check** — `package.json` completeness + tarball-content audit (see below)

### `--pkg-check-only` mode

Run only Step 5. Skips lint/typecheck/test/build. Used by `/solo-npm:init` Phase 1d to validate scaffolds without re-running the full suite, and by users who want a fast publish-readiness check.

## Step 5 — pkg-check

Three tiers:

| Tier | Tool | Purpose |
|---|---|---|
| **Tier 1** | `publint` | Industry-standard manifest validator. Catches malformed `exports`, missing `types` for TS, deprecated fields, etc. |
| **Tier 2** | manual checks | publint blind spots: LICENSE file presence, README non-empty, `engines.node` ↔ `.nvmrc` consistency, **`.gitignore` vs `files`/`exports` divergence**. |
| **Tier 3** | `npm pack --dry-run` | Tarball-content audit: secrets, lockfiles, test files, oversize, missing LICENSE/README in the actual tarball. |

For monorepos, run all three tiers per workspace package, in parallel.

### Tier 1 — publint (canonical manifest validator)

Run via the detected package manager:

```bash
# pnpm:
pnpm dlx publint@0.3 --strict --json
# npm:
npx publint@0.3 --strict --json
# yarn:
yarn dlx publint@0.3 --strict --json
```

Pin `@0.3` (or whichever current major) so output schema is stable. Bump explicitly when needed.

Parse JSON output → array of `{ message, type, name, code, args }`. Map severity:

- publint `error` → solo-npm error (halts release-path)
- publint `warning` → solo-npm warning
- publint `suggestion` → solo-npm warning (low priority — info-only is fine)

Surface each finding as: *"publint [`<code>`]: `<message>`"*.

publint covers (non-exhaustive): `EXPORTS_GLOB_NO_PATTERN_MATCH`, `EXPORTS_TYPES_INVALID_FORMAT`, `IMPLICIT_INDEX_D_TS`, `MISSING_TYPES`, `MAIN_FIELD_NOT_FOUND`, `EXPORTS_VALUE_INVALID`, `USE_EXPORTS_BROWSER`, `USE_TYPE_CONDITION`, deprecated `directories.bin`, deprecated `bundleDependencies`, etc.

### Tier 2 — manual checks

#### 2a. LICENSE file in package dir

Check `<package-dir>/LICENSE` (or `LICENSE.md`, `LICENSE.txt`). Even if `package.json#license` field is set, the file should ship.

- Severity: warning
- Auto-fix: offer MIT scaffold with `{year} {author}` from `package.json#author` (fallback to `git config user.name`).

#### 2b. README.md exists and non-empty

Check `<package-dir>/README.md` exists and is non-trivial (>50 bytes, has at least one heading).

- Severity: warning
- Auto-fix: offer minimal stub from `package.json#name` + `description`.

#### 2c. `engines.node` ↔ `.nvmrc` consistency

If both `package.json#engines.node` and `.nvmrc` are set, they shouldn't contradict. Examples:

- `engines.node: ">=24"` and `.nvmrc: 24` → consistent
- `engines.node: ">=20"` and `.nvmrc: 24` → consistent (24 satisfies >=20)
- `engines.node: ">=22 <24"` and `.nvmrc: 24` → **contradiction** (24 doesn't satisfy upper bound)

- Severity: warning
- No auto-fix (intent is ambiguous; user must resolve).

#### 2d. `.gitignore` vs `files`/`exports` divergence (NEW)

The canonical "I shipped an empty tarball" gotcha: build output is gitignored AND not in the `files` allowlist → tarball is empty.

Algorithm:

1. Parse `.gitignore` (best-effort). v1 supports: bare paths (`dist`, `lib/`), wildcards (`*.log`), comments (`#`), trailing-slash dir matches. **Skips negation rules (`!`)** — treats them as warnings the user must review manually. Document this in the surface output.
2. Collect publish-critical paths from `package.json`:
   - `main`, `module`, `types`, `typings`
   - All paths under `exports.*` (recurse into nested condition objects)
   - All paths under `bin.*`
3. For each publish-critical path:
   - Check if any `.gitignore` pattern matches it (path-based glob match).
   - If matched: check `package.json#files` array.
     - No `files` field OR no entry covers the matched path → **error** with: *"`<field>` points at `<path>` which is gitignored. The published tarball will be missing it. Add `files: ["<path>", ...]` to package.json or stop gitignoring it."*

Auto-fix: offer to add `files: [<gitignored-but-needed paths>, "package.json", "README.md", "LICENSE"]`. User accepts → write to package.json → re-run Tier 2 (and Tier 3) to confirm.

### Tier 3 — `npm pack --dry-run` tarball-content audit (NEW)

Run via the detected package manager:

```bash
# pnpm:
pnpm pack --dry-run --json
# npm:
npm pack --dry-run --json
# yarn (yarn 1.x doesn't support --json on pack; fall back to text parse):
yarn pack --dry-run
```

Output (npm/pnpm format):

```json
[{
  "id": "pkg@1.0.0",
  "name": "pkg",
  "version": "1.0.0",
  "size": 12345,
  "unpackedSize": 23456,
  "files": [
    { "path": "package.json", "size": 567 },
    { "path": "dist/index.js", "size": 12345 }
  ],
  "entryCount": 2
}]
```

Apply pattern audits to the file list:

| Pattern | Severity | Reason | Auto-fix |
|---|---|---|---|
| `.env`, `.env.*` (excludes `.env.example`/`.env.sample`/`.env.template`) | **error** | Secrets — never publish | NO auto-fix; STOP loudly |
| `*.key`, `*.pem`, `id_rsa*`, `*credentials*`, `secrets*.json` | **error** | Likely-secret patterns | NO auto-fix; STOP loudly |
| `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock` | warning | Lockfiles shouldn't ship — clutter, can confuse consumers | Suggest `files` allowlist |
| `**/*.test.*`, `**/*.spec.*`, `__tests__/**`, `tests/**`, `test/**` | warning | Test files inflate tarball | Suggest `files` allowlist |
| `tsconfig*.json` | info | Sometimes intentional for type ergonomics | None |
| `.git*`, `.DS_Store`, `Thumbs.db`, `node_modules/**` | warning | Should never ship | Suggest `.npmignore` entries |
| `.vscode/**`, `.idea/**` | warning | Editor configs | Suggest `files` allowlist |
| (no `LICENSE` in tarball) | warning | Should ship even if `license` field is set | Offer MIT scaffold (covered by Tier 2a) |
| (no `README.md` in tarball) | warning | npm registry shows the README | Offer stub (covered by Tier 2b) |

Size audits:

- `unpackedSize > 5 MB` → warning. Surface top-10-largest-files breakdown so user can investigate.

**B6 defensive JSON parsing (v0.11.0)**: validate the pack output schema before reading fields. npm 7 may not emit `unpackedSize` or `entryCount`; private/legacy registries may emit a different shape. Apply the canonical defensive-parse pattern from `/unpublish` Phase −1.4c:

```javascript
// Validate the response is well-formed BEFORE reading fields
if (!Array.isArray(packOutput) || packOutput.length === 0) {
  // Fall back to text-mode pack parse (yarn 1.x already does this; npm/pnpm rarely needs it)
  // Surface non-fatal warning: "pack JSON output unexpected; falling back to text parse"
}
const pkg = packOutput[0];
let unpackedSize = pkg.unpackedSize;
if (typeof unpackedSize !== 'number') {
  // npm 7 may omit; sum file sizes manually
  unpackedSize = (pkg.files || []).reduce((sum, f) => sum + (f.size || 0), 0);
}
const entryCount = (typeof pkg.entryCount === 'number') ? pkg.entryCount : (pkg.files || []).length;
```

If `pkg.files` is also missing/empty, surface a HARD warning: *"npm pack --dry-run returned no file list; tarball audit cannot proceed. Investigate via `npm pack --dry-run` (without --json) for human-readable output."*
- `entryCount > 200` → info (suggests potential over-inclusion).

#### Secrets-detection: hard STOP

When ANY of the secrets-pattern matches, halt with this verbatim message in **all** invocation contexts (including `/verify` standalone — secrets leakage is too dangerous to defer to a later pass):

```
❌ SECRETS DETECTED IN TARBALL

The following files would be published if you released right now:
  <file-1> (<size> bytes)
  <file-2> (<size> bytes)

Fix BEFORE releasing:
  1. git rm --cached <file>
  2. echo "<file>" >> .gitignore
  3. echo "<file>" >> .npmignore   (or add a `files` allowlist to package.json)
  4. ROTATE any credentials that were in this file (assume them leaked).
  5. Re-run /solo-npm:verify --pkg-check-only.

This is a HARD STOP. /release will not proceed.
```

##### Post-publish: secrets already shipped in a prior release

If `/verify` ran post-fact (e.g., on an existing repo) and the secrets pattern is already in a published tarball, both halves of the fix matter — but they're not equivalent:

1. **Rotate the leaked credentials immediately.** This is the load-bearing fix. The secrets were already cached by mirrors, indexed by archives, and possibly installed by consumers; assume them compromised.
2. **Then, if the affected version is within npm's 72h unpublish window, run `/solo-npm:unpublish`** (or `npm unpublish <pkg>@<version>`) to remove the artifact. Outside the 72h window, `/solo-npm:unpublish` will refuse — fall back to `/solo-npm:deprecate` and rely on rotation.

Unpublish without rotation is **not sufficient**: any consumer who already installed the version still has the secrets in their `node_modules`.

Auto-fix is NOT offered for secrets — the user must do `git rm --cached` themselves and rotate the credentials manually. Auto-removing the file from the tarball without git+rotation steps would leave secrets in `git log` history, which is worse than a published tarball.

#### CRLF-in-tarball detection (Tier-4 #8, v0.13.0) — HARD STOP for executable scripts

The v0.12.0 J check detects `core.autocrlf=true` and warns. v0.13.0 extends this with **detection of actual CRLF in critical published files** — bin scripts, shebang files, `.sh` files. CRLF in those is a real consumer-breaking problem on Unix (the shebang line `#!/usr/bin/env node\r` doesn't parse).

After `npm pack --dry-run --json` provides the file list, scan critical files for CRLF:

```bash
# Files where CRLF causes runtime breakage on Unix consumers:
CRITICAL_PATTERNS='bin/.*|.*\.(sh|bash|zsh)$|^[^/]+$'   # bin/* + *.sh|.bash|.zsh + repo-root scripts

# Helper: extract a file from the would-be tarball without actually publishing
# pnpm pack writes a .tgz to disk in --dry-run? No — pnpm/npm --dry-run is metadata-only.
# We need to actually pack to a temp dir and inspect:
TMP_TARBALL=$(mktemp -d)/check.tgz
timeout 60 npm pack --pack-destination "$(dirname "$TMP_TARBALL")" >/dev/null 2>&1 || {
  echo "WARN: npm pack failed during CRLF check; skipping CRLF detection"
  return 0
}
TARBALL=$(ls "$(dirname "$TMP_TARBALL")"/*.tgz | head -1)

# Extract + scan
EXTRACT_DIR=$(mktemp -d)
tar -xzf "$TARBALL" -C "$EXTRACT_DIR" 2>/dev/null
CRLF_FILES=()
for f in $(find "$EXTRACT_DIR" -type f); do
  REL_PATH="${f#"$EXTRACT_DIR"/package/}"
  # Match REL_PATH against CRITICAL_PATTERNS
  if echo "$REL_PATH" | grep -qE "$CRITICAL_PATTERNS"; then
    # Check first 4KB for CRLF (avoid scanning entire large files)
    if head -c 4096 "$f" | grep -q $'\r'; then
      CRLF_FILES+=("$REL_PATH")
    fi
  fi
done

# Cleanup
rm -rf "$EXTRACT_DIR" "$(dirname "$TMP_TARBALL")"

if [ ${#CRLF_FILES[@]} -gt 0 ]; then
  echo "❌ CRLF DETECTED IN PUBLISHED EXECUTABLES"
  echo
  echo "The following files in your tarball contain Windows CRLF line endings:"
  printf '  %s\n' "${CRLF_FILES[@]}"
  echo
  echo "This breaks Unix consumers. The classic case: a shebang line"
  echo "  #!/usr/bin/env node\\r"
  echo "fails to parse on Linux/macOS — the kernel sees an interpreter named 'node\\r' and errors with 'no such file'."
  echo
  echo "Fix BEFORE releasing:"
  echo "  1. Set core.autocrlf=input for this repo: git config core.autocrlf input"
  echo "  2. Re-checkout the affected files: git checkout -- <file>"
  echo "     (Or set in .gitattributes: '*.sh text eol=lf' / 'bin/* text eol=lf')"
  echo "  3. Re-run /solo-npm:verify --pkg-check-only."
  echo
  echo "This is a HARD STOP. /release will not proceed."
  exit 1
fi
```

Scope: only files matching `CRITICAL_PATTERNS` (bin scripts + shell scripts + repo-root scripts). Don't false-positive on docs/config files where CRLF is harmless on Unix consumers. The check only fires on *.tgz extraction, so it's `npm pack`-bound — for repos that don't actually pack executables, it's a free no-op.

#### Auto-fix offer: suggest `files` allowlist

When test files, lockfiles, or editor configs are detected AND there's no `files` field in `package.json`:

```
Header:   "Tarball cleanup"
Question: "Add a `files` allowlist to limit what ships?"
Options:
  - Yes — set files: ["dist", "package.json", "README.md", "LICENSE"] (Recommended)
  - Customize — specify entries
  - Skip — surface as warnings only
```

On Yes: write to `package.json#files` → commit `chore(pkg): add files allowlist for clean tarball` → re-run Tier 3 to confirm.

### Tier 4 — bundle-size regression detection (NEW in v0.7.0)

After Tier 3's `npm pack --dry-run` parses `unpackedSize`, compare against the previous published version's recorded size.

**Cache layout** (extends `.solo-npm/state.json#pkgCheck`):

```json
{
  "pkgCheck": {
    "lastSize": {
      "@scope/foo@1.5.0": 51234,
      "@scope/foo@1.5.1": 52100,
      "@scope/bar@0.9.0": 12888
    },
    "lastFullScan": "2026-05-04T11:00:00Z",
    "ttlDays": 1
  }
}
```

The cache is keyed by `<pkg-name>@<version>` so size history accumulates per release. Updated by `/release` Phase C.7.6, `/prerelease` Phase C.7.6, `/hotfix` Phase E.7 after each successful publish.

**Algorithm**:

1. From Tier 3's pack output: read `unpackedSize` (current would-be tarball).
2. Look up the most recent prior version's recorded size in `pkgCheck.lastSize`. (For monorepos, do this per package.) The "prior version" is the highest version `< NEXT_VERSION` (or current) recorded for this package.
3. If no prior size cached → **first-run baseline**. Silently record nothing here (the cache update runs from /release post-publish, not from /verify pre-publish). No warning.
4. If prior size cached → compute `delta = (current - prior) / prior * 100`.
5. Severity:
   - `delta > +25%` → **warning** (significant growth — investigate).
   - `delta < -25%` → **info** (significant shrinkage — usually intentional, e.g., dropped dep).
   - `|delta| ≤ 25%` → no surface (normal evolution).

**Top-5-largest breakdown** (when warning fires):

```
⚠ Bundle-size regression on @ncbijs/eutils
  Prior (v1.5.0):  52,100 bytes (50.9 KB unpacked)
  Current:        128,540 bytes (125.5 KB unpacked)
  Delta:          +146.7%

  Top 5 largest files in current tarball:
    dist/some-vendored-lib.js   72,431 bytes  (NEW vs prior — likely accidentally bundled)
    dist/index.js               31,200 bytes
    dist/utils.js                8,440 bytes
    package.json                   612 bytes
    README.md                      210 bytes

  Investigate: did a dep get accidentally bundled? Did the build emit unminified output?
  No auto-fix — size regressions need human investigation.
```

**Auto-fix**: none. Size deltas are too contextual to auto-resolve.

**Special: drop the warning if the version bump is major** (e.g., 1.5.0 → 2.0.0). Major versions are expected to grow significantly; don't surface as a regression. (Heuristic: if the major part changed, suppress Tier 4. Minor/patch within the same major still trigger.)

**Optional integration with `size-limit`**: if the repo has a `size-limit` config (`.size-limit.cjs`/`.size-limit.json` or `package.json#size-limit`), also run `pnpm dlx size-limit` and surface any failing budgets as Tier 4 errors (size-limit's own thresholds, not our 25% heuristic). This is opt-in via the presence of the config; we don't add the dep.

## Combined auto-fix offers

| Trigger | Tier | Action |
|---|---|---|
| Missing `repository.url` | 1 (publint warning) | Derive from `git remote get-url origin` |
| Missing `homepage` | 1 | `repository.url + "#readme"` |
| Missing `bugs.url` | 1 | `repository.url + "/issues"` |
| Missing `engines.node` | 1 | Set `>=24` (matches `/init` default) |
| Missing `LICENSE` file | 2a | MIT scaffold with `{year} {author}` |
| Empty/missing `README.md` | 2b | Stub from `name` + `description` |
| publint reports `IMPLICIT_INDEX_D_TS` (TS pkg missing types) | 1 | Probe `./dist/index.d.ts`, `./types/index.d.ts`, `./build/index.d.ts`; offer if exactly one matches |
| `.gitignore` vs missing `files` divergence | 2d | `files: [<gitignored-but-needed paths>, "package.json", "README.md", "LICENSE"]` |
| Test/lockfile/editor in tarball + no `files` field | 3 | `files: ["dist", "package.json", "README.md", "LICENSE"]` (sensible default) |
| Secrets in tarball | 3 | NO auto-fix — STOP loudly |

Each accepted auto-fix produces a separate `chore(pkg): <fix>` commit (no mega-commits, clean `git log`).

### `AskUserQuestion` shape

When multiple fixes are available, surface a single batched question:

```
Header:   "Pkg-check fixes"
Question: "Apply <N> auto-fixes? (one commit per fix; review before pushing)"
Options:
  - Apply all <N> fixes (Recommended)
  - Apply selected (sub-prompt with multiSelect)
  - Skip fixes — surface as warnings only
  - Abort
```

If the user picks "Apply selected", show a multiSelect `AskUserQuestion` listing each fix.

### Severity by context

| Invocation context | Errors | Warnings | Auto-fix offers |
|---|---|---|---|
| `/verify` standalone (mid-development) | Surface; **continue** verify (don't halt) | Surface | Yes — batched |
| `/solo-npm:verify --pkg-check-only` (e.g., from `/init` Phase 1d) | Halt; offer auto-fixes | Surface | Yes |
| `/solo-npm:release` Phase A.2 (pre-flight) | **Halt the release**; offer auto-fixes before STOP | Surface; allow proceed | Yes — fix-and-retry |
| `/solo-npm:release` Phase C.4 (post-bump) | Halt (shouldn't reach here) | Surface only | No |
| `/solo-npm:prerelease` Phase A | Halt; offer auto-fixes | Surface | Yes |
| `/solo-npm:hotfix` Phase A | Halt; offer auto-fixes | Surface | Yes |
| `/solo-npm:deps` after-batch | **Skipped** (deps don't touch manifest/tarball) | — | — |

**Special: secrets-detection error in Tier 3 halts ALL contexts**, including `/verify` standalone. Too dangerous to defer.

### Caching

- Hash inputs: `{ packageJsonHash, gitignoreHash, distOutputContents, otherRelevantFiles }`.
- Cache `{ publint, manual, pack }` results in `.solo-npm/state.json#pkgCheck` keyed by hash.
- Hash invalidation handles freshness; no TTL needed.
- Cache survives `/release` runs so back-to-back invocations skip recomputation.

### Output

- **All clean**: `PKG_CHECK_OK` (or `PKG_CHECK_OK_CACHED` if cache hit).
- **Warnings only**: per-warning lines + `PKG_CHECK_WARN: <N> warnings`.
- **Errors found** (release-path): per-error lines + `PKG_CHECK_FAIL: <E> errors, <W> warnings`. Halt the caller.
- **Errors + auto-fix accepted**: apply → re-run → if clean, print `PKG_CHECK_OK_AFTER_FIXES` and continue.
- **Secrets detected**: hard STOP with the verbatim secrets-detection block. ALL contexts.

## Output

- **Success**: print `VERIFY_OK` on its own line and exit cleanly.
- **Failure**: print the failing command, the last ~50 lines of its output, and `VERIFY_FAIL: <one-line diagnosis>`.

## When this skill is called

- **Manually**: invoke `/verify` (wrapper) or `/solo-npm:verify` (baseline) after a change to confirm it's ready.
- **Pkg-check only**: `/solo-npm:verify --pkg-check-only` for fast publish-readiness check; called from `/solo-npm:init` Phase 1d.
- **Automatically**: `/solo-npm:release` calls this in Phase A.2 (pre-flight) and Phase C.4 (post-bump verification). `/solo-npm:prerelease` and `/solo-npm:hotfix` invoke in their Phase A pre-flights.
- **Composed**: `/solo-npm:deps` calls this between dep-upgrade batches; rolls back on failure. Skips Tier 5 (deps don't touch manifest/tarball).

As long as the verify skill returns `VERIFY_OK` for a clean state, downstream skills proceed.
