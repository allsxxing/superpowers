# Test Coverage Analysis

## Overview

This document identifies gaps in the current test suite and proposes concrete improvements. The project has two independently executable JavaScript files (`lib/skills-core.js`, `.opencode/plugins/superpowers.js`) and shell-based test runners under `tests/`.

---

## Current Test Coverage

### What Is Tested

| Component | Tests |
|---|---|
| `extractFrontmatter` | Happy path: name + description parsed correctly |
| `stripFrontmatter` | Happy path: frontmatter removed, body preserved |
| `findSkillsInDir` | Finds 3 skills including one nested directory |
| `resolveSkillPath` | Personal shadows superpowers; `superpowers:` prefix; unknown skill returns null |
| `checkForUpdates` | Error cases only: no remote, non-existent dir, non-git dir |
| Plugin loading | File existence, symlinks, JS syntax check |
| Skill triggering | 6 prompt variants for explicit requests; 6 for auto-triggering |
| `subagent-driven-development` | One full integration test (requires Claude Code) |

### What Is Not Tested

| Component | Gap |
|---|---|
| `checkForUpdates` — true/behind case | The function's only meaningful return value (`true`) is never exercised |
| `extractFrontmatter` — edge cases | Missing frontmatter, unclosed frontmatter, keys with hyphens |
| `stripFrontmatter` — edge cases | No frontmatter at all, `---` appearing in the body, empty input |
| `findSkillsInDir` — `maxDepth` boundary | Skills at exactly `maxDepth` vs. one level beyond are not verified |
| `resolveSkillPath` — null dirs | Behaviour when `personalDir` or `superpowersDir` is `null`/`undefined` |
| `normalizePath` in `superpowers.js` | Entire function untested: `~/` expansion, `null`, empty string, non-string |
| `extractAndStripFrontmatter` in `superpowers.js` | Separate reimplementation with different regex — no tests at all |
| `getBootstrapContent` in `superpowers.js` | Missing-skill path (`using-superpowers` absent) and happy path |
| All 14 skills' frontmatter | No test validates that every `SKILL.md` has `name` and `description` |
| `render-graphs.js` | Completely untested |
| `hooks/session-start.sh` | Not tested |
| CI pipeline | No GitHub Actions workflow; tests never run automatically |

---

## Proposed Improvements (Prioritised)

### Priority 1 — Fix the Function-Copy Anti-Pattern

**Problem:** `tests/opencode/test-skills-core.sh` inlines copies of every function it tests rather than importing from `lib/skills-core.js`. Changes to the real source file will not be caught.

**Fix:** Export the module as a CommonJS-compatible wrapper or write a thin Node.js test harness that imports the real ESM module:

```bash
# Instead of inlining, import the real module
result=$(node --input-type=module <<'EOF'
import { extractFrontmatter } from '/home/user/superpowers/lib/skills-core.js';
const r = extractFrontmatter('/path/to/SKILL.md');
console.log(JSON.stringify(r));
EOF
)
```

This ensures that tests always exercise the code that ships.

---

### Priority 2 — Test `checkForUpdates` True/Behind Path

**Problem:** The only branch of `checkForUpdates` that matters — returning `true` when the repo is behind remote — has zero test coverage. The current tests only exercise the `catch` block and the `false` path.

**Proposed test:**

```bash
# Set up a local remote two commits ahead, then verify the function returns true
mkdir -p "$TEST_HOME/remote-repo" && cd "$TEST_HOME/remote-repo"
git init --bare --quiet

mkdir -p "$TEST_HOME/local-repo" && cd "$TEST_HOME/local-repo"
git init --quiet
git remote add origin "$TEST_HOME/remote-repo"
echo "v1" > file.txt && git add . && git commit -m "initial" --quiet
git push origin HEAD:main --quiet

# Add a commit to the remote that the local clone doesn't have
cd "$TEST_HOME/remote-repo"
# push a second commit directly to bare repo via another clone...

result=$(node -e "... checkForUpdates('$TEST_HOME/local-repo') ...")
# expect: true
```

---

### Priority 3 — Add Edge-Case Tests for `extractFrontmatter`

The following inputs are untested and each exercises a different code path:

| Input | Expected behaviour |
|---|---|
| File with no `---` at all | Returns `{ name: '', description: '' }` |
| Opening `---` but no closing `---` | Returns `{ name: '', description: '' }` (loop exits at EOF with `inFrontmatter` still true) |
| Key with a hyphen (`multi-word: value`) | Returns `{ name: '', description: '' }` — regex `^(\w+):` does **not** match hyphens; worth documenting |
| File that does not exist | Returns `{ name: '', description: '' }` via the `catch` block |
| Frontmatter with only `description`, no `name` | Returns `{ name: '', description: '...' }` |

