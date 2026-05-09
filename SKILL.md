---
name: release-management
description: >
  Guides the full release workflow — pre-release verification, semantic version bump, CHANGELOG
  generation, release PR creation, and post-release tag/artifact checks. Use this skill whenever
  you are asked to "cut a release", "bump the version", "prepare a release PR", "write the
  CHANGELOG", or "execute the vX.Y.Z release". Also trigger it when you detect that release-related
  steps are needed during a sprint close-out or when CI/CD is passing and the branch is ready to
  ship. Prevents the common failure mode of improvising the multi-step sequence from memory each
  time.
---

# Release Management

This skill standardises every step of shipping a release: from pre-flight checks through
CHANGELOG authoring, version bumping, PR creation, tagging, and post-release verification.
Follow each phase in order. The checklist items are gates — if any step fails, stop and resolve
it before moving on.

---

## Phase 0 — Identify the Release

Before touching any files, establish the three facts that drive everything else:

1. **Target version** — provided by the caller (e.g., `v1.2.0`), or derive it via the semver
   rules in Phase 1 if not stated.
2. **Release branch** — typically `main`, `master`, or a `release/vX.Y.Z` branch. Confirm with
   the caller if ambiguous.
3. **Scope** — which packages/services are included. In a monorepo, ask which workspace; in a
   single-package repo, this is implicit.

---

## Phase 1 — Determine the Version Bump

If the caller has not specified the exact version, derive it from commits since the last tag using
Semantic Versioning (semver.org):

| Commit type / change | Bump |
|---|---|
| Breaking change (any scope) | **major** — X.0.0 |
| New feature, backwards-compatible | **minor** — x.Y.0 |
| Bug fix, dependency update, docs, refactor | **patch** — x.y.Z |
| Pre-release suffix (alpha/beta/rc) | append `-alpha.N`, `-beta.N`, `-rc.N` |

**How to read the commit log:**

```bash
git log $(git describe --tags --abbrev=0)..HEAD --oneline
```

Look for Conventional Commit prefixes (`feat:`, `fix:`, `feat!:`, `BREAKING CHANGE:`). If the
project does not use Conventional Commits, read commit messages for intent and apply the table
above by judgment.

**Verify the bumped version** against all package files listed in Phase 2 to confirm there are no
conflicts.

---

## Phase 2 — Pre-Release Checklist

Run through every item below. All must pass before writing the CHANGELOG or bumping files.

### 2a. Working tree is clean
```bash
git status --short
```
Expected: empty output. If not, either commit, stash, or explain why the uncommitted changes are
intentional before proceeding.

### 2b. On the correct branch
```bash
git branch --show-current
```
Confirm this matches the release branch identified in Phase 0.

### 2c. Up to date with remote
```bash
git fetch origin
git status -sb   # should show "ahead N" with no "behind"
```
If behind, rebase or merge the remote branch first.

### 2d. CI is passing
Check the CI status for the HEAD commit. In GitHub repos:
```bash
gh run list --branch $(git branch --show-current) --limit 5
```
All recent runs should be ✓ (`completed` / `success`). If any required checks are failing, fix
them before continuing. Do **not** release on a red build.

### 2e. Version consistency across package files
Search for the current version string across all files that track it. Common locations:

| File | How to update |
|---|---|
| `package.json` | `"version": "X.Y.Z"` |
| `package-lock.json` / `yarn.lock` | regenerate after bumping `package.json` |
| `pyproject.toml` | `version = "X.Y.Z"` |
| `Cargo.toml` | `version = "X.Y.Z"` |
| `build.gradle` / `pom.xml` | `<version>X.Y.Z</version>` |
| `VERSION` or `version.txt` | plain text |
| Source constants (e.g., `__version__ = "X.Y.Z"`) | code file |

Run a grep to find all occurrences before touching anything:
```bash
grep -rn "\"version\"" --include="*.json" .
grep -rn "^version" --include="*.toml" .
```

---

## Phase 3 — Update Version Files

Bump every file identified in Phase 2e to the new version. Prefer automated tooling when
available:

```bash
# Node.js (updates package.json + lockfile)
npm version <major|minor|patch> --no-git-tag-version

# Python (with bump2version or poetry)
bump2version <major|minor|patch>
poetry version <major|minor|patch>

# Rust
cargo release version <major|minor|patch>   # dry-run first
```

For files that must be updated manually, edit each one precisely — do not leave partial updates.

After updating all files, verify the grep from Phase 2e returns only the new version string.

---

## Phase 4 — Write the CHANGELOG

