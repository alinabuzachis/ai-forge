---
name: tox-lint
description: >-
  Run all configured tox linters on an Ansible collection or Python project.
  Executes ansible-lint, black, isort, flake8, pylint, flynt, and ruff to ensure
  code quality and style consistency. Use before commits or as part of release workflow.
---

# Skill: tox-lint

## Purpose

Execute tox linting environments to ensure code quality and style consistency.
Runs multiple linters in parallel including black, isort, flake8, pylint,
ansible-lint, flynt, and ruff.

## When to Invoke

TRIGGER when:

- User asks to "run lint" or "check code quality"
- Before committing code changes
- As part of release preparation workflow (after `stable-release-prep`)
- When contributing to Ansible collections
- To validate code meets team standards

DO NOT TRIGGER when:

- Performing full release (use `stable-release` skill instead)
- Running only tests (use `run-tests` skill instead)

## Prerequisites

- Python 3.8+
- tox (pip install tox)
- tox.ini with lint environments configured

## Linting Steps

### Step 1: Verify tox.ini Configuration

```bash
cd /path/to/collection

# Check for lint-labeled environments
grep -q "labels.*lint" tox.ini && echo "Found lint label"

# Or check for linters environment
tox -l | grep -q "linters" && echo "Found linters environment"
```

Expected tox.ini configuration:

```ini
[tox]
labels =
    lint = ansible-lint, black-lint, isort-lint, flynt-lint, flake8-lint, pylint-lint

[testenv:ansible-lint]
...

[testenv:black-lint]
...
```

### Step 2: Run Linters

```bash
# If using labels (modern tox)
tox -m lint

# Or if using linters environment
tox -e linters
```

This runs:

- **ansible-lint**: Ansible best practices
- **black-lint**: Python code formatting check
- **isort-lint**: Import sorting check
- **flynt-lint**: f-string usage check
- **flake8-lint**: Python style guide (PEP 8)
- **pylint-lint**: Python code analysis
- **ruff-lint**: Fast Python linter (if configured)

### Step 3: Run ansible-lint Separately (CRITICAL)

**ALWAYS run ansible-lint separately** if it's not included in tox configuration:

```bash
# Check if ansible-lint was run by tox
if ! (tox -l | grep -q "ansible-lint" || grep -q "ansible-lint" tox.ini); then
  echo "Running ansible-lint separately..."
  ansible-lint
fi
```

**Why this is critical:**

- Many Ansible collections only configure Python linters in tox
- ansible-lint checks Ansible-specific issues like YAML indentation
- CI often runs ansible-lint separately
- Prevents "ansible-lint passed locally but failed in CI"

### Step 4: Review Results

**On success:**

```
✅ ansible-lint: PASSED (0 failures, 0 warnings in 32 files)
✅ black-lint: PASSED (21 files unchanged)
✅ isort-lint: PASSED
✅ flynt-lint: PASSED
✅ flake8-lint: PASSED
✅ pylint-lint: PASSED (10.00/10)

All linters passed! ✨
```

**On failures:**

```
❌ black-lint: FAILED (3 files would be reformatted)
   plugins/modules/my_module.py
   
   ➤ Auto-fix: tox -e black

❌ pylint: FAILED (score: 8.45/10)
   plugins/modules/my_module.py:45: unused-variable 'result'
   
   ⚠️  Manual fix required
```

## Linter Categories

### Auto-Fixable

- **black**: Code formatting → `tox -e black`
- **isort**: Import sorting → `tox -e isort`
- **flynt**: f-string conversions → `tox -e flynt`
- **ruff format**: Fast formatting → `tox -e ruff-format`

Quick fix command:

```bash
tox -e black,isort,flynt
```

### Manual Fix Required

- **pylint**: Code quality, complexity, unused variables
- **flake8**: Logic errors, undefined variables, style
- **ansible-lint**: Best practice violations
- **mypy**: Type checking errors

## Troubleshooting

### "tox.ini not found"

Ensure you're in the collection root:

```bash
cd /path/to/collection
ls tox.ini
```

### "No lint environments configured"

Add lint label to tox.ini:

```ini
[tox]
labels =
    lint = black-lint, flake8-lint, pylint-lint
```

### Slow execution

Run specific linter only:

```bash
tox -e ansible-lint
```

## Integration

This skill integrates with:

- `stable-release-prep` - Run after prep before commit
- `sanity` - Run alongside sanity tests
- `stable-release` - Part of full release workflow (quality checks)
