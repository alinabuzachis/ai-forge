---
name: stable-release-analyze
description: >-
  Analyzes Ansible collection stable branches to determine pending releases
  and calculate appropriate SemVer versions. Checks for unreleased commits,
  analyzes changelog fragments, and recommends next version. Use when asked
  to check which stable branches need releases or what version to release.
---

# Skill: stable-release-analyze

## Purpose

Analyze Ansible collection stable branches for pending releases. Detects all
stable-X branches, counts unreleased commits, analyzes changelog fragments
to determine SemVer impact (MAJOR/MINOR/PATCH), and calculates the next
appropriate version number.

## When to Invoke

TRIGGER when:

- User asks to analyze pending releases
- User asks "which collections need releases?"
- User asks "what version should I release?"
- Beginning a release workflow (before release-prep)
- Planning release schedules

DO NOT TRIGGER when:

- Actually performing a release (use `stable-release-prep` skill instead)
- Reviewing a PR (use `pr-review` skill instead)
- Running tests (use `run-tests` skill instead)

## Inputs

- `collection` (optional): collection name (e.g., `amazon.aws`) or path. Defaults to current directory.

## Prerequisites

- Python 3.8+ with PyYAML library
- Git repository with configured `upstream` remote
- Stable branches following `stable-X` naming convention

## Analysis Steps

### Step 1: Setup and Navigate to Collection

```bash
# Determine collection path
NAMESPACE="amazon"
NAME="aws"
COLLECTION_PATH="${ANSIBLE_COLLECTIONS_PATH:-$HOME/dev/collections/ansible_collections}/$NAMESPACE/$NAME"

# Navigate to collection
cd "$COLLECTION_PATH"

# Verify it's a collection
[ -f "galaxy.yml" ] || { echo "Error: Not an Ansible collection"; exit 2; }
```

### Step 2: Fetch and List Stable Branches

```bash
# Fetch all branches and tags from upstream
git fetch upstream --tags
git fetch upstream

# List all stable branches
STABLE_BRANCHES=$(git branch -r | grep "upstream/stable-" | sed 's/.*upstream\///' | sort -V)

echo "Stable branches found:"
echo "$STABLE_BRANCHES"
```

### Step 3: Analyze Each Stable Branch

For each stable branch, perform this analysis:

```bash
BRANCH="stable-11"

# Checkout and sync
git checkout "$BRANCH"
git pull upstream "$BRANCH"

# Find last release tag
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null)
if [ -z "$LAST_TAG" ]; then
  echo "No tags found on $BRANCH"
  continue
fi

echo "Last release: $LAST_TAG"

# Extract version from galaxy.yml
CURRENT_VERSION=$(grep '^version:' galaxy.yml | awk '{print $2}' | tr -d '"')
echo "Current version: $CURRENT_VERSION"

# Count commits since last tag
COMMIT_COUNT=$(git log ${LAST_TAG}..HEAD --oneline | wc -l | tr -d ' ')
echo "Commits since last tag: $COMMIT_COUNT"

# Check for changelog fragments
FRAGMENT_COUNT=$(ls -1 changelogs/fragments/*.yml 2>/dev/null | grep -v ".keep" | wc -l | tr -d ' ')
echo "Changelog fragments: $FRAGMENT_COUNT"

# If no commits or no fragments, branch is up to date
if [ "$COMMIT_COUNT" -eq 0 ]; then
  echo "✓ $BRANCH: Up to date ($CURRENT_VERSION)"
  continue
fi

if [ "$FRAGMENT_COUNT" -eq 0 ]; then
  echo "⚠️ $BRANCH: $COMMIT_COUNT commits but no changelog fragments"
  continue
fi
```

### Step 4: Analyze Fragment Types for SemVer Impact

```bash
# List all fragment files
FRAGMENTS=$(ls -1 changelogs/fragments/*.yml 2>/dev/null | grep -v ".keep")

# Initialize impact level (PATCH < MINOR < MAJOR)
IMPACT="PATCH"

# Check each fragment for type
for FRAGMENT in $FRAGMENTS; do
  # Read fragment content and check for keys
  if grep -q "breaking_changes:" "$FRAGMENT" || grep -q "removed_features:" "$FRAGMENT"; then
    IMPACT="MAJOR"
    break
  elif grep -q "major_changes:" "$FRAGMENT" || grep -q "minor_changes:" "$FRAGMENT" || grep -q "deprecated_features:" "$FRAGMENT"; then
    IMPACT="MINOR"
  fi
  # bugfixes, security_fixes, trivial → PATCH (default)
done

echo "Fragment impact: $IMPACT"
```

