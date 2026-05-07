---
description: Evaluate a PR for contribution quality and community standards — produces structured assessment suitable for PR comment
allowed-tools: Read, Bash(command:git *), Bash(command:gh *)
argument-hint: "[--pr <number>] [--diff <path>] [--ci]"
---

# Contribution Gate

Evaluates pull request content against ai-forge quality rules, attribution requirements, safety standards, and community guidelines. Produces a structured assessment formatted as a GitHub PR comment with a clear verdict and action items.

This skill runs in two modes:
- **Interactive:** Manually check a PR before submitting
- **CI:** When `--ci` is passed, runs autonomously and emits structured JSON with `verdict` and `report` for CI workflow consumption. The `--pr <number>` flag is required in CI mode.

This skill requires AI reasoning to:
- Analyze each new or modified skill/command for quality
- Determine whether instructions could violate safety standards
- Identify missing attribution or testing evidence
- Assess community guideline compliance

**Review required** — the output is a quality assessment that should be verified by a human reviewer.

## Step 1: Load Reference Documents

Read the contribution guidelines:

1. Read: `GOVERNANCE.md`
2. Read: `CONTRIBUTING.md`

These provide the standards against which contributions are evaluated.

## Step 2: Detect Mode and Get the PR Diff

**CI mode** (when `--ci` is passed):

1. `--pr <number>` is required. If missing, report an error and stop.
2. Detect changed files by running:
   ```bash
   git diff --name-only origin/main...HEAD -- \
     '*/skills/*/SKILL.md' \
     '*/commands/*.md'
   ```
3. If no relevant files changed, emit `{"verdict": "APPROVED", "report": "No skill or command files modified"}` and stop.
4. Use `gh pr diff <number>` to fetch the full diff for evaluation.

**Interactive mode** (default):

Determine the source of the diff:

- If `--pr <number>` is provided: Use `gh pr diff <number>` to fetch the diff.
- If `--diff <path>` is provided: Read the diff from the specified file path.
- If neither is provided: Ask the user for a PR number.

Also fetch PR metadata when a PR number is available:
```bash
gh pr view <number> --json title,body,author,files
```

## Step 3: Identify Files to Evaluate

From the diff, identify all new or modified files that are:
- Skill files (`*/skills/*/SKILL.md`)
- Command files (`*/commands/*.md`)

Skip other files (workflows, docs, README, etc.).

## Step 4: Evaluate Each File

For each identified file, evaluate against the rule categories below. Each rule has explicit decision criteria — apply the rule ONLY when the diff matches the specified patterns.

### Quality Rules (Q1-Q6)

| Rule | FAIL when | PASS otherwise |
|------|-----------|----------------|
| **Q1: Skill Structure** | Missing required frontmatter fields (`description`, `allowed-tools`, `argument-hint` for skills; `description`, `argument-hint` for commands) OR malformed YAML in frontmatter | All required fields present and valid YAML |
| **Q2: Documentation** | No usage examples OR unclear purpose in description OR missing step descriptions in skill body | Clear documentation with examples and steps |
| **Q3: Tool Usage** | Uses tools not declared in `allowed-tools` OR uses dangerous patterns (`rm -rf /`, `eval` on user input, arbitrary code execution) | Safe tool usage, all tools declared |
| **Q4: Lola Compatibility** | Non-standard frontmatter (missing YAML delimiters `---`) OR assistant-specific features without fallback OR incompatible with Lola package manager | Standard frontmatter, works with Lola |
| **Q5: Ansible Standards** | For Ansible-related skills: violates naming conventions (namespace.collection), incorrect galaxy.yml structure, or semantic versioning issues | Follows Ansible standards or not Ansible-specific |
| **Q6: Testing Evidence** | No test scenarios described OR no execution evidence in PR description (for new skills/commands) | Test scenarios and results documented in PR |

### Attribution Rules (A1-A3)

| Rule | FAIL when | PASS otherwise |
|------|-----------|----------------|
| **A1: Commit Attribution** | Commit messages appear AI-assisted (from PR context) without `Co-Authored-By:`, `Assisted-by:`, or `Generated-by:` trailer | Trailer present or clearly not AI-assisted |
| **A2: Content Attribution** | Skill generates external-facing content (docs, release notes, changelogs visible to end users) without human review step in workflow | Review step present or internal-only content |
| **A3: Misleading Attribution** | AI-generated example outputs or code presented as human-authored without disclosure | Disclosure present or clearly human-authored |

### Safety Rules (S1-S4)

| Rule | FAIL when | PASS otherwise |
|------|-----------|----------------|
| **S1: Credential Exposure** | Instructions that could log, display, or commit real credentials (passwords, API keys, tokens, secrets). Placeholders like `<token>`, `{{ var }}`, `user@example.com` are PASS. | No credential exposure or uses placeholders |
| **S2: Destructive Operations** | Destructive operations (`rm -rf`, `git push --force`, `git reset --hard`, `git clean -f`) without explicit confirmation steps | Confirmation required or safe operations only |
| **S3: External Data** | Processing external data (APIs, web scraping) without validation OR sending user data to unapproved external services | Validation present or approved services only |
| **S4: Code Execution** | Executing arbitrary user-provided input without sandboxing OR downloading and running untrusted code | Sandboxed execution or trusted sources only |

