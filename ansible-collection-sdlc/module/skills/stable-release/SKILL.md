---
name: stable-release
description: >-
  Guides the release of an Ansible collection following the stable-branch
  workflow. Analyzes pending releases on stable-X branches, determines
  appropriate SemVer version bumps from changelog fragments, generates
  changelogs, runs quality checks, and creates release PRs. Use when
  releasing from stable branches (stable-1, stable-2, etc.) rather than
  main branch.
---

# Skill: stable-release

## Purpose

Guide the release of an Ansible collection using the stable-branch workflow.
This skill is designed for collections that maintain multiple stable release
branches (stable-1, stable-2, etc.) following the Ansible Cloud Content
Handbook process.

## When to Invoke

TRIGGER when:

- A user asks to release from a stable branch (stable-1, stable-11, etc.)
- A user asks to check which stable branches need releases
- A user mentions "backport release" or "stable branch release"
- A user asks about releasing a patch or minor version on an older stable line

DO NOT TRIGGER when:

- Releasing from main/master branch (use `release` skill instead)
- Reviewing a PR (use `pr-review` skill instead)
- Running tests (use `run-tests` skill instead)

## Inputs

- `branch` (optional): stable branch to release from (e.g., `stable-11`). If not provided, analyzes all stable branches.
- `version` (optional): target release version (e.g., `11.3.0`). If not provided, automatically determined from changelog fragments.

## Prerequisites

- `antsibull-changelog` installed (`pip install antsibull-changelog`)
- `ansible-core` installed (provides ansible-doc for changelog generation)
- `tox` configured with lint environments
- `gh` CLI installed and authenticated
- Git remotes configured: `origin` (your fork), `upstream` (canonical repo)

## Human Confirmation Gates

**Do not proceed past a confirmation gate without explicit human approval.**
Present the relevant information and wait for the human to confirm
before continuing to the next step. Gates are marked with **CONFIRM** below.

## Release Steps

### Step 1: Analyze Stable Branches

**Purpose**: Determine which stable branches have unreleased commits and calculate appropriate versions.

**Commands**:

```bash
# Determine collection path
COLLECTION_PATH="${ANSIBLE_COLLECTIONS_PATH:-$HOME/dev/collections/ansible_collections}/NAMESPACE/NAME"
cd "$COLLECTION_PATH"

# Fetch all branches and tags
git fetch upstream --tags
git fetch upstream

# List all stable branches
git branch -r | grep "upstream/stable-" | sed 's/.*upstream\///'

# For each stable branch, analyze pending releases
# (Repeat for each stable-X branch)
BRANCH="stable-11"
git checkout "$BRANCH"
git pull upstream "$BRANCH"

# Find last tag
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "none")
echo "Last release: $LAST_TAG"

# Count commits since last tag
COMMIT_COUNT=$(git log ${LAST_TAG}..HEAD --oneline | wc -l | tr -d ' ')
echo "Commits since last tag: $COMMIT_COUNT"

# List changelog fragments
ls -1 changelogs/fragments/*.yml 2>/dev/null | grep -v ".keep" || echo "No fragments"

# Analyze fragment types to determine SemVer bump
# - breaking_changes, removed_features → MAJOR
# - major_changes, minor_changes, deprecated_features → MINOR  
# - bugfixes, security_fixes, trivial → PATCH
```

**Output Example**:

```
Stable branches found: 10

✅ stable-11: 11.2.0 → 11.3.0 (MINOR)
   Commits: 12, Fragments: 6
   - error-handler-format-strings.yml: minor_changes (impact: MINOR)
   - 2939-elb-rules-sorting.yml: bugfixes (impact: MINOR)
   - 4 trivial/bugfix fragments

✓ stable-10: Up to date (10.3.0)

⚠️ stable-9: 3 commits but no changelog fragments
```

**CONFIRM**: Which branch should we release? Present the analysis and wait for user to specify branch and confirm version.

### Step 2: Create Prep Branch

**Purpose**: Create release preparation branch and update galaxy.yml version.

**Commands**:

