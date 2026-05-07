---
description: Unpublish or rename npm packages safely — single version (within 72h or post-72h criteria), full-package removal (--force), or rename-redirect (deprecate old + publish new). Triggers from prompts like "unpublish @wrong/foo@1.0.0", "delete the published v1.0.0", "I shipped under the wrong scope", "rename @gagle/eutils to @ncbijs/eutils", "I changed my mind about the package name", "remove @wrong/foo entirely". Strict safety gates: two AskUserQuestion confirmations for destructive ops; HARD STOP if version has dependents; defaults to deprecate (npm's recommended path).
---

# Unpublish

Recovery skill for the rare-but-real cases where the published artifact itself needs to go away or be redirected — not just deprecated. Use sparingly. **The default-recommended path is `/solo-npm:deprecate` for almost everything**; this skill is the destructive escape hatch with strict safety gates.

## When to use

| Scenario | Right tool |
|---|---|
| Wrong release shipped (bad code, but version number is fine to abandon) | `/solo-npm:dist-tag repoint` + `/solo-npm:deprecate` |
| Vulnerable version that can't be fixed, but tests pass | `/solo-npm:deprecate` (CVE-affected range) |
| Just want to retire an old major | `/solo-npm:deprecate` (range-deprecation) |
| **Wrong package name shipped (typo, wrong scope)** | `/solo-npm:unpublish` (this skill) |
| **Renamed the package (mind change after publish)** | `/solo-npm:unpublish rename-redirect` (this skill) |
| **Accidentally published secrets in tarball** | `/solo-npm:unpublish` (this skill) within 72h **AND credential rotation**. Rotation is non-optional and the load-bearing fix; this skill only handles artifact cleanup. |

## Why npm itself recommends deprecate over unpublish

[npm's official policy](https://docs.npmjs.com/policies/unpublish/) discourages unpublish:

- **Unpublish breaks consumer lockfiles**. Anyone who pinned the unpublished version in `package-lock.json` / `pnpm-lock.yaml` gets a broken `npm install` on next clean install.
- **The same version number cannot be republished** (npm enforces version immutability after the 24h-or-72h window).
- **Deprecation preserves install-ability** — consumers who pinned the bad version keep working but see a `npm WARN` message on every install. They migrate at their own pace.

This skill **defaults to recommending deprecate** in every gate. Unpublish is opt-in destructive.

## Operations

| Operation | Effect |
|---|---|
| `unpublish-version` | Remove a single version (e.g., `@scope/foo@1.0.0`); subject to eligibility |
| `unpublish-all` | Remove the entire package across all versions (`--force`); subject to eligibility |
| `rename-redirect` | Multi-step: deprecate old name with a redirect message + (optional) unpublish old eligible versions. The new package gets published separately via `/init` or `/release` |

## Phase −1 — External-tool reliability (H7 canonical pattern)

Strict-safety baseline applied **before** any other phase. This is the canonical reference for the H7 pattern; sibling skills reference this block. Every external tool the skill will invoke must be verified upfront with three checks: **presence**, **version**, and **timeout-wrappable invocation**. The user gets a clear remediation path if anything fails — no silent hangs, no cryptic later-phase errors.

### −1.1 Tool presence + version detection

```bash
# Required tools for /unpublish:
#   npm   — registry mutations + auth
#   git   — local tag cleanup (Phase D.3 if user opts in)
#   gh    — GitHub Release cleanup (Phase D.3 if user opts in)
#   curl  — deps.dev eligibility check (Phase A.3)
#   jq    — JSON parsing
#   node  — state.json cache cleanup script

require_tool() {  # args: $1 = tool name, $2 = optional minimum-version semver, $3 = canonical install URL
  if ! command -v "$1" >/dev/null 2>&1; then
    echo "ERROR: required tool '$1' is not installed."
    echo "       Install from: $3"
    exit 1
  fi
  # Optional version pin (best-effort — version output formats differ per tool)
  if [ -n "$2" ]; then
    VERSION_OUT=$("$1" --version 2>&1 | head -1)
    echo "  ✓ $1 detected: $VERSION_OUT"
  fi

  # Multi-installation detection (Tier-4 #7, v0.13.0): if multiple copies are on PATH,
  # surface them as a diagnostic so the user knows which one we got.
  ALL_PATHS=$(which -a "$1" 2>/dev/null)
  PATH_COUNT=$(echo "$ALL_PATHS" | wc -l | tr -d ' ')
  if [ "$PATH_COUNT" -gt 1 ]; then
    USED_PATH=$(command -v "$1")
    echo "  ℹ multiple '$1' installations on PATH (using $USED_PATH):"
    echo "$ALL_PATHS" | sed 's/^/      /'
    echo "      If the wrong one was selected, reorder PATH or use the explicit path."
  fi
}

require_tool npm  ""  "https://nodejs.org"
require_tool git  ""  "https://git-scm.com"
require_tool curl ""  "https://curl.se"
require_tool jq   ""  "https://jqlang.github.io/jq/"
require_tool node ""  "https://nodejs.org"

# gh is optional (only used in Phase D.3 if user picks auto-cleanup)
if command -v gh >/dev/null 2>&1 && gh auth status >/dev/null 2>&1; then
  GH_AVAILABLE=1
else
  GH_AVAILABLE=0  # Phase D.3 will skip GitHub Release cleanup with a warning
fi
```

### −1.2 Universal timeout wrapper

Every external network call in this skill **must** be wrappable to bound execution time. Hangs (stale tokens, slow registries, ISP DNS issues) are a strict-safety failure: the user gets no signal that anything is wrong. Convention:

| Call type | Timeout | Why |
|---|---|---|
| `npm whoami` | 30s | Auth check; long enough for slow networks, short enough to surface hung tokens |
| `npm view ...` | 30s | Read-only registry lookups |
| `npm unpublish/deprecate/...` | 60s | Mutations may take longer especially with 2FA flows |
| `curl <api>` | `--max-time 10 --connect-timeout 5` | External APIs: deps.dev, downloads-counter |
| `gh <subcommand>` | 30s | GitHub API |
| `git ls-remote/fetch/push` | 60s | Network git operations |

Implementation: wrap with `timeout(1)` if available (Linux + macOS via `coreutils`), or use the tool's native timeout flag where one exists. Fail-mode: timeout → surface *"<tool> timed out after Ns — network unreachable, token stale, or service slow. Retry, or check manually."*

### −1.3 `.gitconfig` user identity check

```bash
GIT_USER_NAME=$(git config user.name 2>/dev/null)
GIT_USER_EMAIL=$(git config user.email 2>/dev/null)
if [ -z "$GIT_USER_NAME" ] || [ -z "$GIT_USER_EMAIL" ]; then
  echo "ERROR: git config user.name and user.email must be set before running /solo-npm:unpublish."
  echo "       Without them, `git tag -d` operations in Phase D.3 work, but any future commit"
  echo "       (e.g., re-publishing under correct name) will fail with cryptic exit-128 errors."
  echo
  echo "Fix:   git config --global user.name 'Your Name'"
  echo "       git config --global user.email 'you@example.com'"
  exit 1
fi
```

This is checked even though `/unpublish` itself doesn't make commits — it's the entry point check for the H7 pattern, applied universally so the user fixes config issues once. Skills that DO commit (release, prerelease, hotfix, deps, init) absolutely depend on this.

### −1.3b Environment-variable validation (Tier-3 D, v0.12.0)

```bash
# $HOME must exist and be writable — npm writes ~/.npmrc, ~/.npm/_cacache
if [ -z "$HOME" ] || [ ! -w "$HOME" ]; then
  echo "ERROR: \$HOME is unset or not writable (HOME='$HOME')."
  echo "       npm requires \$HOME to read/write ~/.npmrc and ~/.npm/_cacache."
  echo "       This usually only happens in sandboxed CI environments."
  echo "       Fix:   export HOME=/path/to/writable/home   then re-invoke."
  exit 1
fi

# $TMPDIR fallback for subprocess temp files
: "${TMPDIR:=/tmp}"
if [ ! -w "$TMPDIR" ]; then
  echo "ERROR: \$TMPDIR='$TMPDIR' is not writable."
  echo "       Subprocess temp files (e.g., curl response capture) need it."
  echo "       Fix:   export TMPDIR=/tmp   or another writable dir, then re-invoke."
  exit 1
fi
```

Niche check (most environments are fine), but solo-npm-managed CI environments occasionally hit it.

### −1.4 Atomic state.json write helper (used in Phase D.2)

Every state.json write in this skill (and the H7 reference for sibling skills) uses an atomic write pattern: write to a temp file, then rename. Non-atomic `writeFileSync` truncates the file before writing, so a process killed mid-write leaves a zero-byte (or partially-written) cache.

```javascript
// Used wherever the skill writes .solo-npm/state.json:
const tmp = '.solo-npm/state.json.tmp';
fs.writeFileSync(tmp, JSON.stringify(state, null, 2) + '\n');
fs.renameSync(tmp, '.solo-npm/state.json');
// rename() is atomic on POSIX; on the same filesystem, it's a single inode swap.
```

If the process is killed between writeFile and rename, the leftover `.tmp` file is detected on next read (H2 corruption guard already wraps `JSON.parse` in try/catch — the `.tmp` file is harmless because it's not the real state file).

**Schema preservation on writes (D1, v0.11.0)**: every `state.json` write uses **read-modify-write** — the entire object is read in, the relevant section mutated, the entire object written back. Never replace a top-level section with `state.section = newValue` (that would silently drop any unknown keys a future schema added). Pattern:

```javascript
const state = JSON.parse(fs.readFileSync(path, 'utf8'));
// Mutate ONLY the keys you intend to update; leave everything else untouched
state.audit = state.audit || {};
state.audit.tier1Count = newCount;
state.audit.lastFullScan = new Date().toISOString();
// Other state.audit.* fields (e.g., a future `state.audit.scoreCard`) are preserved.
// Atomic write
const tmp = path + '.tmp';
fs.writeFileSync(tmp, JSON.stringify(state, null, 2) + '\n');
fs.renameSync(tmp, path);
```

This makes `state.json` forward-compatible: an older skill version reads + preserves fields it doesn't understand, instead of truncating them. A user mixing skill versions across repos (or mid-upgrade) gets non-destructive behavior.

### −1.4b `npm view` defensive schema parsing (D2)

`npm view <pkg> --json` output schema varies subtly across npm 7/8/9/10 and across registry hosts. Specifically:

| Field | Variability |
|---|---|
| `dist.attestations` | Missing entirely on private registries that don't support provenance; missing on packages predating Sigstore |
| `time.<version>` | npm 7/8 returns ISO-8601 (`"2024-03-15T12:34:56.789Z"`); some private registries return Unix epoch as a number |
| `time.modified`, `time.created` | Always present, but format same as above |
| `dist-tags` | Always object, but may be empty `{}` (no tags) on never-published packages |
| `versions` | Empty array `[]` on packages that exist but have no published versions (rare; legacy) |
| `repository` | Can be string (legacy) or object (current); both are valid |

Convention: **never assume a field exists or has a fixed type without checking**. Wrap field accesses in fallbacks:

```bash
# When parsing time.<version>:
PUBLISHED_AT=$(npm view "<pkg>@<v>" "time.<v>" 2>/dev/null)
# Try ISO-8601 first; fall back to epoch parse; fall back to "unknown"
case "$PUBLISHED_AT" in
  [12][0-9][0-9][0-9]-*) ;;        # ISO-8601 (starts with year)
  [0-9]*) PUBLISHED_AT=$(date -d "@$((PUBLISHED_AT/1000))" -Iseconds);;  # Unix ms epoch
  *) PUBLISHED_AT="unknown";;
esac

# When checking dist.attestations:
HAS_PROVENANCE=$(npm view "<pkg>@<v>" dist.attestations --json 2>/dev/null | jq -r 'if . == null then "no" else "yes" end')
```

In skill bodies, document the fallback inline rather than assuming the happy-path schema. If a field is essential and missing → STOP with diagnostic, not assume default.

**npm CLI BCL extensions (G, v0.12.0)**: additional cross-version cases beyond the table above:

| Field / call | Variability | Fallback |
|---|---|---|
| `dist-tags` returned as `{}` empty object | Equivalent to "no tags configured" — common on never-published packages | Treat as no-`@latest`-yet; fine in /unpublish where the package is being removed anyway. In other skills, surface "no dist-tags set" rather than crashing |
| `dist-tags` field entirely missing | Some private registries omit it | Same as above — treat as empty |
| `npm whoami` returns `@username` (with `@` prefix) | Some npm versions/configurations | Strip leading `@` before comparison: `WHOAMI=${WHOAMI#@}` |
| `npm whoami` returns email rather than username | Account configured with email-as-identifier (rare) | Use as-is for comparison; document that the username displayed may not match the package owner-list format |
| `npm audit --json` schema | npm 6 uses `{actions[], advisories{}, metadata{}}`; npm 7+ uses `{vulnerabilities{}, metadata{}}` | Detect via presence of `actions` field → npm 6 fallback. v0.10.0+ documents this in /audit Phase 1 |
| `npm version` output | Older versions print prefixed `v`; newer print bare semver | Always parse with semver lib, not raw string match |
| `npm config get <key>` returning `undefined` literal vs empty string | Older npm prints `undefined`; newer prints empty | Normalize: `[ -z "$VAL" ] || [ "$VAL" = "undefined" ]` → treat as unset |

#### Comprehensive BCL table (Tier-4 #2, v0.13.0)

Beyond the high-impact cases above, here's the full enumeration of every `npm <subcmd>` call solo-npm makes + its known cross-version variations + the convention we apply at every call site.

| npm call | Skills using it | Cross-version variation | Convention |
|---|---|---|---|
| `npm publish` | release (CI), prerelease (CI), hotfix (CI), init Phase 2c | npm 9+ supports `--provenance=true` (OIDC-required); older needs explicit auth token | OIDC path goes through CI — uniform. Local Phase 2c uses `--provenance=false` to bypass attestation (chicken-and-egg) |
| `npm publish --json` | release C.7 verify (post-publish read) | Schema introduced in npm 9; older versions emit text-only on stdout | Don't depend on `--json` post-publish output; use `npm view <pkg>@<v>` afterwards instead (already the convention) |
| `npm view <pkg>` | All read-path skills | Returns text by default; `--json` for parseable | Always pair with `--json 2>/dev/null` per v0.10.1 stderr-redirect convention |
| `npm view <pkg> <field>` | many skills | `<field>` syntax (`time.<v>`, `dist-tags.next`) supported npm 7+ | Treat absent field as empty (jq `// 0`); detect undefined per −1.4b row 7 |
| `npm view <pkg> versions --json` | deprecate, dist-tag | Empty array on never-published packages; missing field on private-registry edge cases | Treat empty array as "no versions" gracefully, surface "no versions match" not crash |
| `npm view <pkg> dist-tags --json` | dist-tag, status, /audit | Returns `{}` on never-published; missing field on errors | Treat `{}` as "no dist-tags configured"; this differs from "registry returned error" |
| `npm view <pkg>@<v> deprecated` | deprecate verify | Returns empty string for not-deprecated; full message for deprecated | Compare exact-string match; empty = not-deprecated |
| `npm view <pkg>@<v> time.<v>` | unpublish A.2 (publish-age check) | npm 7/8: ISO-8601; some private registries: Unix epoch ms | Per −1.4b table |
| `npm view <pkg>@<v> dist.attestations` | release C.7 (provenance check), trust verify | Missing on private registries / pre-Sigstore packages | Per −1.4b table |
| `npm version <bump>` | release C.1 (auto-bump), prerelease C.1, hotfix E.2 | npm 6 prints `v1.2.3`; npm 7+ prints `1.2.3` | Always pipe through `sed 's/^v//'` to normalize |
| `npm version --no-git-tag-version <bump>` | hotfix (publishConfig.tag manipulation) | Stable across npm 7+ | None |
| `npm whoami` | trust, release A.3, every commit-and-push skill, /unpublish C.0 | Some configurations return `@username` (with `@` prefix) | `WHOAMI=${WHOAMI#@}` to strip; per −1.4b table |
| `npm whoami --registry <url>` | trust (per-package auth verify) | Same `@` prefix variation; some npm versions ignore `--registry` flag for npmjs.org | Detect via `npm config get registry`; only invoke `--registry` if non-default |
| `npm login` | Manual user step (web 2FA flow) | Stable across npm 6+; user-driven | None |
| `npm logout` | Not invoked by skills | n/a | n/a |
| `npm dist-tag add/rm/ls` | dist-tag C, release C.7 (verify), prerelease | Stable across npm 7+; npm 6 doesn't support `--json` on `ls` | Skill targets npm 7+ baseline (per package.json#engines) |
| `npm deprecate <pkg> "<msg>"` | deprecate C, unpublish C.3 (rename redirect) | Stable; message length capped at ~1024 chars (registry-side) | Phase A.5 already validates message length |
| `npm owner add/rm/ls` | owner C, unpublish A.2 (sole-owner check) | `ls` returns text by default; `--json` added npm 7+ | Always use `--json` per E from v0.10.0 |
| `npm unpublish <pkg>@<v>` | unpublish C.1 | Stable; subject to npm policy (72h, dependents, etc.) | None — policy enforcement is registry-side |
| `npm unpublish <pkg> --force` | unpublish C.2 | Stable | None |
| `npm audit --json` | audit Phase 1, deps Phase 1c | npm 6: `{actions[], advisories{}, metadata{}}`; npm 7+: `{vulnerabilities{}, metadata{}}` | Per −1.4b table; /audit Phase 1 has BCL detection |
| `npm outdated --json` | deps Phase 1b | Schema changed across npm 7→8→9 (`dependencyType` field added in npm 8) | Defensive: check for field presence, fall back to dep-tree walk if missing |
| `npm install <pkg>@<target>` | deps Phase 4b | Stable; lockfile semantics differ pnpm/yarn/npm | Detected per package manager in Phase 1a |
| `npm pack --dry-run --json` | verify Tier 3, release C.7.6 | `unpackedSize`, `entryCount` added npm 8+ | Per −1.4c |
| `npm pack --dry-run` (no --json, fallback) | yarn 1.x in /verify Tier 3 | Text output; format stable across yarn 1.x | Already documented as fallback in /verify Tier 3 |
| `npm config get <key>` | init (Tier-3 E), trust pre-flight | Returns `undefined` literal on older versions | Per `−1.4b` last row |
| `npm config get @<scope>:registry` | init (Tier-3 E) | Stable across npm 6+ | Same `undefined` normalization |
| `npm config set <key> <value>` | not invoked by skills (we read, not write) | n/a | n/a |
| `npm-trust --doctor --json` | release A.3, init Phase 2a, status, audit Phase 4 trust prefix | npm-trust@^0.4 schema (per D3 pin) | Defensive: check `schema.Version`; fall back to per-step discovery if doctor unavailable |
| `npm-trust verify` | trust Phase 6 | Same schema | Same |
| `npm-trust list --json` | trust Phase 1 | Same | Same |
| `npm-trust configure --auto` | trust Phase 9 | Same | Same; web-2FA handoff per existing trust skill body |

**Convention at every call site**: 
1. Wrap the call in `timeout 30` (or longer for CI watch)
2. Redirect stderr `2>/dev/null` if piping to a JSON parser
3. Check return code; on failure, classify per −1.10 SSL / H1 OTP / H4 propagation / general npm error
4. On success, validate expected fields exist before reading; fall back per the table above
5. For mutation calls (publish/deprecate/dist-tag/owner/unpublish), apply H1 OTP + H3 auth-race + H5 lock + H8 backoff

This BCL table is the canonical reference. Sibling skills don't duplicate the table; they apply the convention at each call site they make.

### −1.4c `npm pack --dry-run` defensive parsing (D4)

`npm pack --dry-run --json` output schema:

| Field | Stability |
|---|---|
| `name`, `version` | Stable across npm 7+ |
| `files[]` | Stable; each entry has `path` + `size` |
| `entryCount` | Added npm 8+; may be missing on npm 7 |
| `unpackedSize` | Added npm 8+; may be missing on older |
| `size` | Tarball size (always present) |

Fallback for missing `unpackedSize`:

```javascript
let unpackedSize = pack[0].unpackedSize;
if (typeof unpackedSize !== 'number') {
  // Sum file sizes manually
  unpackedSize = (pack[0].files || []).reduce((sum, f) => sum + (f.size || 0), 0);
}
```

`/verify` Tier 3 (size audit) and `/release` C.7.6 (bundle-size cache) both apply this fallback.

### −1.4d `npm-trust` npx pin (D3, applies to /trust skill)

`/trust` Pre-flight section currently uses `npx -y npm-trust@latest` as a fallback when the CLI isn't installed locally. `@latest` fetches whatever happens to be current on npm — meaning the skill's expectations about flags (`--auto`, `--doctor`, `--json`) may not be satisfied by a future npm-trust release.

Pin to a known-compatible major version:

```bash
# Instead of: npx -y npm-trust@latest
# Use:        npx -y npm-trust@^0.4
```

Bump the pin explicitly when solo-npm tests against a new npm-trust major. Document the pinned major in `/trust` Pre-flight section so future contributors know to update it deliberately.



### −1.5 Detached HEAD detection (A1, applies to commit/tag-creating skills)

Even though `/unpublish` itself doesn't create commits, the same Phase −1.5 check is the canonical reference for sibling skills (release, prerelease, hotfix, deps, init) that DO. `git status --porcelain` returns clean on detached HEAD, but commits/tags created on detached HEAD become unreachable as soon as the user checks out a different branch.

```bash
if ! git symbolic-ref -q HEAD >/dev/null 2>&1; then
  CURRENT_SHA=$(git rev-parse --short HEAD)
  echo "ERROR: detached HEAD detected (currently at $CURRENT_SHA)."
  echo "       Commits or tags created here would become unreachable when you next checkout a branch."
  echo
  echo "Fix:   git checkout main          # or your release branch"
  echo "       Then re-invoke the skill."
  exit 1
fi
```

Apply universally in Phase A of release, prerelease, hotfix, deps, and init. `/unpublish` itself doesn't strictly need this (no commits are created), but the check is cheap and the canonical reference body owns it.

### −1.5b CRLF / `core.autocrlf` detection (Tier-3 J, v0.12.0)

Windows users with `core.autocrlf=true` see fake whitespace diffs (CRLF on disk, LF in repo). For npm-shipped repos, this also affects what ends up in the tarball — Unix consumers running `node ./bin/cli` get CRLF endings that some shells mishandle.

```bash
AUTOCRLF=$(git config core.autocrlf 2>/dev/null)
if [ "$AUTOCRLF" = "true" ]; then
  echo "WARN: git config core.autocrlf=true detected on this repo."
  echo "      For npm-published packages, this can cause:"
  echo "        a) Fake whitespace diffs on every checkout (CRLF on disk vs LF in repo)"
  echo "        b) CRLF line endings in your published tarball (Unix consumers may mishandle bin scripts)"
  echo
  echo "      Recommended for npm-shipped repos:"
  echo "        git config core.autocrlf input    # checks out with LF; commits with LF"
  echo
  echo "      This is a WARNING, not a STOP — current operation continues."
fi
```

Non-fatal; surface once and continue. Skills that mutate files (release, prerelease, hotfix, deps, init) may want to elevate to a STOP if they actually detect CRLF in their output, but that's per-skill discretion.

### −1.6 Worktree detection (A2, applies universally)

A skill running inside a `git worktree` has access to the main worktree's `.git` directory but operates on a separate working tree. State reads (`git status`, `git tag`) work correctly, but `state.json` and `.solo-npm/` artifacts may live in the main worktree's checkout, not the current one — leading to confusing cache-state mismatches.

```bash
if [ "$(git rev-parse --git-path HEAD)" != ".git/HEAD" ] && \
   [ "$(git rev-parse --git-path HEAD)" != "$(git rev-parse --git-dir)/HEAD" ]; then
  echo "ERROR: detected git worktree (not the main working directory)."
  echo "       solo-npm skills assume the main worktree because .solo-npm/ artifacts and"
  echo "       package.json scaffolding decisions are tied to it."
  echo
  echo "Fix:   cd \$(git worktree list | head -1 | awk '{print \$1}')"
  echo "       Then re-invoke the skill."
  exit 1
fi
```

Apply universally in Phase A of every skill that mutates git or scaffolding state.

### −1.6b Submodule state detection (Tier-4 #6, v0.13.0)

Repos with git submodules need consistent submodule state across the supermodule's working tree, otherwise published tarballs can ship stale submodule content (the supermodule's tracked submodule SHA != what's actually checked out).

```bash
# Only run if .gitmodules exists; pure no-op for repos without submodules
if [ -f .gitmodules ]; then
  SUBMODULE_STATUS=$(git submodule status 2>/dev/null)

  # Detect uninitialized submodules (lines starting with "-")
  UNINIT=$(echo "$SUBMODULE_STATUS" | grep -c '^-' || true)
  # Detect modified submodules (lines starting with "+")
  MODIFIED=$(echo "$SUBMODULE_STATUS" | grep -c '^+' || true)
  # Detect merge conflicts in submodules (lines starting with "U")
  CONFLICTED=$(echo "$SUBMODULE_STATUS" | grep -c '^U' || true)

  if [ "$CONFLICTED" -gt 0 ]; then
    echo "ERROR: $CONFLICTED submodule(s) have unresolved merge conflicts."
    echo "       Resolve via: git submodule update --remote / --init / etc."
    echo "       Releasing in this state would publish a tarball with broken submodule refs."
    exit 1
  fi

  if [ "$MODIFIED" -gt 0 ]; then
    echo "ERROR: $MODIFIED submodule(s) have local modifications not matching the supermodule's tracked SHA."
    echo "       Lines marked with '+':"
    echo "$SUBMODULE_STATUS" | grep '^+' | sed 's/^/         /'
    echo "       Either commit the supermodule update (git add <submodule-path> && git commit)"
    echo "       OR reset the submodule (git submodule update --recursive)."
    exit 1
  fi

  if [ "$UNINIT" -gt 0 ]; then
    echo "WARN: $UNINIT submodule(s) are uninitialized (haven't been cloned)."
    echo "      If your published tarball depends on submodule content, run:"
    echo "         git submodule update --init --recursive"
    echo "      Then re-invoke. (If submodules are dev-only and excluded from the tarball, this is safe to ignore.)"
    # Non-fatal: dev-only submodules may legitimately be uninitialized in CI/release contexts.
  fi
fi
```

Apply in Phase A of skills that touch git state for release: release, prerelease, hotfix, deps. Skip for read-only skills (status, audit) and CLI-only skills (deprecate, dist-tag, owner, unpublish itself).

### −1.7 `gh` pre-flight (A3, hoisted from per-call to Phase −1)

The v0.10.0 implementation in `/unpublish` Phase −1.1 already detects `gh` presence + auth and sets `GH_AVAILABLE`. v0.11.0 hoists this same pattern to Phase A of **every** skill that uses `gh` (release, prerelease, hotfix, status), so the user knows up-front if gh is unavailable rather than discovering it mid-flow:

```bash
# Same pattern as /unpublish Phase −1.1 — applied as Phase A.0 in every gh-using skill
if command -v gh >/dev/null 2>&1; then
  if gh auth status >/dev/null 2>&1; then
    GH_AVAILABLE=1
    GH_VERSION=$(gh --version | head -1 | awk '{print $3}')
  else
    GH_AVAILABLE=0
    GH_REASON="installed but not authenticated (run 'gh auth login')"
  fi
else
  GH_AVAILABLE=0
  GH_REASON="not installed (https://cli.github.com)"
fi

# Phase A summary block surfaces this for the user:
#   gh: ready (v2.40.0)
#   gh: installed but not authenticated — run 'gh auth login' to enable CI watch + GitHub Release creation
#   gh: not installed — install from https://cli.github.com to enable CI watch + GitHub Release creation
#
# Then any later phase that wants gh checks $GH_AVAILABLE before invoking.
```

The skills using `gh` should NOT halt if `GH_AVAILABLE=0` — instead they should gracefully skip the gh-dependent step (CI watch, GitHub Release creation/deletion) with a clear "what was skipped and why" note. The user has explicit control to install/auth and re-run.

### −1.8 Stale-lock auto-cleanup (extends H5)

The lock file at `.solo-npm/locks/<sanitized>.lock` holds a PID. If the lock-holding process was killed (SIGKILL, OOM, machine crash), the lock persists. Phase C.0's H5 check is upgraded to **auto-clean stale locks** rather than refusing forever:

```bash
LOCK_FILE=".solo-npm/locks/$(echo "<NAME>" | sed 's|/|_|g').lock"
if [ -f "$LOCK_FILE" ]; then
  STALE_PID=$(cat "$LOCK_FILE" 2>/dev/null)
  if [ -n "$STALE_PID" ] && kill -0 "$STALE_PID" 2>/dev/null; then
    echo "ERROR: another /unpublish run holds $LOCK_FILE (PID $STALE_PID is alive)."
    echo "       Wait for it to finish, or kill PID $STALE_PID manually if it's hung."
    exit 1
  fi
  # PID dead or lockfile content unparseable → stale lock; clean it
  echo "WARN: removing stale lockfile $LOCK_FILE (prior holder PID $STALE_PID is no longer alive)"
  rm -f "$LOCK_FILE"
fi
mkdir -p .solo-npm/locks
echo $$ > "$LOCK_FILE"
trap 'rm -f "$LOCK_FILE"' EXIT
# NOTE: trap doesn't fire on SIGKILL — that's why the stale-cleanup above exists.
```

### −1.9 H8 rate-limit backoff helper (Tier-3 A, NEW v0.12.0)

Portfolio-fan-out operations (`/status` Phase 2, `/deprecate` Phase A.3, `/dist-tag` Phase A.3, `/owner` Phase A.3) parallelize many `npm view` calls. The npm registry rate-limits aggressive callers with HTTP 429. Currently those failures collapse into "—" with no signal that data was rate-limited (vs unavailable).

Canonical helper — sibling skills reference this implementation:

```bash
# H8 helper: run a registry call with exponential backoff on 429.
# Args: $1 = the bash command to run, captured as a string
# Returns: 0 on success (output on stdout), non-zero with diagnostic on stderr if exhausted.
npm_with_h8_backoff() {
  local CMD="$1"
  local DELAYS="1 2 4 8"  # seconds; total max wait = 15s + 4 attempts
  local ATTEMPT=0
  local OUT RC

  for DELAY in $DELAYS; do
    ATTEMPT=$((ATTEMPT + 1))
    OUT=$(eval "$CMD" 2>&1)
    RC=$?
    if [ $RC -eq 0 ]; then
      echo "$OUT"
      return 0
    fi
    # Detect 429 / rate-limit in stderr
    if echo "$OUT" | grep -qE "429|rate.?limit|Too Many Requests|E429"; then
      # Apply ±20% jitter to avoid thundering-herd retries
      JITTER=$((RANDOM % (DELAY * 4) / 10))   # 0..0.4*DELAY in tenths
      ACTUAL_DELAY=$((DELAY + JITTER / 10))
      echo "WARN: rate-limited on attempt $ATTEMPT; retrying in ${ACTUAL_DELAY}s (with jitter)" >&2
      sleep $ACTUAL_DELAY
      continue
    fi
    # Not a rate-limit error — return immediately, don't retry
    echo "$OUT" >&2
    return $RC
  done

  echo "ERROR: rate-limited after $ATTEMPT attempts (15s+ total wait). Try again later." >&2
  return 1
}

# Usage: result=$(npm_with_h8_backoff 'timeout 30 npm view @scope/foo --json 2>/dev/null')
```

Apply selectively: read-only `npm view` calls in fan-out paths benefit. Single-call paths (`/release` C.7 verify) don't need backoff — propagation lag (H4) is the dominant failure there, and H4's 3×5s retry already covers it.

### −1.9b Proactive rate-limit tracker (Tier-4 #1, v0.13.0) — for `gh` API only

H8 (above) is **reactive** — kicks in after 429. For GitHub API specifically, `gh` exposes `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers, allowing a **proactive** tracker that slows down BEFORE hitting 429.

**Why gh-only**: npm registry doesn't reliably emit X-RateLimit headers (some endpoints, undocumented). deps.dev doesn't document rate limits. The tracker is genuinely useful only against gh's documented 5000/hr authenticated quota.

```bash
# Canonical helper: gh_with_rate_tracker
# Reads X-RateLimit-Remaining from response; if low, sleeps until X-RateLimit-Reset.
# Falls back to H8 reactive backoff on actual 429.
gh_with_rate_tracker() {
  local CMD="$1"
  # gh api supports --include to surface response headers in stdout
  # For wrapper commands (gh repo view, gh run list), capture via -H "Accept: application/vnd.github+json" + extract from gh's debug log

  # Simpler: every N calls, do a quota check via `gh api rate_limit`
  if [ -z "$GH_RATE_LAST_CHECK" ] || [ $(($(date +%s) - GH_RATE_LAST_CHECK)) -gt 60 ]; then
    QUOTA_JSON=$(timeout 10 gh api rate_limit 2>/dev/null)
    if [ -n "$QUOTA_JSON" ]; then
      GH_RATE_REMAINING=$(echo "$QUOTA_JSON" | jq -r '.resources.core.remaining // 5000')
      GH_RATE_RESET=$(echo "$QUOTA_JSON" | jq -r '.resources.core.reset // 0')
      GH_RATE_LAST_CHECK=$(date +%s)
    fi
  fi

  # If budget low (< 100 calls remaining and reset > 30s away), sleep proactively
  if [ -n "$GH_RATE_REMAINING" ] && [ "$GH_RATE_REMAINING" -lt 100 ]; then
    NOW=$(date +%s)
    WAIT=$((GH_RATE_RESET - NOW))
    if [ $WAIT -gt 0 ] && [ $WAIT -lt 600 ]; then
      echo "WARN: GitHub API quota low ($GH_RATE_REMAINING remaining); sleeping ${WAIT}s until reset" >&2
      sleep $WAIT
      # Refresh after sleep
      GH_RATE_LAST_CHECK=""
    elif [ $WAIT -ge 600 ]; then
      echo "WARN: GitHub API quota low ($GH_RATE_REMAINING remaining); reset in $((WAIT/60))min." >&2
      echo "      Falling through to H8 reactive backoff if 429 hits." >&2
    fi
  fi

  # Now run the actual command with H8 reactive backoff as the safety net
  npm_with_h8_backoff "$CMD"
}
```

Apply in `/status` Phase 2 around the per-repo `gh repo view` / `gh run list` fan-out — the highest-impact case (30+ repo portfolios). Don't apply at every single gh call site; the helper invokes the rate check at a 60s interval, so wrapping the fan-out loop is enough.

### −1.10 SSL/TLS error detection + remediation (Tier-3 B, NEW v0.12.0)

External calls to npm registry, deps.dev, GitHub all use HTTPS. SSL/TLS failures present a confusing variety of error messages — corporate proxies, transient handshake failures, expired CA bundles, custom certs.

Detection patterns + remediation:

```bash
# After any external HTTPS call (curl, git push, npm publish/view), if the call failed,
# scan stderr for SSL-related patterns:
ssl_remediation_if_applicable() {
  local STDERR="$1"
  if echo "$STDERR" | grep -qE "SSL_ERROR|SSL_connect|certificate verify|cert.*expired|unable to get local issuer|self.signed|TLSV1_ALERT|ssl3_get_server"; then
    echo "ERROR: SSL/TLS error from external call."
    echo "       This is most often one of:"
    echo
    echo "  1. Transient network/handshake glitch — retry in 30s. (We've seen this on github.com pushes during peak hours.)"
    echo
    echo "  2. Corporate proxy with MITM cert chain. Configure your CA bundle:"
    echo "       npm config set cafile /path/to/corp-ca-bundle.pem"
    echo "       git config --global http.sslCAInfo /path/to/corp-ca-bundle.pem"
    echo "       export SSL_CERT_FILE=/path/to/corp-ca-bundle.pem  # for curl"
    echo
    echo "  3. System CA bundle outdated. Update OS certs:"
    echo "       macOS:  brew install ca-certificates"
    echo "       Linux:  update-ca-certificates  (Debian) / update-ca-trust  (RHEL)"
    echo
    echo "  4. Last resort (DEV ONLY, never in CI/prod): bypass cert check"
    echo "       curl --insecure ...    or    GIT_SSL_NO_VERIFY=true git push ..."
    echo "       This DOES NOT solve the underlying problem — only verifies the issue is cert-related."
    return 0   # signal caller that SSL remediation was surfaced
  fi
  return 1   # not an SSL error
}
```

The transient-retry case is highest-frequency for solo devs on home networks. The corporate-proxy case is highest-frequency in enterprise environments. Both surface clear remediation that cuts the user's debug time from minutes to seconds.

### −1.11 SIGINT signal handling (Tier-3 C, NEW v0.12.0)

`trap … EXIT` cleans up on graceful exit (success or error-with-`exit`). It does NOT fire on SIGKILL (covered by `−1.8` stale-lock cleanup). For SIGINT (Ctrl+C), we extend the trap to do **state-aware cleanup** so an interrupted skill doesn't leave half-done git/npm state that breaks the next attempt.

Convention: skills set state-flag environment variables as they progress through destructive multi-step phases. The SIGINT handler reads those flags and cleans accordingly.

```bash
# Set up SIGINT handler ONCE at the start of Phase −1 (this skill) or Phase A (sibling skills).
LOCAL_TAG_PENDING_PUSH=""   # set to the tag name (e.g., "v1.0.0") when local tag created but not yet pushed
STASH_PENDING=""            # set to the stash message when a stash is on the stack but not popped
PUBLISH_CONFIG_PENDING=""   # set to the package.json path when publishConfig.tag was modified but not yet committed

cleanup_on_signal() {
  echo
  echo "WARN: interrupted by user. Cleaning up in-flight state…"
  if [ -n "$LOCAL_TAG_PENDING_PUSH" ]; then
    echo "  - deleting local-only tag $LOCAL_TAG_PENDING_PUSH (was created locally but not yet pushed to origin)"
    git tag -d "$LOCAL_TAG_PENDING_PUSH" 2>/dev/null
  fi
  if [ -n "$STASH_PENDING" ]; then
    echo "  - dropping pending stash '$STASH_PENDING' (was created mid-skill; uncommitted changes preserved in working tree)"
    git stash drop 2>/dev/null
  fi
  if [ -n "$PUBLISH_CONFIG_PENDING" ]; then
    echo "  - reverting publishConfig.tag mutation in $PUBLISH_CONFIG_PENDING (was set mid-skill; not committed)"
    git checkout -- "$PUBLISH_CONFIG_PENDING" 2>/dev/null
  fi
  # The EXIT trap (lockfile cleanup) still fires after this on signal exit.
  echo "Cleanup complete. Safe to re-invoke the skill."
  exit 130   # standard exit code for "interrupted by SIGINT"
}
trap cleanup_on_signal INT
```

Sibling skills reference this pattern and set their own state flags at the corresponding mutation points. `/release`, `/prerelease`, `/hotfix` set `LOCAL_TAG_PENDING_PUSH` between `git tag` and `git push --tags`. `/deps` sets `STASH_PENDING` between `git stash` and `git stash pop`. `/hotfix` sets `PUBLISH_CONFIG_PENDING` between the package.json mutation and the commit.

This is **scoped, not exhaustive** — covers the high-value cleanups where mid-skill interruption commonly leaves bad state. Doesn't try to handle every possible interruption point (which would be a state machine).

## Phase 0 — Read prompt context

Scan the user's prompt for hints; pre-fill what's clear.

| Prompt mentions | Pre-fill |
|---|---|
| *"unpublish"*, *"delete published"*, *"remove from npm"* | `OPERATION = unpublish-version` (or `-all` if no version named) |
| *"rename X to Y"*, *"@oldscope/foo to @newscope/foo"* | `OPERATION = rename-redirect`, `OLD_NAME = @oldscope/foo`, `NEW_NAME = @newscope/foo` |
| *"wrong scope"*, *"I changed my mind"*, *"shipped under the wrong"* | `OPERATION = unpublish-all` (or rename if both old and new mentioned) |
| `@<scope>/<name>@<version>` | Set `OLD_NAME` + `VERSION` |
| `@<scope>/<name>` (no version) | Set `OLD_NAME` only; default to `unpublish-all` |

Examples:
- *"unpublish @gagel/foo@1.0.0 — typo in the scope"* → `OPERATION=unpublish-version, OLD_NAME=@gagel/foo, VERSION=1.0.0`. Skip prompts; jump to Phase A.
- *"rename @gagle/eutils to @ncbijs/eutils"* → `OPERATION=rename-redirect`. Skip prompts.
- *"I shipped @wrong/foo by mistake — wipe it"* → `OPERATION=unpublish-all, OLD_NAME=@wrong/foo`. Skip prompts.
- Bare *"unpublish"* → AskUserQuestion which package + which version.

## Phase 0.5 — Prompt-extraction validation (E1, E2, E3 canonical from v0.11.0)

After Phase 0 pre-fills slots from the user's prompt, validate every extracted value against a strict regex before passing to Phase A. Lenient extraction lets typos and malformed values flow through to destructive operations; strict-safety requires rejection at the boundary.

| Slot | Regex (anchored) | Why |
|---|---|---|
| **Semver version** (`VERSION`, `NEXT_VERSION`, range targets) | `^\d+\.\d+\.\d+(-[A-Za-z0-9.-]+)?(\+[A-Za-z0-9.-]+)?$` | npm semver: M.m.p with optional pre-release suffix and build metadata |
| **Pre-release identifier** (`IDENTIFIER`) | `^(alpha\|beta\|rc\|canary\|experimental\|next\|preview\|edge)$` | Whitelist — typos like "alpa", "betta" rejected |
| **Package scope** (`SCOPE`, when of the form `@<scope>`) | `^@[a-z0-9][a-z0-9._-]*$` | npm scope rules |
| **Package name** (`OLD_NAME`, `NEW_NAME`, `NAME`) | `^(@[a-z0-9][a-z0-9._-]*/)?[a-z0-9][a-z0-9._-]*$` | npm package name: optional scope + name, lowercase, no leading `.` or `_` |
| **dist-tag name** (`TAG`) | `^[a-z][a-z0-9-]*$` | npm dist-tag rules: lowercase letter start, alphanumerics + hyphens |
| **Semver range** (for `/deprecate` `RANGE`) | semver-parse via `semver.validRange()` | Defer to npm's bundled semver lib for range syntax |

**On validation failure**: STOP with diagnostic naming the slot + offending value + the expected format. Don't silently fall through to AskUserQuestion (which would just re-prompt for the same slot the user already gave).

```bash
# Example validation in /unpublish for VERSION extraction:
if [ -n "$VERSION" ] && ! echo "$VERSION" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+(-[A-Za-z0-9.-]+)?(\+[A-Za-z0-9.-]+)?$'; then
  echo "ERROR: extracted VERSION='$VERSION' is not a valid semver."
  echo "       Phase 0 saw it in your prompt but it doesn't match: M.m.p[-prerelease][+build]"
  echo "       Examples of valid: 1.0.0, 1.0.0-beta.1, 2.3.4+sha.abc123"
  echo "       Re-invoke with a corrected version, or omit and the skill will ask interactively."
  exit 1
fi
```

For `/unpublish`, the canonical extracted slots needing validation are: `OLD_NAME`, `NEW_NAME`, `VERSION`. For sibling skills, the relevant slots vary — they reference this canonical pattern and apply only the regexes for slots they actually extract.

## Phase 0.5b — Shell-safety hardening (Tier-4 #4, v0.13.0)

Phase 0.5 regex validation rejects malformed semver/identifier/scope-name. **It does NOT reject shell metacharacters that pass the regex but flow into shell interpolation downstream**. Example: a `MESSAGE` slot containing `; rm -rf $HOME` passes the regex (it's allowed as a deprecation message), but if any downstream skill body interpolates `$MESSAGE` into a shell command without quoting, the user just got their HOME wiped.

Mitigations applied at two boundaries:

### 0.5b.1 Reject high-risk shell metacharacters in extracted slots

For slots that flow into shell command construction (NAME, SCOPE, TAG, RANGE; not MESSAGE which is registry-bound text):

```bash
# Shell metacharacters that should never appear in name/tag/range slots:
# ;  &  |  `  $  (  )  <  >  newline  carriage-return
shell_safety_check() {
  local SLOT_NAME="$1"
  local SLOT_VALUE="$2"
  if echo "$SLOT_VALUE" | LC_ALL=C grep -qE '[;&|`$()<>]|[[:cntrl:]]'; then
    echo "ERROR: extracted $SLOT_NAME='$SLOT_VALUE' contains shell metacharacters."
    echo "       Phase 0.5 regex passed but shell-safety check rejected — these characters"
    echo "       would be interpreted by a downstream shell interpolation:"
    echo "         ;  &  |  \`  \$  (  )  <  >  control-chars"
    echo
    echo "       This is a strict-safety refusal. If you genuinely need one of these characters"
    echo "       in your value (rare, e.g., a registry URL with a port), invoke the underlying"
    echo "       command manually — solo-npm refuses to participate."
    exit 1
  fi
}

# Apply to every extracted slot that will flow into shell interpolation:
shell_safety_check "OLD_NAME" "$OLD_NAME"
shell_safety_check "NEW_NAME" "$NEW_NAME"
shell_safety_check "VERSION" "$VERSION"
```

### 0.5b.2 Always quote shell interpolation points in skill bodies

The convention at every shell-interpolation site:

```bash
# WRONG — shell interprets metacharacters in $NAME:
npm view $NAME --json

# CORRECT — double quotes prevent shell expansion:
npm view "$NAME" --json

# Even more conservative for npm calls — pass the value via stdin or env var to avoid
# any argument-splitting ambiguity:
NPM_NAME="$NAME" timeout 30 sh -c 'npm view "$NPM_NAME" --json' 2>/dev/null
```

The H1/H4/H8 wrappers from earlier sections all already use double-quoted interpolation. Sibling skills that construct shell commands need to maintain this convention; the shell-safety check at extraction is the second line of defense.

For MESSAGE / commit-message / deprecation-message slots that contain user free-text, **shell interpolation must always be via single-quoted strings** (or via env-var pass-through). E.g.:

```bash
# WRONG: $MSG could break out of the double-quoted context:
git commit -m "chore: release $MSG"

# CORRECT: pass via heredoc or single-quoted:
git commit -m "chore: release v${NEXT_VERSION}"   # don't put user free-text in the commit message at all
# OR via env var:
COMMIT_MSG="$MSG" git commit -F <(echo "$COMMIT_MSG")
```

Shell-safety hardening applies in `/unpublish` (canonical here), `/release` Phase 0.5b, `/prerelease` Phase 0.5b, `/hotfix` Phase 0.5b, `/dist-tag` Phase 0.5b, `/deprecate` Phase 0.5b. Each references this canonical block + applies `shell_safety_check` to its skill-specific slots.

## Phase A — Pre-flight + per-version eligibility

### A.1 Auth check

Run `npm whoami`. If exit code != 0 (not authenticated), surface foolproof handoff (matches the pattern in `/trust`, `/dist-tag`, `/deprecate`, `/owner`):

> Run `npm login` to authenticate with npm:
> 1. Type `npm login` and press Enter.
> 2. It prints a URL — open it in your browser.
> 3. Sign in to npmjs.com.
> 4. Approve the login via your authenticator app (web 2FA).
> 5. Return here when the terminal shows your username.

Then `AskUserQuestion`:
```
Header:   "Authenticated"
Question: "Done with `npm login`?"
Options:
  - Yes, continue (Recommended) — re-runs `npm whoami` to verify
  - Abort
```

### A.2 Per-version eligibility check

For each affected version (single version for `unpublish-version`; all published versions for `unpublish-all`; old-name versions for `rename-redirect`):

```bash
# 1. Publish age
PUBLISHED_AT=$(npm view "<NAME>@<VERSION>" time."<VERSION>")
HOURS_SINCE_PUBLISH=$(( (now - publishedAt) / 3600 ))

# 2. Dependents (via deps.dev — free, no auth, integrated in v0.9.0)
#    Capture HTTP status separately to differentiate "not indexed yet" (404)
#    from "API down" (5xx/timeout) — see §A.3 below.
HTTP_STATUS=$(curl -s --max-time 10 --connect-timeout 5 \
  -o /tmp/depsdev.json -w "%{http_code}" \
  "https://api.deps.dev/v3/systems/npm/packages/<NAME>/versions/<VERSION>:dependents")
# --max-time 10: total timeout incl. connect + transfer
# --connect-timeout 5: stop waiting for TCP handshake after 5s
# On timeout, curl exits with code 28 → HTTP_STATUS will be empty → routes into the
# 5xx/timeout STOP branch in the table below (correct strict-safety behavior).
case "$HTTP_STATUS" in
  200) DEPENDENTS=$(jq '.dependentCount // 0' /tmp/depsdev.json) ;;
  404) DEPENDENTS=0; DEPSDEV_NOTE="not-indexed" ;;
  *)   DEPSDEV_NOTE="api-error:$HTTP_STATUS" ;;