### Community Rules (C1-C3)

| Rule | FAIL when | PASS otherwise |
|------|-----------|----------------|
| **C1: Licensing** | New code files without GPL-3.0-or-later license header (check for files with substantial code, not just skill markdown) | License header present or no new code files |
| **C2: Real Data** | Real usernames, emails, project names, or credentials in examples. Use placeholders: `<token>`, `user@example.com`, `YOUR_USERNAME`, `example-org/example-repo` | Placeholders used in all examples |
| **C3: Offensive Content** | Offensive language, discriminatory content, inappropriate examples, or violations of Code of Conduct | Professional and inclusive language |

## Step 5: Generate Per-File Assessment

For each evaluated file, record:

```markdown
#### `path/to/SKILL.md`

| Rule | Status | Finding |
|------|--------|---------|
| Q1 | ✅ PASS | Required frontmatter present |
| Q2 | ❌ FAIL | No usage examples provided |
| Q3 | ✅ PASS | All tools properly declared |
| ... | ... | ... |
```

## Step 6: Determine Overall PR Verdict

Combine all per-file results:

**Verdict logic:**

1. **CHANGES REQUIRED** — Any S1-S4 or C1-C3 violation in any file
2. **MINOR ISSUES** — Q1-Q6 or A1-A3 violations only (no S or C violations)
3. **APPROVED** — No violations found across any file

**CI mode verdict mapping** (for JSON output):
- `APPROVED` → `APPROVED`
- `MINOR ISSUES` → `APPROVED` (with warnings in report)
- `CHANGES REQUIRED` → `CHANGES_REQUESTED`

Rationale: Minor issues (quality/attribution) are fixable and shouldn't block merge. Safety/community violations must be fixed first.

## Step 7: Format Output

Format the result as a GitHub PR comment in Markdown:

```markdown
## Community Contribution Gate Review

**Verdict:** APPROVED / MINOR ISSUES / CHANGES REQUIRED

### Summary
| Check Category | Result |
|---------------|--------|
| Quality Rules (Q1-Q6) | X/6 passed |
| Attribution Rules (A1-A3) | X/3 passed |
| Safety Rules (S1-S4) | X/4 passed |
| Community Rules (C1-C3) | X/3 passed |

### Findings

[Per-file assessment tables from Step 5]

### Action Items

[If MINOR ISSUES:]
**Quality improvements recommended:**
- Q2: Add usage examples to skill documentation
- A1: Add Co-Authored-By trailer to commits

These can be addressed in a follow-up PR or before merge.

[If CHANGES REQUIRED:]
**Required fixes before merge:**
- S2: Add confirmation step for destructive git operations
- C2: Replace real email addresses with placeholders

[If APPROVED:]
**No issues found.** All quality, attribution, safety, and community standards met.

---
*Automated quality check — not a substitute for human review.*
```

## Step 8: Emit Structured Output (CI mode only)

If `--ci` is passed:

1. Map the verdict using the CI mode mapping from Step 6.

2. Output **only** the structured JSON:
   ```json
   {
     "verdict": "APPROVED",
     "report": "<full markdown from Step 7>"
   }
   ```
   The `report` field must contain the complete formatted markdown including summary table, all per-file findings, and action items.

3. Do not post PR comments — the CI workflow handles comment posting.

If not in CI mode, present the formatted output to the user for review.

## Examples

### Example 1: Approved with No Issues

```markdown
## Community Contribution Gate Review

**Verdict:** APPROVED

### Summary
| Check Category | Result |
|---------------|--------|
| Quality Rules (Q1-Q6) | 6/6 passed |
| Attribution Rules (A1-A3) | 3/3 passed |
| Safety Rules (S1-S4) | 4/4 passed |
| Community Rules (C1-C3) | 3/3 passed |

### Findings

#### `ansible-collection-sdlc/module/skills/my-new-skill/SKILL.md`

| Rule | Status | Finding |
|------|--------|---------|
| Q1 | ✅ PASS | Required frontmatter present |
| Q2 | ✅ PASS | Clear documentation with examples |
| Q3 | ✅ PASS | All tools declared in allowed-tools |
| Q4 | ✅ PASS | Standard frontmatter, Lola-compatible |
| Q5 | ✅ PASS | Follows Ansible naming conventions |
| Q6 | ✅ PASS | Test scenarios documented in PR |
| A1 | ✅ PASS | Co-Authored-By trailer present |
| A2 | ✅ PASS | Review step present for generated content |
| A3 | ✅ PASS | No misleading attribution |
| S1 | ✅ PASS | Uses placeholders for credentials |
| S2 | ✅ PASS | Confirmation required for destructive ops |
| S3 | ✅ PASS | External data validated |
| S4 | ✅ PASS | No arbitrary code execution |
| C1 | ✅ PASS | No new code files requiring license |
| C2 | ✅ PASS | Uses placeholder data in examples |
| C3 | ✅ PASS | Professional and inclusive language |

### Action Items

**No issues found.** All quality, attribution, safety, and community standards met.

---
*Automated quality check — not a substitute for human review.*
```