```bash
# Ensure on correct stable branch
git checkout stable-11
git pull upstream stable-11

# Create prep branch
VERSION="11.3.0"
git checkout -b "prep_v${VERSION}"

# Update galaxy.yml version
sed -i '' "s/^version: .*/version: ${VERSION}/" galaxy.yml

# Verify change
git diff galaxy.yml
```

### Step 3: Create Release Summary Fragment

**Purpose**: Create VERSION.yml fragment with release summary.

**CRITICAL**: Module and plugin names MUST be wrapped in double backticks (\`\`name\`\`).

**Commands**:

```bash
# Create release summary fragment
cat > "changelogs/fragments/${VERSION}.yml" <<'EOF'
release_summary: >
  This minor release adds new features to the ``ec2_instance`` and
  ``rds_instance`` modules, including support for new AWS regions
  and improved error handling.
EOF

# Customize the summary based on actual changes
# Review existing fragments to understand what changed
ls -1 changelogs/fragments/*.yml
cat changelogs/fragments/*.yml
```

**Formatting Rules**:

- Use `>` (folded block scalar), NOT `|` (literal block scalar)
- Wrap all module/plugin names in double backticks: \`\`module_name\`\`
- Keep lines under 160 characters for ansible-lint compliance
- Match tone to fragment type:
  - PATCH: "This patch release includes bugfixes for..."
  - MINOR: "This minor release adds new features to..."
  - MAJOR: "This major release includes breaking changes to..."

### Step 4: Generate Changelog

**Purpose**: Run antsibull-changelog to generate CHANGELOG.rst and update changelog.yaml.

**Commands**:

```bash
# Run antsibull-changelog
antsibull-changelog release --version "${VERSION}"

# Verify changelog was generated
grep -q "v${VERSION}" CHANGELOG.rst && echo "✅ CHANGELOG.rst updated"
grep -q "${VERSION}:" changelogs/changelog.yaml && echo "✅ changelog.yaml updated"

# Check which fragments were processed
git status --short changelogs/fragments/
```

**CRITICAL**: Fix antsibull-changelog indentation bugs:

antsibull-changelog has a known bug where it generates incorrect YAML indentation.
You MUST validate and fix the entire changelog.yaml file:

```bash
# Validate YAML with ansible-lint
ansible-lint --offline changelogs/changelog.yaml

# Common indentation issues to fix:
# 1. fragments: list items must be 6 spaces (not 4)
# 2. minor_changes: list items must be 8 spaces (not 6)
# 3. modules: name/namespace must be 8 spaces (aligned with description)
# 4. plugins: nested items must be 8 spaces, name/namespace 10 spaces
```

**Example fixes needed**:

```yaml
# WRONG:
    fragments:
    - 11.3.0.yml

# CORRECT:
    fragments:
      - 11.3.0.yml

# WRONG:
      minor_changes:
      - Added feature

# CORRECT:
      minor_changes:
        - Added feature
```

### Step 5: Run Quality Checks

**Purpose**: Validate code quality before committing.

**Commands**:

```bash
# Run tox linters
tox -m lint

# If tox doesn't include ansible-lint, run separately:
ansible-lint --offline

# Verify changelog.yaml passes lint
ansible-lint --offline changelogs/changelog.yaml
```

**CONFIRM**: Review lint results. If failures, fix them before proceeding.

### Step 6: Commit and Push

**Purpose**: Commit release changes and push to fork.

**Commands**:

```bash
# Stage release files
git add galaxy.yml CHANGELOG.rst changelogs/

# Verify no unwanted files (like .plugin-cache.yaml)
git status

# Create commit
git commit -m "$(cat <<EOF
Release v${VERSION}

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"

# Push to fork
git push origin "prep_v${VERSION}"
```

### Step 7: Create Pull Request

**Purpose**: Open PR to upstream stable branch.

**CONFIRM**: Ask user if they want to create PR now.

**Commands**:

```bash
# Create PR using gh CLI
gh pr create \
  --repo ansible-collections/COLLECTION_NAME \
  --base stable-11 \
  --head YOUR_USERNAME:prep_v${VERSION} \
  --title "Release v${VERSION}" \
  --body "$(cat <<'EOF'
## Release v11.3.0

This PR prepares the v11.3.0 release for the amazon.aws collection.

### Changes
- Updated galaxy.yml to v11.3.0
- Generated changelog from 6 fragments
- Updated CHANGELOG.rst

### Quality Checks
- ✅ Lint: All checks passed
- ✅ Sanity: Skipped (run in CI)

### Checklist
- [x] Version updated in galaxy.yml
- [x] Changelog generated with antsibull-changelog
- [x] YAML indentation fixed and validated
- [x] Lint checks passed

---
*Generated following the [Ansible Cloud Content Handbook](https://github.com/ansible-collections/cloud-content-handbook)*
EOF
)"
```

**Output**: PR URL for monitoring.

### Step 8: Post-Merge Tagging

**Purpose**: After PR is merged, tag the release.

**Commands** (run after PR merge):

```bash
# Sync with upstream
git checkout stable-11
git pull upstream stable-11

# Create and push tag
git tag "v${VERSION}"
git push upstream "v${VERSION}"

# Verify GitHub Actions publish to Galaxy
gh run list --workflow=release.yml --limit 5
```

## SemVer Calculation Rules

Based on Ansible changelog fragment types:

| Fragment Type | Impact | Version Bump | Example |
|--------------|--------|--------------|---------|
| `breaking_changes` | **MAJOR** | X.0.0 | 11.2.0 → 12.0.0 |
| `removed_features` | **MAJOR** | X.0.0 | 11.2.0 → 12.0.0 |
| `major_changes` | **MINOR** | x.Y.0 | 11.2.0 → 11.3.0 |
| `minor_changes` | **MINOR** | x.Y.0 | 11.2.0 → 11.3.0 |
| `deprecated_features` | **MINOR** | x.Y.0 | 11.2.0 → 11.3.0 |
| `bugfixes` | **PATCH** | x.y.Z | 11.2.0 → 11.2.1 |
| `security_fixes` | **PATCH** | x.y.Z | 11.2.0 → 11.2.1 |
| `trivial` | **PATCH** | x.y.Z | 11.2.0 → 11.2.1 |

**Rule**: The highest impact across all fragments determines the bump.

## Common Issues and Fixes

### Issue: changelog.yaml indentation errors

**Symptom**: ansible-lint fails in CI with YAML parsing errors even though it passed locally.

**Cause**: antsibull-changelog generates incorrect indentation (4 spaces instead of 6/8).

**Fix**: Read entire changelog.yaml and fix ALL list indentations:
- fragments: 6 spaces
- minor_changes/bugfixes: 8 spaces  
- modules: name/namespace at 8 spaces
- plugins: nested items at 8+ spaces

**Validation**:

```bash
ansible-lint --offline changelogs/changelog.yaml
# Should output: "Passed: 0 failure(s), 0 warning(s)"
```

### Issue: .plugin-cache.yaml committed

**Symptom**: CI complains about .plugin-cache.yaml in commit.

**Cause**: collection_prep auto-generates this file, should never be committed (it's in build_ignore).

**Fix**:

```bash
rm -f changelogs/.plugin-cache.yaml
git reset changelogs/.plugin-cache.yaml
```

### Issue: README.md links point to wrong branch

**Symptom**: Documentation links point to main instead of stable-11.

**Cause**: collection_prep defaults to main branch in generated links.

**Fix**: Update README.md links to match target stable branch:

```markdown
# Change:
/blob/main/docs/

# To:
/blob/stable-11/docs/
```

## Integration with Other Skills

- **commit** - Use after release to create conventional commits for post-release version bump
- **pr-review** - Use to review the release PR before merging
- **run-tests** - Optional, run integration tests before release (time-consuming)

## Configuration

Optional environment variables:

```bash
export GITHUB_USERNAME="your-username"
export ANSIBLE_COLLECTIONS_PATH="~/dev/collections/ansible_collections"
export SANITY_MODE="smart"  # For optional sanity testing
```

## Reference

- [Ansible Cloud Content Handbook - Stable Release Process](https://github.com/ansible-collections/cloud-content-handbook/blob/main/stable-release.md)
- [antsibull-changelog Documentation](https://github.com/ansible-community/antsibull-changelog)

## Exit Codes

When implemented as automation:

- `0`: Release workflow completed successfully
- `1`: Workflow failed at any step
- `2`: Configuration error or missing requirements
