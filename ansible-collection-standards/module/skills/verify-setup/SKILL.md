---
description: Verify contributor environment is ready for ai-forge contributions
allowed-tools: Bash(command:git *), Bash(command:lola *), Bash(command:python *), Bash(command:which *), Bash(command:pre-commit *), Bash(command:gh *)
argument-hint: ""
---

# Verify Contribution Setup

Checks that your development environment is configured correctly for contributing to ansible-community/ai-forge.

This skill verifies:
- Git configuration (user name, email, signing)
- Pre-commit hooks installation
- Lola package manager
- AI assistant availability
- Development tools (Python, linters)
- Repository setup (fork, upstream remote)

## Usage

```bash
/verify-setup
```

Run this before your first contribution to catch configuration issues early.

## Step 1: Check Git Configuration

Verify git user configuration:

```bash
# Check user name
git config user.name || echo "❌ Git user.name not set"

# Check user email
git config user.email || echo "❌ Git user.email not set"

# Check commit signing (optional)
git config commit.gpgsign && echo "✅ GPG signing enabled" || echo "ℹ️  GPG signing not enabled (optional)"
```

**If not set**, provide instructions:
```
To configure git:
  git config --global user.name "Your Name"
  git config --global user.email "your.email@example.com"

Optional (recommended):
  git config --global commit.gpgsign true
```

## Step 2: Check Pre-commit Hooks

Verify pre-commit is installed and hooks are set up:

```bash
# Check pre-commit command exists
if command -v pre-commit &> /dev/null; then
  echo "✅ pre-commit installed: $(pre-commit --version)"
  
  # Check if hooks are installed in this repo
  if [ -f .git/hooks/pre-commit ]; then
    echo "✅ Pre-commit hooks installed"
  else
    echo "⚠️  Pre-commit hooks not installed in this repository"
    echo "   Run: pre-commit install"
  fi
else
  echo "❌ pre-commit not found"
  echo "   Install: pip install pre-commit"
fi
```

## Step 3: Check Lola Installation

Verify Lola package manager is available:

```bash
if command -v lola &> /dev/null; then
  LOLA_VERSION=$(lola --version 2>&1 | head -1)
  echo "✅ Lola installed: $LOLA_VERSION"
else
  echo "❌ Lola not found"
  echo "   Install: pip install lola-ai"
fi
```

## Step 4: Check AI Assistant

Check if user has an AI assistant configured:

```bash
# This skill is being invoked by an AI assistant, so inherently this check passes
echo "✅ AI assistant active (you're using it now!)"

# Check which assistant by examining environment or context
# Common assistants: Claude Code, Cursor, Gemini CLI
echo "   Note: Test that you can load skills from local paths"
```

## Step 5: Check Development Tools

Verify required and optional tools:

```bash
# Python version
if command -v python3 &> /dev/null; then
  PYTHON_VERSION=$(python3 --version 2>&1)
  echo "✅ Python installed: $PYTHON_VERSION"
  
  # Check version >= 3.8
  PYTHON_MINOR=$(python3 -c "import sys; print(sys.version_info.minor)")
  if [ "$PYTHON_MINOR" -ge 8 ]; then
    echo "   (meets requirement: Python 3.8+)"
  else
    echo "   ⚠️  Python 3.8+ recommended"
  fi
else
  echo "❌ Python 3 not found"
fi

# yamllint
if command -v yamllint &> /dev/null; then
  echo "✅ yamllint installed"
else
  echo "ℹ️  yamllint not found (used by pre-commit hooks)"
  echo "   Install: pip install yamllint"
fi

# markdownlint (npm package)
if command -v markdownlint-cli2 &> /dev/null; then
  echo "✅ markdownlint-cli2 installed"
else
  echo "ℹ️  markdownlint-cli2 not found (used by pre-commit hooks)"
  echo "   Install: npm install -g markdownlint-cli2"
fi

# shellcheck (optional)
if command -v shellcheck &> /dev/null; then
  echo "✅ shellcheck installed"
else
  echo "ℹ️  shellcheck not found (optional, used for shell script validation)"
fi
```

## Step 6: Check Repository Setup

Verify fork and upstream configuration:

