# Ansible Collection SDLC Module

A Lola module for the full software development lifecycle of Ansible collections: conventional commits, changelog
fragments, PR reviews, releases, and testing.  Streamlines day-to-day development workflows from code commit to
production release.

## Installation

```bash
# Install Lola package manager
pip install lola-cli

# Register the module from GitHub
lola mod add https://github.com/ansible-community/ai-forge/ansible-collection-sdlc

# Or clone and register locally
git clone https://github.com/ansible-community/ai-forge.git
lola mod add ./ai-forge/ansible-collection-sdlc

# Install to Claude Code
lola install ansible-collection-sdlc -a claude-code

# Install to Cursor
lola install ansible-collection-sdlc -a cursor

# Install to other assistants
lola install ansible-collection-sdlc -a gemini-cli
lola install ansible-collection-sdlc -a opencode
```

## Configuration (Optional)

Skills work with sensible defaults. For customization, create a configuration file:

```bash
# Copy the template
cp ansible-release.conf.template ~/.ansible-release.conf

# Edit with your preferences
vim ~/.ansible-release.conf

# Source before using skills (or add to your shell profile)
source ~/.ansible-release.conf
```

Available configuration options:
- `ANSIBLE_COLLECTIONS_PATH` - Where collections are stored
- `GITHUB_USERNAME` - Your GitHub username for PRs
- `SANITY_MODE` - Default sanity test mode (smart/full/changed-only)
- `AUTO_CREATE_PR` - Automatically create PRs (true/false/prompt)
- Collection-specific overrides (e.g., `amazon_aws_SANITY_MODE`)

See `ansible-release.conf.template` for all options and documentation.

## Components

### Skills

- **changelog-fragment** - Create or update changelog fragments for documenting changes with automatic change analysis
- **commit** - Create conventional commits with FQCN scopes for Ansible collection content
- **create-branch** - Create feature branches following project conventions with proper fork workflow setup
- **create-pr** - Create draft pull requests with pre-flight checks, changelog validation, and automated formatting
- **docs-generate** - Generate module documentation and update README using collection_prep
- **implement-sonarcloud-fixes** - Implement fixes for SonarCloud issues with testing and PR creation
- **next-release** - Calculate next patch/minor/major release versions for version_added tags following SemVer
- **pr-review** - Review PRs against project standards and the Ansible Collection Review Checklist
- **release** - Guide collection releases with automatic version detection from changelog fragments
- **remove-deprecations** - Find and remediate overdue deprecation warnings with guided removal workflow
- **run-tests** - Run and write sanity, unit, and integration tests using ansible-test
- **sanity** - Run Ansible sanity tests with smart change detection (fast, targeted testing)
- **sonarcloud-analysis** - Fetch and analyse SonarCloud issues for projects or pull requests
- **stable-release** - Guide releases from stable-X branches with SemVer calculation from changelog fragments
- **stable-release-analyze** - Analyze stable branches for pending releases and calculate versions
- **stable-release-prep** - Prepare release branch with changelog generation and version updates
- **tox-lint** - Run all configured tox linters (black, isort, flake8, pylint, ansible-lint, ruff)

#### Helper Skills

- **current-release** - Fetch current release version from git tags/branches or galaxy.yml (used by other skills)
- **get-branch-changes** - Determine merge-base and changed files for current branch, avoiding unrelated changes when behind target (used by other skills)
- **get-pr-action-results** - Get GitHub Actions/GitLab CI results for PRs and branches, analyze failures, and suggest fixes (used by other skills)
- **get-pr-number** - Determine pull request number for a branch (used by other skills)
- **get-upstream-info** - Determine upstream repository information and service identifiers (used by other skills)

### Commands

- **/check-pr-actions** - Check GitHub Actions/GitLab CI status and analyze failures
- **/check-pr-sonarcloud** - Check SonarCloud analysis results for the current pull request

### Agents

None currently defined.

### MCP Servers

None currently defined.

## Development

This module follows the Lola module structure:

```
ansible-collection-sdlc/
├── README.md           # This file
└── module/             # Lola-importable content
    ├── AGENTS.md       # Module-level instructions
    ├── skills/         # Skill folders with SKILL.md
    ├── commands/       # Slash command .md files
    ├── agents/         # Subagent .md files
    └── mcps.json       # MCP server configuration
```

## Dependencies

- **antsibull-changelog** (optional) - Used for changelog generation
- **gh CLI** (optional) - Used for GitHub/GitLab operations (PRs, releases, upstream detection)
- **ansible-test** - Used for running sanity, unit, and integration tests
- **curl** (optional) - Used for fetching SonarCloud analysis results

## License

GPL-3.0-or-later
