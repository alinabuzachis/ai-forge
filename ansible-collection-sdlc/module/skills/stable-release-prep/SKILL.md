---
name: stable-release-prep
description: >-
  Prepare Ansible collection release by creating prep branch from stable branch,
  updating galaxy.yml version, creating release summary fragment with proper
  backtick formatting for module names, and running antsibull-changelog release.
  Use after stable-release-analyze determines a version is needed.
---

# Skill: stable-release-prep

## Purpose

Prepare a release branch for an Ansible collection on a stable branch workflow.
Creates `prep_vX.Y.Z` branch, updates galaxy.yml, generates release summary fragment,
and runs antsibull-changelog to produce CHANGELOG.rst.

## When to Invoke

TRIGGER when:

- User asks to "prepare release" after analyzing stable branches
- After analysis identifies a pending release
- User requests to "create release branch" or "update version"
- Before running quality checks (lint/sanity) for a release

DO NOT TRIGGER when:

- Performing full release (use `stable-release` skill instead)
- Just analyzing (use `stable-release-analyze` skill instead)
- Reviewing a PR (use `pr-review` skill instead)

## Inputs

- `collection`: collection name (e.g., `amazon.aws`) or path. Defaults to current directory.
- `version`: target release version (e.g., `11.3.0`). Required.
- `branch`: stable branch to release from (e.g., `stable-11`). Required.

## Prerequisites

- Python 3.8+
- antsibull-changelog (pip install antsibull-changelog)
- ansible-core (provides ansible-doc for changelog)
- PyYAML (for fragment parsing)
- Git repository with configured `upstream` remote
- Clean working tree on stable branch

## Release Preparation Steps

### Step 1: Determine Collection Path and Sync

```bash
# Parse collection name
NAMESPACE="amazon"
NAME="aws"
COLLECTION_PATH="${ANSIBLE_COLLECTIONS_PATH:-$HOME/dev/collections/ansible_collections}/$NAMESPACE/$NAME"

# Navigate to collection
cd "$COLLECTION_PATH"

# Verify it's a collection
[ -f "galaxy.yml" ] || { echo "Error: Not an Ansible collection"; exit 2; }

# Sync with upstream
BRANCH="stable-11"
git checkout "$BRANCH"
git pull upstream "$BRANCH"
```

### Step 2: Validate Parameters

```bash
# Extract current version from galaxy.yml
CURRENT_VERSION=$(grep '^version:' galaxy.yml | awk '{print $2}' | tr -d '"')
echo "Current version: $CURRENT_VERSION"

# Target version
VERSION="11.3.0"
echo "Target version: $VERSION"

# Verify new version is higher than current
# (You should implement version comparison logic)
```

### Step 3: Create Prep Branch

```bash
# Create prep branch
git checkout -b "prep_v${VERSION}"

echo "✅ Branch created: prep_v${VERSION}"
```

### Step 4: Update galaxy.yml Version

```bash
# Update version field
sed -i '' "s/^version: .*/version: ${VERSION}/" galaxy.yml

# Verify change
git diff galaxy.yml

echo "✅ galaxy.yml updated: $CURRENT_VERSION → $VERSION"
```

### Step 5: Create Release Summary Fragment