```bash
# Check if we're in a git repository
if ! git rev-parse --git-dir > /dev/null 2>&1; then
  echo "❌ Not in a git repository"
  echo "   Clone your fork first: git clone git@github.com:YOUR_USERNAME/ai-forge.git"
  exit 1
fi

# Check current repository
CURRENT_REPO=$(git remote get-url origin 2>/dev/null || echo "none")
echo "Current repository: $CURRENT_REPO"

# Check if origin is a fork (not ansible-community/ai-forge)
if echo "$CURRENT_REPO" | grep -q "ansible-community/ai-forge"; then
  echo "⚠️  Origin points to upstream (not your fork)"
  echo "   You should be working from your fork"
  echo "   Fork at: https://github.com/ansible-community/ai-forge/fork"
else
  echo "✅ Origin is your fork"
fi

# Check upstream remote exists
if git remote get-url upstream &> /dev/null; then
  UPSTREAM_URL=$(git remote get-url upstream)
  echo "✅ Upstream remote configured: $UPSTREAM_URL"
  
  # Verify it points to ansible-community/ai-forge
  if echo "$UPSTREAM_URL" | grep -q "ansible-community/ai-forge"; then
    echo "   (correctly points to ansible-community/ai-forge)"
  else
    echo "   ⚠️  Upstream should point to ansible-community/ai-forge"
  fi
else
  echo "❌ Upstream remote not configured"
  echo "   Add upstream: git remote add upstream https://github.com/ansible-community/ai-forge.git"
fi

# Check if main branch is up to date
CURRENT_BRANCH=$(git branch --show-current)
if [ "$CURRENT_BRANCH" = "main" ]; then
  git fetch upstream main &> /dev/null
  
  BEHIND=$(git rev-list --count HEAD..upstream/main 2>/dev/null || echo "unknown")
  if [ "$BEHIND" = "0" ]; then
    echo "✅ Main branch up to date with upstream"
  elif [ "$BEHIND" = "unknown" ]; then
    echo "ℹ️  Could not check if main is up to date"
  else
    echo "⚠️  Main branch is $BEHIND commits behind upstream"
    echo "   Update: git checkout main && git merge upstream/main"
  fi
else
  echo "ℹ️  Current branch: $CURRENT_BRANCH (not main)"
fi
```

## Step 7: Generate Summary Report

Collect all results and provide next steps:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Contribution Setup Verification
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Display summary of all checks with ✅/❌/⚠️/ℹ️ status]

Ready to contribute: [YES/NO/ALMOST]

Next steps:
  [If ready]
  1. Create feature branch: git checkout -b my-feature
  2. Make your changes
  3. Test locally: ask your AI assistant to run/test your skill
  4. Commit: git add <files> && git commit -m "Your message"
  5. Push: git push origin my-feature
  6. Create PR: gh pr create --repo ansible-community/ai-forge

  [If not ready]
  - Fix the ❌ items listed above
  - Install missing required tools
  - Configure git settings
  - Run /verify-setup again

  [If almost ready]
  - Optional items (ℹ️) can be installed later
  - Pre-commit hooks will catch issues before commit
  - You can proceed with contributions

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Troubleshooting

### "Git user not set"

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### "Pre-commit hooks not installed"

```bash
# Install pre-commit
pip install pre-commit

# Install hooks in repository
cd ~/path/to/ai-forge
pre-commit install

# Test hooks work
pre-commit run --all-files
```

### "Upstream remote not found"

```bash
cd ~/path/to/ai-forge
git remote add upstream https://github.com/ansible-community/ai-forge.git
git fetch upstream
```

### "Not a fork"

If you cloned ansible-community/ai-forge directly:

1. Fork the repository at https://github.com/ansible-community/ai-forge/fork
2. Add your fork as origin:
   ```bash
   git remote set-url origin git@github.com:YOUR_USERNAME/ai-forge.git
   ```
3. Add upstream:
   ```bash
   git remote add upstream https://github.com/ansible-community/ai-forge.git
   ```

## References

- [CONTRIBUTING.md](../../../CONTRIBUTING.md) - Full contribution guidelines
- [GOVERNANCE.md](../../../GOVERNANCE.md) - Project governance
- [Pre-commit hooks](https://pre-commit.com/) - Automated quality checks
- [Lola package manager](https://github.com/lola-ai/lola) - AI skill management