esac

# 3. Weekly downloads
DOWNLOADS=$(curl -s --max-time 10 --connect-timeout 5 "https://api.npmjs.org/downloads/point/last-week/<NAME>" | jq '.downloads // 0')
# On timeout/failure: curl exits non-zero, jq receives empty input, returns 0.
# Treating as 0 dl/wk on transient failure is conservative-safe — it makes the version
# MORE eligible for unpublish, not less. The user can manually verify via the npmjs.com
# download stats if the eligibility analysis surprises them.

# 4. Ownership (use --json to avoid header-line off-by-one of `wc -l`)
OWNER_COUNT=$(npm owner ls --json "<NAME>" | jq 'length')
```

Classify each version:

| Classification | Condition | Allowed? |
|---|---|---|
| `eligible-72h` | `HOURS_SINCE_PUBLISH < 72` AND `DEPENDENTS == 0` | ✓ Allowed |
| `eligible-criteria` | `HOURS_SINCE_PUBLISH >= 72` AND `DEPENDENTS == 0` AND `DOWNLOADS < 300` AND `OWNER_COUNT == 1` | ✓ Allowed |
| `blocked-dependents` | `DEPENDENTS > 0` | ❌ HARD STOP — no override |
| `blocked-criteria` | `HOURS_SINCE_PUBLISH >= 72` AND criteria not met | ❌ HARD STOP — no override |

### A.3 deps.dev response handling (4-way)

deps.dev's "no data" can mean two different things — the wrong-name fast-cleanup case is exactly when 404 (not indexed yet) fires, so we must NOT collapse it into "API down":

| HTTP response | Meaning | Treatment |
|---|---|---|
| `200 OK` with `dependentCount` field | Indexed; value is authoritative | Use the value |
| `404 Not Found` for the package | Not indexed yet (typically 5–60min after first publish) | Treat as `dependentCount = 0` AND surface a non-fatal note: *"Note: deps.dev has not yet indexed `<pkg>` (typical 5–60min lag after first publish). Treating as 0 dependents — safe assumption for a package no consumer has had time to install yet."* Continue. |
| `5xx`, `429`, timeout, network error | API genuinely down | **STOP** with: *"Cannot verify dependents — deps.dev API unreachable (HTTP `<status>`). Refusing unpublish without verification (would risk breaking consumers). Try again later, or unpublish manually after verifying via `npm view <pkg> dependents`."* |
| Unexpected schema (no `dependentCount`, malformed JSON) | API contract drift | **STOP** with: *"deps.dev returned unexpected response shape; refusing to proceed without verification. File an issue if this persists."* |

The 404-treatment is safe by definition: a package no consumer has had time to install cannot have dependents. The 5xx/timeout-treatment preserves strict safety: API down → cannot verify → STOP. **The user has no override flag** for the strict-stop cases — bypassing dependent verification would defeat the whole skill. Manual `npm unpublish` is the explicit escape.

### A.4 Render eligibility analysis (always shown)

Before any gate fires. The `Status` column **must name the failing condition with its actual value** when blocked — generic *"blocked (criteria not met)"* hides which knob is the problem and forces the user to dig.

```
Pre-unpublish analysis for @wrong/foo:

  Version    Published         Dependents   Downloads/wk   Owners   Status
  v1.0.0     18h ago            0           12             1        ✓ eligible (within 72h)
  v1.0.1      4h ago            0            0             1        ✓ eligible (within 72h)
  v0.9.5     35d ago            2            8             1        ❌ blocked: dependents (2)
  v0.8.0     90d ago            0          350             1        ❌ blocked: 350 dl/wk (limit 300)
  v0.7.0    120d ago            0           50             2        ❌ blocked: 2 owners (limit 1)

  2 of 5 versions eligible for unpublish.
  3 versions BLOCKED (cannot bypass — would break consumers OR violate npm policy).

  ⚠ npm STRONGLY RECOMMENDS deprecate over unpublish.
     Deprecation preserves install-ability for consumers who pinned this
     version in their lockfile. Unpublish breaks them.
