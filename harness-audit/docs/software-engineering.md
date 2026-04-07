# Software Engineering Domain Adapter

Domain-specific detection logic, file paths, commands, and auto-fix templates for software projects.

Use alongside the main `SKILL.md` and `scoring-rubric.md`.

---

## Data Gathering Commands

### Self-Deception 1: The Stale Goal

```bash
# Find goal documents
find docs/ -maxdepth 2 \( -name "PRD*" -o -name "spec*" -o -name "requirements*" \
  -o -name "features*" -o -name "decisions*" \) 2>/dev/null

# Check instruction file references to goals
grep -i -E "(goal|objective|spec|prd|requirement|feature)" CLAUDE.md 2>/dev/null

# Goal freshness: compare dates
echo "Goal doc last modified:"
git log -1 --format='%ar' -- docs/PRD.md docs/requirements.md docs/features.md 2>/dev/null
echo "Most recent project commit:"
git log -1 --format='%ar' 2>/dev/null

# Check if goals have acceptance criteria
grep -c -i -E "(accept|criteria|验收|done when|success if)" docs/PRD.md docs/features.md 2>/dev/null
```

### Self-Deception 2: The Monolith

```bash
# Instruction file size
wc -l CLAUDE.md .cursorrules AGENTS.md .github/copilot-instructions.md 2>/dev/null

# Progressive disclosure: links to other docs
grep -c "\[.*\](.*\.md)" CLAUDE.md 2>/dev/null

# Module-level instruction files
find . -name "CLAUDE.md" -not -path "./CLAUDE.md" -not -path "*/node_modules/*" 2>/dev/null

# Emphasis ratio: "critical" words vs total rules
CRITICAL=$(grep -c -i -E "must|never|always|绝对|必须|不得|禁止" CLAUDE.md 2>/dev/null)
TOTAL=$(grep -c "^-\|^[0-9]\.\|^\*" CLAUDE.md 2>/dev/null)
echo "Emphasis ratio: $CRITICAL / $TOTAL"
```

### Self-Deception 3: The Paper Rule

```bash
# Extract "don't do X" rules from instruction file
echo "=== Paper rules in CLAUDE.md ==="
grep -n -i -E "don't|never|must not|不要|不得|禁止|不允许" CLAUDE.md 2>/dev/null

# Check CI for custom (non-default) checks
echo "=== CI workflows ==="
ls .github/workflows/*.yml .gitlab-ci.yml 2>/dev/null
# Look for custom check steps (not just standard lint/test/build)
grep -l "harness\|architecture\|custom.*check\|grep.*-r" .github/workflows/*.yml 2>/dev/null

# Pre-commit hooks
echo "=== Pre-commit hooks ==="
cat .husky/pre-commit 2>/dev/null || cat .git/hooks/pre-commit 2>/dev/null

# Custom check scripts
echo "=== Custom check scripts ==="
find scripts/ -name "*harness*" -o -name "*check*" -o -name "*lint*" 2>/dev/null

# Architecture tests
echo "=== Architecture tests ==="
find . -name "*.arch.*" -o -name "*architecture*test*" -not -path "*/node_modules/*" 2>/dev/null | head -5

# Commitlint
ls .commitlintrc* commitlint.config.* 2>/dev/null
```

### Self-Deception 4: The Gut Review

```bash
# Quality scorecard
ls docs/QUALITY_SCORE.md 2>/dev/null

# Test configuration
ls jest.config.* vitest.config.* coverage/ 2>/dev/null

# Coverage thresholds in config
grep -A5 "threshold\|coverage" jest.config.* vitest.config.* 2>/dev/null

# Review automation
ls .github/workflows/*review* .github/workflows/*check* 2>/dev/null

# NestJS: Swagger coverage (API documentation completeness)
for f in $(find . -name "*.controller.ts" -not -path "*/node_modules/*" 2>/dev/null); do
  grep -L "@ApiTags" "$f" 2>/dev/null
done
```

### Self-Deception 5: The Frozen Harness

```bash
# Harness commits in last 30 days
HARNESS_COMMITS=$(git log --since="30 days ago" --oneline -- \
  CLAUDE.md .cursorrules AGENTS.md docs/ scripts/harness-checks.sh \
  .eslintrc* commitlint.config.* .husky/ 2>/dev/null | wc -l)
TOTAL_COMMITS=$(git log --since="30 days ago" --oneline 2>/dev/null | wc -l)
echo "Harness commits (30d): $HARNESS_COMMITS / $TOTAL_COMMITS total"

# Last modified dates
echo "Instruction file last modified:"
git log -1 --format='%ar %s' -- CLAUDE.md 2>/dev/null
echo "Check scripts last modified:"
git log -1 --format='%ar %s' -- scripts/harness-checks.sh .husky/ 2>/dev/null
echo "Docs last modified:"
git log -1 --format='%ar %s' -- docs/ 2>/dev/null
```

---

## Files to Read

```
# Goal documents
README.md
docs/PRD.md
docs/product-spec.md
docs/requirements.md
docs/features.md
docs/decisions/*.md

# Instruction files
CLAUDE.md
.cursorrules
AGENTS.md
.github/copilot-instructions.md

# Memory
~/.claude/projects/*/memory/MEMORY.md
~/.claude/projects/*/memory/*.md

# CI/CD & checks
.github/workflows/*.yml
.gitlab-ci.yml
scripts/harness-checks.sh
.husky/pre-commit
.commitlintrc*
commitlint.config.*

# Quality signals
docs/QUALITY_SCORE.md
coverage/
jest.config.*
vitest.config.*

# Documentation
docs/
CONTRIBUTING.md
```

