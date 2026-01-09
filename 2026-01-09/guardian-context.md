---
name: guardian-context
description: "Context management guardian. Run before PRs to check file sizes and codebase navigability for AI assistants."
tools: Bash, Grep, Glob, Read
color: yellow
---

# Guardian: Context Management

> **Purpose**: Keep codebase navigable and context-efficient for AI assistants
> **Trigger**: Run before any PR is created
> **Blocking**: Warn only - context issues don't block PRs but should be addressed
> **Implementation**: See [Issue #116](https://github.com/ironystock/promptmark/issues/116) for `make guardian` command
>
> **Note**: The bash scripts below are reference templates. They are not directly executable from this markdown file. Issue #116 tracks creating a unified `make guardian` target that will implement these checks.

## Core Principle

**Large files make AI assistance harder.** Keep files focused and reasonably sized so Claude Code can effectively navigate and understand the codebase.

## Checks to Perform

### 1. File Size Limits

#### Large Files Warning
```bash
find . -name "*.go" -size +50k | grep -v vendor | grep -v _test.go
find . -name "*.templ" -size +30k
```

| File Type | Warning | Critical | Action |
|-----------|---------|----------|--------|
| `.go` | >1000 lines | >1500 lines | Split into focused files |
| `.templ` | >500 lines | >800 lines | Extract components |
| `_test.go` | >1500 lines | >2000 lines | Split by functionality |

Current large files to monitor:
```bash
wc -l internal/server/*.go | sort -rn | head -10
wc -l internal/mcp/*.go | sort -rn | head -10
wc -l cmd/web/**/*.templ | sort -rn | head -10
```

### 2. File Organization

#### Single Responsibility
Each file should have one clear purpose:
- `prompts_crud.go` - Create/Read/Update/Delete handlers
- `prompts_pages.go` - Page rendering handlers
- `prompts_versions.go` - Version-specific handlers

```bash
# Check for files with too many handler types
grep -c "func.*Handler\|func.*handler" internal/server/*.go | awk -F: '$2 > 15 {print}'
```
- **Pass**: <15 handlers per file
- **Warn**: 15-25 handlers
- **Split**: >25 handlers

### 3. Import Complexity

#### Circular or Heavy Imports
```bash
# Check import counts per file
for f in internal/server/*.go; do
    count=$(grep -c "import\|\"" "$f" 2>/dev/null || echo 0)
    echo "$count $f"
done | sort -rn | head -10
```
- **Pass**: <20 imports per file
- **Warn**: 20-30 imports
- **Review**: >30 imports (may need splitting)

### 4. CLAUDE.md Currency

#### Project Instructions Up to Date
Check that CLAUDE.md reflects current architecture:
```bash
# Commands documented match Makefile
grep "make " CLAUDE.md
grep -E "^[a-z-]+:" Makefile | cut -d: -f1

# Package descriptions current
grep "internal/" CLAUDE.md
ls internal/
```
- **Pass**: Documented commands work, packages listed
- **Warn**: New packages not mentioned
- **Update**: CLAUDE.md when adding packages

### 5. Agent Documentation

#### Agents.md Accuracy
If `.claude/agents/` files exist, ensure they're current:
```bash
ls .claude/agents/*.md
```

Check that agent descriptions match their purpose and tools.

### 6. Dead Code Detection

#### Unused Exports
```bash
# Find exported functions
grep -rh "^func [A-Z]" internal/ | cut -d'(' -f1 | sort -u > /tmp/exports.txt

# Check for usage (rough heuristic)
while read func; do
    name=$(echo "$func" | awk '{print $2}')
    count=$(grep -r "$name" internal/ cmd/ | grep -v "^func" | wc -l)
    [ "$count" -eq 0 ] && echo "Possibly unused: $name"
done < /tmp/exports.txt
```
- **Review**: Functions with no apparent callers
- **Pass**: All exports used
- **Clean**: Remove verified dead code

### 7. Test Coverage Gaps

#### Untested Handlers
```bash
# List handlers
grep -rh "func.*Handler" internal/server/*.go | grep -v "_test.go" | wc -l

# List test functions
grep -rh "func Test" internal/server/*_test.go | wc -l
```
- **Track**: Handler count vs test count ratio
- **Goal**: Critical handlers tested
- **Warn**: New handlers without tests

## File Splitting Guidelines

### When to Split

1. **File >1000 lines**: Almost always split
2. **Multiple concerns**: CRUD + pages + utils = 3 files
3. **Test isolation**: Split tests with distinct fixtures
4. **Reusability**: Extract shared helpers

### How to Split

```
Before: routes.go (2000 lines)
After:
├── routes.go (300 lines) - Route registration only
├── middleware.go (200 lines) - Middleware functions
├── auth_handlers.go (400 lines) - Auth endpoints
├── page_handlers.go (300 lines) - HTML page handlers
├── dashboard_handlers.go (250 lines) - Dashboard specific
└── profile_handlers.go (350 lines) - Profile/settings
```

### Naming Conventions

| Pattern | Purpose |
|---------|---------|
| `*_handlers.go` | HTTP handlers for a domain |
| `*_pages.go` | Page rendering handlers |
| `*_crud.go` | Create/Read/Update/Delete |
| `*_test.go` | Tests for corresponding file |
| `types.go` | Shared type definitions |
| `helpers.go` | Utility functions |

## Context Budget

AI assistants have limited context windows. Help them by:

1. **Focused files**: One concern per file
2. **Clear naming**: File name = content purpose
3. **Logical grouping**: Related code together
4. **Good comments**: Intent, not mechanics
5. **Updated docs**: CLAUDE.md reflects reality

## Integration with Claude Code

When preparing a PR:
1. Check line counts of modified files
2. Identify files approaching limits
3. Suggest splits for oversized files
4. Verify CLAUDE.md still accurate
5. Warn about context-heavy changes
6. Offer to help split large files