```

Naming rules for the Status column when blocked:

| Failing condition | Status text |
|---|---|
| `DEPENDENTS > 0` (any window) | `blocked: dependents (<N>)` |
| Past 72h AND `DOWNLOADS >= 300` | `blocked: <N> dl/wk (limit 300)` |
| Past 72h AND `OWNER_COUNT > 1` | `blocked: <N> owners (limit 1)` |
| Past 72h AND multiple criteria fail | List all (comma-separated, e.g., `blocked: dependents (2), 350 dl/wk`) |
| deps.dev returned 404 (not indexed yet) | Suffix `Status` value with `(deps.dev not yet indexed)` |

Always show this regardless of operation. Eligibility data drives the gate.

### A.5 NEW_NAME validation (rename-redirect only)

Skip if `OPERATION != rename-redirect`.

For renames, validate that the new name is either available (404) or already owned by the current user. Otherwise the deprecation message *"Renamed to `<NEW_NAME>`"* would point consumers at a name the user can't actually publish to.

```bash
# 1. Check if NEW_NAME exists on the registry
NEW_VIEW_STATUS=$(npm view "<NEW_NAME>" name --json 2>&1; echo "EXIT=$?")
```

| State | Detection | Treatment |
|---|---|---|
| NEW_NAME unregistered (404) | `npm view` exits non-zero with `E404` in stderr | ✓ Fine — user will publish under new name later via `/init` or `/release` |
| NEW_NAME registered AND current user owns it | `npm owner ls --json <NEW_NAME>` parsed; `npm whoami` value present in array | ✓ Fine — user can publish further versions to the new name |
| NEW_NAME registered AND owned by someone else | Owner array does not contain `npm whoami` | ❌ **HARD STOP** — *"`<NEW_NAME>` is registered on npm and is owned by `<other-user>`. The redirect message would point consumers at a package you can't publish to. Pick a different new name, or contact the current owner about a transfer."* |
| `npm view` succeeds but registry response is malformed | jq parse fails | ❌ **HARD STOP** — *"Cannot verify ownership of `<NEW_NAME>` — registry returned unexpected shape. Refusing to proceed."* |

The check is read-only and cheap; runs once per rename invocation.

## Phase B — Two-gate confirmation (destructive ops)

### Gate 1 — choose path (deprecate vs unpublish)

```
Header:   "Recovery path"
Question: "How to proceed with @wrong/foo?"
Options:
  - Deprecate eligible versions (Recommended) — npm's preferred path; consumers who pinned this version keep working but see a WARN message
  - Unpublish eligible versions (DESTRUCTIVE) — removes the artifact; consumers with this version pinned in lockfiles get broken installs
  - Abort