---

## Software-Specific Scoring Examples

### Self-Deception 1: The Stale Goal

| Score | Example |
|-------|---------|
| 0 | No README, no PRD, no spec. Agent reads code to guess purpose. |
| 2 | `docs/PRD.md` exists but CLAUDE.md doesn't mention it. Agent doesn't know to check it. |
| 3 | PRD referenced from CLAUDE.md, but written 90 days ago. Product has pivoted since. |
| 5 | `features.json` with machine-readable acceptance criteria, updated with every sprint. Agent can verify task alignment. |

### Self-Deception 2: The Monolith

| Score | Example |
|-------|---------|
| 1 | CLAUDE.md is 250 lines. Architecture rules, code style, build commands, personal preferences all mixed. Emphasis ratio: 65%. |
| 3 | CLAUDE.md is 90 lines with sections. Links to `docs/architecture.md`. But no package-level docs. |
| 5 | CLAUDE.md is 55 lines: build commands, top 5 critical rules, and a TOC linking to package-level docs. Each package has its own instruction file. |

### Self-Deception 3: The Paper Rule

| Score | Example |
|-------|---------|
| 1 | CLAUDE.md has 8 "never do X" rules. CI runs `eslint` and `tsc` only. Zero custom checks. Paper rule count: 8. |
| 3 | 5 "never" rules, 3 have corresponding architecture tests. Paper rule count: 2. |
| 5 | All documented rules enforced. `scripts/harness-checks.sh` has 12 grep-based checks. Adding a new one = adding one line. |

### Self-Deception 4: The Gut Review

| Score | Example |
|-------|---------|
| 1 | Tests exist but no coverage thresholds. PRs merged without structured review. |
| 3 | `docs/QUALITY_SCORE.md` with module grades. Code review checklist exists. |
| 5 | Quality scorecard maintained, coverage thresholds enforced in CI, dedicated review agent scans for doc-code inconsistencies. |

### Self-Deception 5: The Frozen Harness

| Score | Example |
|-------|---------|
| 1 | CLAUDE.md last modified 4 months ago. 0 harness commits in 30 days. |
| 3 | 3 harness commits in 30 days. One new check added after a recurring bug. |
| 5 | 12 harness commits in 30 days (15% of total). Three error→rule conversions documented in git history. Monthly harness review scheduled. |

---

## Auto-Fix Templates

### Fix: Add goal reference to CLAUDE.md

```markdown
# Add at the top of CLAUDE.md, after the first heading:

## Project Goals
See [docs/PRD.md](docs/PRD.md) for current product objectives and acceptance criteria.
All tasks should trace back to a stated objective in the PRD.
```

### Fix: Create harness-checks.sh

```bash
#!/bin/bash
# Harness checks — project-specific rules that CI can't catch with default lint
# Add one grep per rule. Exit 1 if any violation found.

ERRORS=0

# Example: No direct DOM manipulation in service layer
if grep -r "document\.\|window\.\|getElementById" src/services/ 2>/dev/null; then
  echo "❌ VIOLATION: DOM access in service layer"
  ERRORS=$((ERRORS + 1))
fi

# Example: No console.log in production code
if grep -r "console\.log" src/ --include="*.ts" --exclude="*.test.*" --exclude="*.spec.*" 2>/dev/null; then
  echo "❌ VIOLATION: console.log in production code"
  ERRORS=$((ERRORS + 1))
fi

# ADD NEW RULES HERE — one grep per rule

if [ $ERRORS -gt 0 ]; then
  echo "Found $ERRORS harness violations"
  exit 1
fi
echo "✅ All harness checks passed"
```

### Fix: Create QUALITY_SCORE.md template

```markdown
# Quality Scorecard

Last updated: {date}

| Module | Tests | Docs | Types | Grade |
|--------|-------|------|-------|-------|
| {module1} | ✅/❌ | ✅/❌ | ✅/❌ | A/B/C/D |
| {module2} | ✅/❌ | ✅/❌ | ✅/❌ | A/B/C/D |

## Grade Definitions
- **A**: Full test coverage, docs current, types complete
- **B**: Tests exist, docs exist but may be stale, types mostly complete
- **C**: Partial tests, minimal docs, some type gaps
- **D**: No tests, no docs, or significant type gaps

## Update Triggers
Update this scorecard when: new module added, major refactor completed, 
or monthly review (whichever comes first).
```

### Fix: Split monolith CLAUDE.md

When CLAUDE.md is >100 lines with no links, propose this restructure:

```
CLAUDE.md (60 lines max)
├── Build/Dev commands (top — most frequently needed)
├── Critical Rules (5-10 non-negotiable rules)
├── Links: "For architecture details, see docs/architecture.md"
├── Links: "For API conventions, see docs/api-conventions.md"  
└── Links: "For module-specific rules, see each module's CLAUDE.md"

docs/architecture.md ← extracted from CLAUDE.md
docs/api-conventions.md ← extracted from CLAUDE.md
src/module-a/CLAUDE.md ← module-specific rules
src/module-b/CLAUDE.md ← module-specific rules
```

---

## Harness Vitality Formula

```
Vitality = harness commits (30d) / total commits (30d) × 100%

  0%     = Flywheel stopped. Harness is a static artifact.
  1-5%   = Minimal maintenance. Likely reactive only.
  5-15%  = Healthy. Error→rule conversion happening.
  >15%   = Active harness engineering phase.
```