Use the **Keep a Changelog** format (keepachangelog.com). Add a new section at the top of
`CHANGELOG.md`, above the previous release entry.

### CHANGELOG section template

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Added
- <concise description of new feature> ([#PR](link))

### Changed
- <description of behavioural change>

### Fixed
- <description of bug fix> ([#PR](link))

### Removed
- <description of removed feature or API>

### Security
- <description of security fix>

### Breaking Changes  ← include only for major bumps
- <what changed and migration path>
```

**Rules:**
- Only include sections that have entries. Omit empty sections.
- Write entries from the user/consumer perspective, not the implementation perspective.
- Link to PRs or issues where available.
- For major releases, include a "Migration from vX" section after the standard sections if
  breaking changes require user action.
- Keep the `[Unreleased]` link at the top updated if the project uses one.

**Deriving entries from commits:**
```bash
git log $(git describe --tags --abbrev=0)..HEAD --pretty="format:- %s (%h)" | sort
```
Group and rewrite these into the changelog sections by type.

---

## Phase 5 — Commit the Release

Stage and commit all version-file and CHANGELOG changes in a single release commit:

```bash
git add CHANGELOG.md package.json package-lock.json   # add all changed version files
git commit -m "chore(release): v<X.Y.Z>"
```

Commit message convention: `chore(release): vX.Y.Z` (or `release: vX.Y.Z` if the project does
not use Conventional Commits).

Do **not** tag yet — the tag goes on the merge commit after the PR is merged.

---

## Phase 6 — Open the Release PR

Create a PR from the release branch targeting the mainline branch.

### PR title
```
Release v<X.Y.Z>
```

### PR body template
```markdown
## Release v<X.Y.Z>

### What's changed

<paste the CHANGELOG section for this version here>

### Pre-release checklist

- [ ] CI green on this branch
- [ ] Version bumped in all package files
- [ ] CHANGELOG updated
- [ ] No uncommitted changes

### Post-merge steps (for the merger)

1. Merge this PR
2. On the merged commit: `git tag -a v<X.Y.Z> -m "Release v<X.Y.Z>" && git push origin v<X.Y.Z>`
3. Verify the tag appears at: `gh release list` or the repository's Releases page
4. If a publish step exists (npm, PyPI, crates.io, Docker), run it or confirm the CI publish job
   fired automatically
```

Open the PR:
```bash
gh pr create \
  --title "Release v<X.Y.Z>" \
  --body "$(cat /tmp/release-pr-body.md)" \
  --base main
```

---

## Phase 7 — Post-Merge Verification

After the release PR is merged, confirm the following before closing the release task:

### 7a. Tag exists
```bash
git fetch --tags
git tag --list "v<X.Y.Z>"
```
If the tag is missing, create it manually on the merge commit:
```bash
git tag -a v<X.Y.Z> -m "Release v<X.Y.Z>" <merge-commit-sha>
git push origin v<X.Y.Z>
```

### 7b. GitHub Release created (if applicable)
```bash
gh release view v<X.Y.Z>
```
If missing, create it:
```bash
gh release create v<X.Y.Z> --title "v<X.Y.Z>" --notes "$(changelog-section-content)"
```

### 7c. Artifacts published
Check that any publish pipeline triggered and completed:
```bash
gh run list --workflow publish.yml --limit 5
```
For npm: `npm view <package-name> version`
For PyPI: `pip index versions <package-name>`
For Docker: `docker pull <image>:<tag>`

---

## Common Failure Modes

| Symptom | Likely cause | Fix |
|---|---|---|
| CI failing on release branch | Flaky test or merge conflict | Fix tests/conflicts before proceeding |
| Version mismatch between files | Missed a package file | Re-run Phase 2e grep, update all files |
| Tag already exists | Duplicate release attempt | Check if previous release was complete; delete stale tag only if no artifacts were published |
| CHANGELOG has no entries | No commits since last tag | Confirm you're on the right branch; check `git log` range |
| Publish job didn't fire | CI config targets tag pushes | Push the tag explicitly; verify the publish trigger |

---

## Reference: Commit Type → CHANGELOG Section Mapping

| Conventional Commit prefix | CHANGELOG section |
|---|---|
| `feat:` | Added |
| `fix:` | Fixed |
| `refactor:`, `perf:` | Changed |
| `docs:` | (omit unless user-facing) |
| `chore:`, `ci:`, `test:` | (omit unless notable) |
| `feat!:` / `BREAKING CHANGE:` | Breaking Changes + Changed |
| `security:` | Security |
| `remove:` / `feat!: remove` | Removed |