```

If user picks **Deprecate**: chain to `/solo-npm:deprecate` with the eligible version range pre-filled.

**Chain-failure recovery (H6 pattern)**: capture the chained skill's exit signal. If `/solo-npm:deprecate` STOPs internally (empty matching range, message length > 1024, OTP needed and user couldn't provide, etc.), surface the child diagnostic verbatim in `/unpublish` context and present:

```
Header:   "Deprecate chain stopped"
Question: "/solo-npm:deprecate stopped: <verbatim child error>. How to proceed?"
Options:
  - Adjust message/range and retry deprecate
  - Switch to Unpublish path (re-fire Gate 1 with the destructive option)
  - Abort
```

Don't silently swallow the child failure.

If user picks **Unpublish**: proceed to Gate 2.

### Gate 2 — confirm destructive op (only fires after Gate 1 unpublish)

```
Header:   "Confirm unpublish"
Question: "Unpublish @wrong/foo eligible versions? Read carefully:"
Options:
  - I understand and want to proceed — irreversible; same version cannot be republished; consumer lockfiles may break
  - Cancel — let me deprecate instead
```

Two gates make accidental destruction harder. Both must explicitly approve.

### Phase B for `rename-redirect`

For renames, the gate has different shape — orchestration not just destruction:

```
Header:   "Rename plan"
Question: "Rename @gagle/eutils → @ncbijs/eutils?"
Options:
  - Deprecate-only (Recommended) — old name keeps working with a redirect message; consumers can migrate at their pace
  - Deprecate + unpublish eligible versions — clears old where allowed (only for fresh-publish renames within 72h)
  - Abort