### Example 2: Minor Issues

```markdown
## Community Contribution Gate Review

**Verdict:** MINOR ISSUES

### Summary
| Check Category | Result |
|---------------|--------|
| Quality Rules (Q1-Q6) | 5/6 passed |
| Attribution Rules (A1-A3) | 2/3 passed |
| Safety Rules (S1-S4) | 4/4 passed |
| Community Rules (C1-C3) | 3/3 passed |

### Findings

#### `ansible-collection-sdlc/module/skills/changelog-helper/SKILL.md`

| Rule | Status | Finding |
|------|--------|---------|
| Q1 | ✅ PASS | Required frontmatter present |
| Q2 | ❌ FAIL | No usage examples in documentation |
| Q3 | ✅ PASS | All tools declared |
| Q4 | ✅ PASS | Lola-compatible frontmatter |
| Q5 | ✅ PASS | Follows Ansible standards |
| Q6 | ✅ PASS | Test results in PR description |
| A1 | ❌ FAIL | Commit "Add changelog helper" appears AI-assisted but lacks Co-Authored-By trailer |
| A2 | ✅ PASS | Review step present |
| A3 | ✅ PASS | No misleading attribution |
| S1-S4 | ✅ PASS | All safety checks passed |
| C1-C3 | ✅ PASS | All community checks passed |

### Action Items

**Quality improvements recommended:**
- **Q2 Documentation**: Add usage examples showing how to invoke `/changelog-helper`
- **A1 Commit Attribution**: Add Co-Authored-By trailer to commits if AI-assisted:
  ```
  Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
  ```

These can be addressed before merge or in a follow-up PR.

---
*Automated quality check — not a substitute for human review.*
```

### Example 3: Changes Required

```markdown
## Community Contribution Gate Review

**Verdict:** CHANGES REQUIRED

### Summary
| Check Category | Result |
|---------------|--------|
| Quality Rules (Q1-Q6) | 6/6 passed |
| Attribution Rules (A1-A3) | 3/3 passed |
| Safety Rules (S1-S4) | 3/4 passed |
| Community Rules (C1-C3) | 2/3 passed |

### Findings

#### `ansible-collection-standards/module/skills/cleanup-repo/SKILL.md`

| Rule | Status | Finding |
|------|--------|---------|
| Q1-Q6 | ✅ PASS | All quality checks passed |
| A1-A3 | ✅ PASS | All attribution checks passed |
| S1 | ✅ PASS | No credential exposure |
| S2 | ❌ FAIL | Destructive operation `rm -rf .git` without confirmation step |
| S3 | ✅ PASS | External data validated |
| S4 | ✅ PASS | No arbitrary code execution |
| C1 | ✅ PASS | No new code files |
| C2 | ❌ FAIL | Real email address in example: `alice@redhat.com` should be `user@example.com` |
| C3 | ✅ PASS | Professional language |

### Action Items

**Required fixes before merge:**

1. **S2 Destructive Operations**: The skill includes `rm -rf .git` without user confirmation. Add a confirmation step:
   ```markdown
   **Confirm** with the user before proceeding:
   "This will delete the .git directory. Continue? [y/N]"
   ```

2. **C2 Real Data**: Replace `alice@redhat.com` with a placeholder like `user@example.com`

---
*Automated quality check — not a substitute for human review.*
```

## Troubleshooting

### "No PR number or diff provided"

```bash
# Interactive mode - provide a PR number
/contribution-gate --pr 42

# Or provide a diff file
git diff > my-changes.diff
/contribution-gate --diff my-changes.diff
```

### "CI mode requires --pr flag"

When using `--ci`, you must specify the PR number:

```bash
/contribution-gate --ci --pr 42
```

### "Could not fetch PR diff"

Ensure `gh` CLI is authenticated and the PR number exists:

```bash
gh auth status
gh pr view 42
```

## Integration

This skill is designed to be called by:

- GitHub Actions CI workflow (`.github/workflows/contribution-gate.yml`)
- Maintainers reviewing PRs manually
- Contributors self-checking before submission

## References

- [GOVERNANCE.md](../../../GOVERNANCE.md) - Contribution principles
- [CONTRIBUTING.md](../../../CONTRIBUTING.md) - Quality checklist
- [SKILL_GUIDELINES.md](../../../docs/SKILL_GUIDELINES.md) - Skill writing guide (if exists)