---

### Priority 4 — Add Edge-Case Tests for `stripFrontmatter`

| Input | Expected behaviour |
|---|---|
| Content with no frontmatter | Returns the full content unchanged |
| Empty string | Returns `''` |
| Frontmatter-only (no body) | Returns `''` |
| `---` appearing inside the body | Only the first pair of `---` delimiters is treated as frontmatter |

---

### Priority 5 — Test `normalizePath` in `superpowers.js`

This utility is called in the plugin's main entry point and handles user-supplied config paths. None of its branches are tested.

Suggested cases:

```javascript
normalizePath('~/projects/foo', '/home/user')  // → '/home/user/projects/foo'
normalizePath('~', '/home/user')               // → '/home/user'
normalizePath('/absolute/path', '/home/user')  // → '/absolute/path'
normalizePath(null, '/home/user')              // → null
normalizePath('', '/home/user')                // → null
normalizePath(42, '/home/user')                // → null (non-string)
```

Because `superpowers.js` uses ESM with no test harness, the simplest approach is a dedicated `tests/opencode/test-plugin-utils.sh` that runs the logic via `node --input-type=module`.

---

### Priority 6 — Validate All Skills Have Correct Frontmatter

**Problem:** There is no automated check that each of the 14 `SKILL.md` files contains a valid `name` and `description` field. A typo in any skill's frontmatter would silently produce empty entries in the skill list.

**Proposed test (`tests/opencode/test-skill-manifest.sh`):**

```bash
SKILLS_DIR="$(cd "$(dirname "$0")/../.." && pwd)/skills"

for skill_file in "$SKILLS_DIR"/*/SKILL.md; do
    skill_name=$(basename "$(dirname "$skill_file")")
    result=$(node -e "
      // parse frontmatter and print name/description
    " "$skill_file")

    name=$(echo "$result" | jq -r '.name')
    desc=$(echo "$result" | jq -r '.description')

    [ -n "$name" ] || echo "  [FAIL] $skill_name: missing 'name' in frontmatter"
    [ -n "$desc" ] || echo "  [FAIL] $skill_name: missing 'description' in frontmatter"
done
```

---

### Priority 7 — Test `findSkillsInDir` `maxDepth` Boundary

The default `maxDepth` is 3. The test only verifies one level of nesting. Add:

- A skill at depth 3 (should be found).
- A skill at depth 4 (should **not** be found with default `maxDepth`).
- A call with `maxDepth=1` to confirm the boundary is respected.

---

### Priority 8 — Test `resolveSkillPath` with Null/Missing Directories

The function accepts `null` for both `superpowersDir` and `personalDir`. Current tests always pass valid paths. Add:

```javascript
resolveSkillPath('any-skill', null, null)         // → null
resolveSkillPath('any-skill', superpowersDir, null) // skips personal lookup
resolveSkillPath('superpowers:skill', null, null)   // → null
```

---

### Priority 9 — Add a CI Workflow

None of the tests run automatically on pull requests. A minimal GitHub Actions workflow would catch regressions immediately:

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: bash tests/opencode/run-tests.sh
```

The existing unit tests (`test-plugin-loading.sh`, `test-skills-core.sh`) have no external dependencies and can run in CI without OpenCode or Claude Code installed.

---

### Priority 10 — Integration Tests for Remaining Skills

12 of 14 skills have no integration tests. The `skill-triggering/` tests only verify that a skill *triggers*, not that it *behaves correctly*. At minimum, the following high-value skills warrant integration tests modelled on the existing `test-subagent-driven-development-integration.sh`:

1. `systematic-debugging` — verify the debug loop, hypothesis formation, and resolution steps.
2. `test-driven-development` — verify red-green-refactor cycle produces passing tests.
3. `writing-plans` — verify a plan document is produced with the expected structure.
4. `verification-before-completion` — verify the checklist is populated and checked off.

---

## Summary Table

| Priority | Area | Effort | Value |
|---|---|---|---|
| 1 | Import real module instead of inlining copies | Low | High |
| 2 | `checkForUpdates` — behind/true path | Medium | High |
| 3 | `extractFrontmatter` edge cases | Low | High |
| 4 | `stripFrontmatter` edge cases | Low | Medium |
| 5 | `normalizePath` unit tests | Low | Medium |
| 6 | Skill frontmatter validation | Low | Medium |
| 7 | `findSkillsInDir` maxDepth boundary | Low | Medium |
| 8 | `resolveSkillPath` null directories | Low | Low |
| 9 | GitHub Actions CI workflow | Low | High |
| 10 | Integration tests for remaining skills | High | Medium |