```

If `Deprecate + unpublish eligible`: also fires Gate 2 (destructive confirmation) before any unpublish.

The user does NOT publish the new name from this skill — they run `/release` (or `/init` for a fresh repo) under the new name separately. This skill only handles the old name.

### Phase B for `unpublish-all`

For full-package removal, Gate 1 surfaces ALL versions that would be removed:

```
Header:   "Full-package unpublish"
Question: "Remove @wrong/foo entirely (all 3 eligible versions; 1 blocked)?"
Options:
  - Deprecate the entire package (Recommended) — same end-user experience: consumers see deprecation; package disappears from search
  - Unpublish all eligible versions — `npm unpublish @wrong/foo --force`; blocked versions remain on registry
  - Abort
```

Then Gate 2 if user picks unpublish.

If ALL versions are blocked (every one has dependents): the unpublish path is greyed out / unavailable; only deprecate is offered.

## Phase C — Execute

### C.0 Pre-execute: auth-race re-check (H3 pattern) + acquire lock (H5 pattern)

After Gate 2 (the destructive confirmation), the user may have spent minutes deciding. The npm session may have expired during the gate. **Before the first destructive call**, re-verify auth and acquire a per-package lock:

```bash
# 1. Re-check auth (Phase A.1 ran minutes ago; session may be stale)
WHOAMI_NOW=$(npm whoami 2>&1)
if [ $? -ne 0 ] || [ "$WHOAMI_NOW" != "$WHOAMI_AT_PHASE_A" ]; then
  # Surface the same foolproof npm-login handoff from Phase A.1
  # AskUserQuestion: "Done with re-login?"; on Yes loop back to this check
  exit  # Don't proceed to destructive op with stale auth
