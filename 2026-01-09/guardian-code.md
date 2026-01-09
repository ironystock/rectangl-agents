---
name: guardian-code
description: "Code quality guardian. Run before PRs to validate Go format, vet, build, and test compliance."
tools: Bash, Grep, Glob, Read
model: opus
color: red
---

# Guardian: Code Quality

> **Purpose**: Validate code quality standards before PR creation
> **Trigger**: Run before any PR is created
> **Blocking**: Yes - PR cannot be created if checks fail
> **Implementation**: See [Issue #116](https://github.com/ironystock/promptmark/issues/116) for `make guardian` command
>
> **Note**: The bash scripts below are reference templates. They are not directly executable from this markdown file. Issue #116 tracks creating a unified `make guardian` target that will implement these checks.

## Checks to Perform

### 1. Go Format Compliance
```bash
gofmt -l .
```
- **Pass**: No output (all files formatted)
- **Fail**: Any files listed need formatting
- **Fix**: Run `gofmt -w .` or `make fmt`

### 2. Go Vet
```bash
go vet ./...
```
- **Pass**: No errors
- **Fail**: Any vet warnings
- **Fix**: Address each warning before proceeding

### 3. Build Success
```bash
go build ./...
```
- **Pass**: Clean build with no errors
- **Fail**: Compilation errors
- **Fix**: Resolve all build errors

### 4. Test Suite
```bash
go test ./...
```
- **Pass**: All tests pass
- **Fail**: Any test failures
- **Fix**: Fix failing tests or update expectations

### 5. No Hardcoded Secrets
Scan for patterns that suggest hardcoded secrets:
```bash
grep -rn --include="*.go" -E "(password|secret|api_key|apikey|token)\s*[:=]\s*[\"'][^\"']{8,}" . \
    | grep -v "_test.go" \
    | grep -v "example" \
    | grep -v "os\.Getenv" \
    | grep -v "// " \
    | grep -v "const ("
```
- **Pass**: No matches (or only false positives)
- **Fail**: Any hardcoded credentials found
- **Fix**: Move to environment variables

**Expected false positives** (review manually):
- Variable names containing keywords (e.g., `jwtSecret := os.Getenv(...)`)
- Configuration struct field names
- Error message strings containing keywords
- Documentation comments

### 6. No TODO/FIXME in New Code
Check staged files for unresolved markers:
```bash
git diff --cached --name-only | xargs grep -l "TODO\|FIXME" 2>/dev/null
```
- **Pass**: No TODOs in new/modified code (existing ones are grandfathered)
- **Fail**: New TODOs added without issue reference
- **Exception**: `TODO(#123)` format with issue number is allowed

### 7. Import Organization
Verify imports follow Go conventions:
- Standard library first
- Third-party packages second
- Local packages last
- Grouped with blank lines

### 8. Error Handling Patterns
Check for common anti-patterns:
```bash
grep -rn --include="*.go" "_ = err" . | grep -v "_test.go"
grep -rn --include="*.go" "err != nil { return }" . | grep -v "_test.go"
```
- **Warn**: Ignored errors should be intentional and commented
- **Warn**: Empty error returns should include error message

### 9. Goroutine Safety

#### Missing Panic Recovery
```bash
# Find goroutines without panic recovery
grep -rn --include="*.go" 'go func\|go [a-z]*\.' . | grep -v '_test.go' | grep -v 'go.mod\|go.sum'
```
- **Review**: Each goroutine should have panic recovery
- **Pass**: Goroutines wrap body with `defer func() { recover() }()`
- **Fail**: Naked goroutines that could crash silently
- **Pattern required**:
```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("panic recovered: %v", r)
        }
    }()
    // goroutine body
}()
```

#### Goroutine Leaks
```bash
# Find goroutines without termination conditions
grep -rn --include="*.go" 'for {' . | grep -v '_test.go'
```
- **Review**: Infinite loops in goroutines should have exit conditions
- **Pass**: Loop has `select` with context cancellation or ticker
- **Fail**: Infinite loop without shutdown mechanism

### 10. Migration Patterns

#### Missing Migration Tracking
```bash
# Find migrations that check row counts for idempotency
grep -rn --include="*.go" 'COUNT\(\*\).*== 0\|COUNT\(\*\).*== "0"' . | grep -i migrat
```
- **Review**: Migrations should use tracking table, not row counts
- **Pass**: Uses `migrations_log` table or similar marker
- **Fail**: Checks `COUNT(*) == 0` which fails on empty tables
- **Pattern required**:
```go
// Good: Check migration marker
s.db.QueryRow("SELECT COUNT(*) FROM migrations_log WHERE name = ?", migrationName)

// Bad: Check row count
s.db.QueryRow("SELECT COUNT(*) FROM target_table")
```

## Execution Script

```bash
#!/bin/bash
set -e

echo "=== Guardian: Code Quality ==="

echo "1. Checking gofmt..."
if [ -n "$(gofmt -l .)" ]; then
    echo "FAIL: Files need formatting:"
    gofmt -l .
    exit 1
fi
echo "PASS: All files formatted"

echo "2. Running go vet..."
if ! go vet ./...; then
    echo "FAIL: go vet found issues"
    exit 1
fi
echo "PASS: go vet clean"

echo "3. Building..."
if ! go build ./...; then
    echo "FAIL: Build failed"
    exit 1
fi
echo "PASS: Build successful"

echo "4. Running tests..."
if ! go test ./...; then
    echo "FAIL: Tests failed"
    exit 1
fi
echo "PASS: All tests pass"

echo "=== Code Quality: PASSED ==="
```

## When to Skip

This guardian should NEVER be skipped. All code quality checks are mandatory.

## Integration with Claude Code

When preparing a PR:
1. Run all checks listed above
2. Report any failures to the user
3. Suggest fixes for each failure
4. Only proceed with PR creation when all checks pass