**CRITICAL**: Module and plugin names MUST be wrapped in double backticks (\`\`name\`\`).

**CRITICAL**: Use `>` (folded block scalar), NOT `|` (literal block scalar).

```bash
# Create release summary fragment
cat > "changelogs/fragments/${VERSION}.yml" <<'EOF'
release_summary: >
  This minor release adds new features to the ``ec2_instance`` and
  ``rds_instance`` modules, including enhanced error handling with
  f-string parameter interpolation and improved ELB listener rule sorting.
EOF

echo "✅ Release summary created"
```

**Auto-generation logic**:
1. Parse existing fragments to understand changes
2. Run `git diff LAST_TAG..HEAD --stat` to detect modified files
3. Extract module names from `plugins/modules/` changes
4. Generate appropriate summary based on fragment types:
   - Only bugfixes → "This patch release includes bugfixes for..."
   - Minor changes → "This minor release adds new features to..."
   - Breaking changes → "This major release includes breaking changes to..."

### Step 6: Run antsibull-changelog Release

```bash
# Run antsibull-changelog
antsibull-changelog release --version "${VERSION}"

# Verify CHANGELOG.rst was updated
grep -q "v${VERSION}" CHANGELOG.rst && echo "✅ CHANGELOG.rst updated"

# Verify changelog.yaml was updated
grep -q "${VERSION}:" changelogs/changelog.yaml && echo "✅ changelog.yaml updated"

echo "✅ Changelog generated"
```

This will:
- Process all fragment YAML files in `changelogs/fragments/`
- Generate/update `CHANGELOG.rst`
- Update `changelogs/changelog.yaml`
- Delete processed fragment files (except VERSION.yml release summary)

### Step 7: Fix Common antsibull-changelog Bugs

**CRITICAL**: antsibull-changelog generates incorrect YAML indentation!

**Issue 1: changelog.yaml indentation errors**

ansible-lint will fail in CI with incorrect indentation (4 spaces instead of 6/8 for list items).

**You MUST read and validate the ENTIRE changelog.yaml file**:

```bash
# Validate with ansible-lint
ansible-lint --offline changelogs/changelog.yaml
```

**Common patterns to fix**:

```yaml
# WRONG Pattern 1: fragments list items at 4 spaces
    fragments:
    - 11.3.0.yml
    - other-fragment.yml

# CORRECT: fragments list items at 6 spaces
    fragments:
      - 11.3.0.yml
      - other-fragment.yml

# WRONG Pattern 2: minor_changes list items at 6 spaces
      minor_changes:
      - Added new feature

# CORRECT: minor_changes list items at 8 spaces
      minor_changes:
        - Added new feature

# WRONG Pattern 3: modules with name/namespace at 6 spaces
    modules:
      - description: Call a tool
      name: run_tool
      namespace: ''

# CORRECT: modules with name/namespace at 8 spaces
    modules:
      - description: Call a tool
        name: run_tool
        namespace: ''
```

**Issue 2: release_summary line too long**

ansible-lint enforces 160-character line length. If exceeded, break into multiple lines:

```yaml
# WRONG (line too long):
      release_summary: This minor release adds new features and enhancements to the amazon.aws collection...

# CORRECT (broken into multiple lines):
      release_summary: This minor release adds new features and enhancements to the
        amazon.aws collection, including enhanced error handling and improved sorting.
```

**Issue 3: .plugin-cache.yaml committed**

Remove if present:

```bash
if [ -f "changelogs/.plugin-cache.yaml" ]; then
  rm -f "changelogs/.plugin-cache.yaml"
  echo "Removed .plugin-cache.yaml (auto-generated, should not be committed)"
fi
```

### Step 8: Display Changes

```bash
# Show what was changed
git status --short
git diff --stat

echo "
Changed files:
  M galaxy.yml
  M CHANGELOG.rst
  M changelogs/changelog.yaml
  D changelogs/fragments/*.yml (processed fragments)
"
```

### Step 9: Next Steps

Present these next steps to the user:

1. Review changes: `git diff`
2. Run quality checks: `tox -m lint` and optionally `ansible-test sanity`
3. Commit and push:
   ```bash
   git add -A
   git commit -m "Release v${VERSION}"
   git push origin "prep_v${VERSION}"
   ```

## Release Summary Formatting Rules

Per cloud-content-handbook guidelines:

**✅ Correct:**

```yaml
release_summary: >
  This release includes updates to the ``my_module`` and ``other_module``
  modules for better ``aws_service`` integration.
```

**❌ Incorrect (missing backticks):**

```yaml
release_summary: >
  This release includes updates to the my_module and other_module
  modules for better aws_service integration.
```

**❌ Incorrect (using `|` instead of `>`):**

```yaml
release_summary: |
  This release includes updates to the ``my_module`` module.
```

**Why use `>` (folded block scalar)?**

- `>` collapses line breaks into spaces → clean single-line output in changelog.yaml
- `|` (literal block scalar) preserves blank lines → awkward formatting with extra newlines

## Troubleshooting

### "antsibull-changelog command not found"

```bash
pip install antsibull-changelog ansible-core
```

### "Version must be higher than current version"

Check current version:

```bash
grep version galaxy.yml
```

Ensure target version is higher: 11.3.0 > 11.2.0 ✓

### "No changelog fragments found"

Ensure fragments exist:

```bash
ls changelogs/fragments/*.yml
```

At least one non-.keep fragment must exist.

### "antsibull-changelog fails"

Verify changelogs/config.yaml exists:

```bash
cat changelogs/config.yaml
```

Validate fragment YAML syntax:

```bash
yamllint changelogs/fragments/*.yml
```

## Integration

This skill integrates with:
- `stable-release-analyze` - Analyzes pending releases (run before this)
- `docs-generate` - Generates documentation (run after this)
- `tox-lint` - Runs linters (run after this)
- `sanity` - Runs sanity tests (run after this)

## Exit Codes

- `0`: Release prep successful
- `1`: Preparation failed (git errors, validation failures)
- `2`: Invalid parameters or configuration