fi

# 2. Acquire file lock with stale-PID auto-cleanup (per Phase −1.5)
#    Live lock holder → STOP. Dead lock holder → WARN + clean + proceed.
LOCK_FILE=".solo-npm/locks/$(echo "<NAME>" | sed 's|/|_|g').lock"
if [ -f "$LOCK_FILE" ]; then
  STALE_PID=$(cat "$LOCK_FILE" 2>/dev/null)
  if [ -n "$STALE_PID" ] && kill -0 "$STALE_PID" 2>/dev/null; then
    echo "ERROR: another /unpublish run holds $LOCK_FILE (PID $STALE_PID is alive)."
    echo "       Wait for it to finish, or kill PID $STALE_PID manually if it's hung."
    exit 1
  fi
  echo "WARN: removing stale lockfile $LOCK_FILE (prior holder PID $STALE_PID is no longer alive)"
  rm -f "$LOCK_FILE"
fi
mkdir -p .solo-npm/locks
echo $$ > "$LOCK_FILE"
trap 'rm -f "$LOCK_FILE"' EXIT
```

If either check fails, halt before any destructive npm call.

### C.1 Per-version unpublish (for `unpublish-version` and `rename-redirect`)

For each eligible version, with **200ms inter-call backoff**:

```bash
npm unpublish "<NAME>@<VERSION>"
```

Halt on first failure. Surface npm error verbatim. Common failures:

| Error | Meaning | Recovery |
|---|---|---|
| 403 | Auth lapsed during run | Re-authenticate (re-run C.0 auth check); resume |
| 404 | Version doesn't exist (already unpublished?) | Skip; continue to next version |
| 409 | Conflict — version became dependent during the run | Re-classify; this version flips to `blocked-dependents` |
| **EOTP / `OTP required`** in stderr (H1 pattern) | 2FA-on-writes is enabled; npm waits for OTP at stdin; skill cannot type into stdin | Surface manual handoff: *"npm requires an OTP for this operation. Run `npm unpublish <pkg>@<version> --otp=<your-OTP-code>` manually outside the skill, then re-invoke `/solo-npm:unpublish` to clean up state and run remaining versions."* |
| Network error | Transient | Retry once with 1s delay; if persists, halt with clear message |

### C.2 Full-package unpublish (for `unpublish-all`)

```bash
npm unpublish "<NAME>" --force
```

`--force` removes all versions in one call. Same failure handling as C.1, including the EOTP/OTP-required handoff (H1 pattern). If some versions are blocked, the call may partially fail — re-classify after the fact.

### C.3 Rename-redirect deprecation (for `rename-redirect`)

If user picked Deprecate-only OR Deprecate+unpublish, run the deprecation step FIRST so consumers always have a redirect message even if subsequent unpublish fails:

```bash
# Deprecate the entire range with a redirect message:
npm deprecate "<OLD_NAME>" "Renamed to <NEW_NAME>. Install: npm i <NEW_NAME>"
```

The wildcard form (`<OLD_NAME>` without `@<version>`) deprecates ALL existing versions. Future versions of the same name (if accidentally published) won't be auto-deprecated; the user should not publish to the old name again.

**EOTP handling (H1 pattern)**: if the deprecate call hangs awaiting OTP or stderr contains `EOTP`/`OTP required`, surface the same manual-handoff pattern as C.1, adapted: *"Run `npm deprecate <OLD_NAME> '<msg>' --otp=<code>` manually, then re-invoke `/solo-npm:unpublish` to continue."*

If Deprecate+unpublish: after deprecate succeeds, run C.1 per eligible version.

## Phase D — Post-execution

### D.1 Verify (with propagation-lag retry — H4 pattern)

npm CDN can take 30–60s to reflect an unpublish. Don't fail verify on the first 200; **3 attempts × 5s sleep** before treating as inconsistent:

```bash
verify_gone() {  # args: $1 = full coordinate (e.g., "@wrong/foo@1.0.0" or "@wrong/foo")
  for i in 1 2 3; do
    OUT=$(npm view "$1" 2>&1)
    if echo "$OUT" | grep -qE "E404|404 Not Found|^npm ERR! 404"; then
      return 0  # successfully gone
    fi
    [ $i -lt 3 ] && sleep 5
  done
  return 1  # registry still showing data after 3 attempts
}
```

Apply per operation:

- **`unpublish-version`**: call `verify_gone "<NAME>@<VERSION>"` for each unpublished version.
- **`unpublish-all`**: call `verify_gone "<NAME>"` once.
- **`rename-redirect`**: also run `npm view "<OLD_NAME>@<v>" deprecated` for one of the deprecated versions; expect the redirect message.

If `verify_gone` returns 1 after 3 attempts, **don't HARD STOP** — the unpublish itself succeeded; this is verification lag. Surface a non-fatal note: *"Registry not yet reflecting unpublish for `<coord>` after 15s — npm CDN may take up to 5 minutes to propagate. Re-check later with `npm view <coord>`."* Continue to D.2.

### D.2 Cache cleanup (with corruption guard — H2 pattern)

For each unpublished `<NAME>@<VERSION>`, update `.solo-npm/state.json`. Wrap the read in try/catch so a malformed cache file doesn't crash the skill — cache hygiene is best-effort, not load-bearing:

```bash
node -e "
  const fs = require('fs');
  const path = '.solo-npm/state.json';
  let state;
  try {
    state = JSON.parse(fs.readFileSync(path, 'utf8'));
  } catch (e) {
    console.error('WARN: .solo-npm/state.json is malformed or missing; cache cleanup skipped. The unpublish itself succeeded — to regenerate the cache, remove the file and re-run any solo-npm skill.');
    process.exit(0);  // non-fatal
  }

  // Always: remove per-version pkgCheck entry
  if (state.pkgCheck?.lastSize?.['<NAME>@<VERSION>']) {
    delete state.pkgCheck.lastSize['<NAME>@<VERSION>'];
  }

  // unpublish-all only: remove package-level cache entries
  if ('<OPERATION>' === 'unpublish-all') {
    if (state.depsdev?.perPackage?.['<NAME>']) {
      delete state.depsdev.perPackage['<NAME>'];
    }
    if (state.trust?.['<NAME>']) {
      delete state.trust['<NAME>'];  // trust config no longer relevant
    }
    // pkgCheck.lastSize: remove all entries for this package
    if (state.pkgCheck?.lastSize) {
      for (const k of Object.keys(state.pkgCheck.lastSize)) {
        if (k.startsWith('<NAME>@')) delete state.pkgCheck.lastSize[k];
      }
    }
  }

  // Atomic write per Phase −1.4: write to .tmp then rename().
  // Non-atomic writeFileSync truncates first, so a process killed mid-write
  // would corrupt the cache. rename() is a single atomic inode swap on POSIX.
  const tmp = path + '.tmp';
  fs.writeFileSync(tmp, JSON.stringify(state, null, 2) + '\n');
  fs.renameSync(tmp, path);