### Step 5: Calculate Next Version

```bash
# Parse current version
IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT_VERSION"

# Calculate next version based on impact
if [ "$IMPACT" = "MAJOR" ]; then
  NEXT_VERSION="$((MAJOR + 1)).0.0"
elif [ "$IMPACT" = "MINOR" ]; then
  NEXT_VERSION="${MAJOR}.$((MINOR + 1)).0"
else  # PATCH
  NEXT_VERSION="${MAJOR}.${MINOR}.$((PATCH + 1))"
fi

echo "✅ $BRANCH: $CURRENT_VERSION → $NEXT_VERSION ($IMPACT)"
echo "   Commits: $COMMIT_COUNT, Fragments: $FRAGMENT_COUNT"
```

### Step 6: Display Fragment Details

```bash
# List fragment files and their types
echo "   Fragment details:"
for FRAGMENT in $FRAGMENTS; do
  BASENAME=$(basename "$FRAGMENT")
  # Determine fragment type
  if grep -q "breaking_changes:" "$FRAGMENT"; then
    TYPE="breaking_changes (MAJOR)"
  elif grep -q "removed_features:" "$FRAGMENT"; then
    TYPE="removed_features (MAJOR)"
  elif grep -q "major_changes:" "$FRAGMENT"; then
    TYPE="major_changes (MINOR)"
  elif grep -q "minor_changes:" "$FRAGMENT"; then
    TYPE="minor_changes (MINOR)"
  elif grep -q "deprecated_features:" "$FRAGMENT"; then
    TYPE="deprecated_features (MINOR)"
  elif grep -q "bugfixes:" "$FRAGMENT"; then
    TYPE="bugfixes (PATCH)"
  elif grep -q "security_fixes:" "$FRAGMENT"; then
    TYPE="security_fixes (PATCH)"
  elif grep -q "trivial:" "$FRAGMENT"; then
    TYPE="trivial (PATCH)"
  else
    TYPE="unknown"
  fi
  echo "     - $BASENAME: $TYPE"
done
```

## Output Example

```
Collection: amazon.aws
Current version: 11.2.0

Stable branches found: 10

✓ stable-2: Up to date (2.3.0)

✓ stable-3: Up to date (3.5.1)

⚠️ stable-6: 1 commits but no changelog fragments

✅ stable-11: 11.2.0 → 11.3.0 (MINOR)
   Commits: 12, Fragments: 6
   Fragment details:
     - error-handler-format-strings.yml: minor_changes (MINOR)
     - 2939-elb-rules-sorting.yml: bugfixes (PATCH)
     - ansible-test-gh-action-migration.yml: trivial (PATCH)
     - elb_logging_service_principal.yml: trivial (PATCH)
     - reliability-duplicate-assignments.yml: bugfixes (PATCH)
     - 1915-cloudfront_distribution_info-TypeError.yml: bugfixes (PATCH)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary: 1 release(s) needed
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

## Troubleshooting

### "Remote 'upstream' not found"

```bash
cd "$COLLECTION_PATH"
git remote add upstream https://github.com/ansible-collections/COLLECTION.git
git fetch upstream --tags
```

### "No stable branches found"

Verify the collection uses `stable-X` branch naming:

```bash
git branch -a | grep stable
```

### "Cannot determine collection path"

Ensure you're either:
1. In a collection directory with `galaxy.yml`, OR
2. Providing a valid collection name like `amazon.aws`

## Integration

This skill integrates with:
- `stable-release-prep` - Prepares release after analysis determines version
- `stable-release` - Full release orchestrator (uses this for analysis step)

## Exit Codes

- `0`: Analysis complete, results displayed
- `1`: Analysis failed (git errors, missing dependencies)
- `2`: Invalid collection or configuration