"
```

**Audit cache is NOT cleared** — advisories are version-keyed AND keyed historical records of what was vulnerable; preserving them is intentional even after unpublish.

### D.3 Optional cleanup gate (git tag + GitHub Release)

After successful unpublish (and before final summary), offer to clean up the git tag and GitHub Release that `/release` Phase C.7 created. These are user-controlled artifacts; **don't auto-delete** — surface a gate.

Build the cleanup target list. For:
- `unpublish-version`: `tags = ["v<VERSION>"]`
- `unpublish-all`: `tags = ["v<v>" for each unpublished version]`
- `rename-redirect`: `tags = ["v<v>" for each version actually unpublished]` (NOT versions that were only deprecated)

If `tags` is empty (nothing to clean up — e.g., rename was deprecate-only), skip the gate and go straight to D.4.

Otherwise fire AskUserQuestion:

```
Header:   "Optional cleanup"
Question: "Also delete the local + remote git tag(s) and GitHub Release(s) for the unpublished version(s)? Tags involved: v1.0.0, v1.0.1"
Options:
  - Yes, both — delete git tags + GitHub Releases (Recommended for wrong-name/typo cleanup)
  - Just the git tags — keep the GitHub Releases as historical record
  - No, keep both — leave for manual cleanup or audit trail
  - Abort cleanup — same as "No"
```

**Execution per choice** (handle each tag/release independently — partial success is acceptable; missing `gh` should not block git-tag deletion):

```bash
# For "Yes, both" or "Just the git tags":
for v in <tag-list>; do
  # Local + remote tag deletion
  if git tag -d "$v" 2>&1; then
    git push origin ":refs/tags/$v" 2>&1 || echo "WARN: local tag $v deleted but remote push failed"
  else
    echo "WARN: git tag -d $v failed (already deleted? not present?); skipping remote"
  fi
done

# For "Yes, both" only — also delete GitHub Releases
if command -v gh >/dev/null 2>&1 && gh auth status >/dev/null 2>&1; then
  for v in <tag-list>; do
    gh release delete "$v" --yes 2>&1 || echo "WARN: gh release delete $v failed"
  done
else
  echo "NOTE: gh not installed or not authenticated; GitHub Release cleanup skipped."
  echo "      Install + authenticate from https://cli.github.com, then run:"
  echo "      gh release delete v1.0.0 --yes  # one per tag"
fi
```

**For "No / Abort"**: emit the manual-cleanup snippet so the user has it if they change their mind:

```
Skipping cleanup. To clean up later:

  git tag -d v1.0.0 && git push origin :refs/tags/v1.0.0
  gh release delete v1.0.0 --yes
  # repeat per version
```

Capture the result of each operation (deleted / kept / failed-with-reason) so D.4 can render it.

### D.4 Final summary + next-step suggestions

Layout the summary to surface (a) what was unpublished, (b) cleanup result, (c) next steps:

```
Unpublish complete.

  Unpublished from registry:
    @wrong/foo@1.0.0  (was 18h ago; 12 dl/wk)
    @wrong/foo@1.0.1  (was 4h ago; 0 dl/wk)
  Skipped (blocked):
    @wrong/foo@0.9.5  (blocked: dependents (2); cannot remove — left in place)

  Cleanup (per your choice "Yes, both"):
    git tag v1.0.0  ✓ deleted (local + remote)
    git tag v1.0.1  ✓ deleted (local + remote)
    GitHub Release v1.0.0  ✓ deleted
    GitHub Release v1.0.1  ⚠ skipped (gh not authenticated; install/login from https://cli.github.com)

Next steps:
  - Republish under the correct name when ready: /solo-npm:init (fresh repo) or /release (existing repo, after updating package.json#name)
  - The blocked version (v0.9.5) remains on npm. To handle it, /solo-npm:deprecate with a migration message.
  - Re-check unpublished versions later if registry verify lagged: npm view @wrong/foo@1.0.0
```

For `rename-redirect`:

```
Rename complete.

  @gagle/eutils:
    Deprecated all versions with redirect message: "Renamed to @ncbijs/eutils. Install: npm i @ncbijs/eutils"
    Unpublished v1.0.0, v1.0.1 (within 72h, no dependents)

  Cleanup (per your choice "Just the git tags"):
    git tag v1.0.0  ✓ deleted (local + remote)
    git tag v1.0.1  ✓ deleted (local + remote)
    GitHub Releases — kept (your choice)

Next steps:
  - Publish @ncbijs/eutils: run /solo-npm:init in the new repo, or update package.json#name and run /release
  - Consumers running `npm i @gagle/eutils` will see the redirect message and can migrate at their pace
```

## Hard stops (no override)

Two conditions cannot be bypassed by any flag, gate option, or chain:

| Condition | Message |
|---|---|
| **Any version has dependents (`DEPENDENTS > 0`)** | *"`<NAME>@<VERSION>` has `<N>` dependents (per deps.dev). Cannot unpublish without breaking them. Deprecate instead, or contact those dependents to migrate first. There is no `--force-with-dependents` override in this skill."* |
| **Post-72h, criteria not met (`DOWNLOADS >= 300` OR `OWNER_COUNT > 1`)** | *"`<NAME>@<VERSION>` is past the 72h window and doesn't meet npm's post-72h criteria (downloads `<N>`/wk, owners `<M>`). Per [npm policy](https://docs.npmjs.com/policies/unpublish/), this requires manual contact at `[email protected]`, OR deprecate via `/solo-npm:deprecate`."* |

These are not "warnings the user can override" — they're refusals. The skill exits cleanly without performing any unpublish. The user can still:
- Use `/solo-npm:deprecate` (which works regardless of dependents)
- Use manual `npm unpublish` outside the skill (the skill doesn't prevent the user; it just refuses to participate in unsafe operations)

## What this skill does NOT do

- **Republish under a new name** — `/release` (existing repo) or `/init` (fresh repo) handles the new name's publish flow.
- **Override npm policy** — there's no flag to bypass the 72h rule, dependent check, or download threshold.
- **Auto-chain from `/release`, `/audit`, or any other skill** — destructive enough that direct invocation only.
- **Cherry-pick versions to keep** — `unpublish-version` operates on the user-named version; `unpublish-all` operates on everything (with `--force`); rename operates on the entire old name. No middle ground.
- **Republish the unpublished version** — npm enforces version-number immutability after the 24/72h window; same number cannot be re-used. Bump to the next number.

## Composition

`/solo-npm:unpublish` is **not auto-chained** from any other skill. It's invoked directly when:

- The user types *"unpublish ..."*, *"delete published ..."*, *"rename ... to ..."*, *"I shipped under the wrong scope"*, etc.
- The user discovers a published-secrets emergency and wants to remove the artifact. Rotation is the load-bearing fix (non-optional); this skill only handles the artifact cleanup half.
- The user changes their mind about a package name post-publish.

Other skills that handle adjacent recovery scenarios:

- `/solo-npm:dist-tag repoint` — repoint `@latest` to a different version (NON-destructive; good for "make this version effectively retired without removing it")
- `/solo-npm:deprecate` — mark versions deprecated with a message (NON-destructive; npm's recommended path for almost everything)

`/solo-npm:unpublish` exists for the rare cases where deprecation isn't enough — wrong name, rename, or publishing-secrets emergency.

## Concurrency / locking

This skill acquires a file lock at `.solo-npm/locks/<sanitized-pkg-name>.lock` in Phase C.0 before any destructive npm call. If another `/unpublish` (or `/deprecate`/`/dist-tag`/`/owner` — they share the same lock prefix in v0.10.0+) is already running on the same package, this run halts cleanly with a diagnostic before doing anything destructive. The lock file holds the PID; if a previous run died without cleanup, the user can `rm .solo-npm/locks/<name>.lock` manually.

Single-user, single-session is the supported mode. If you're orchestrating multiple sessions across machines, serialize them yourself — the lock only protects against same-machine races.
