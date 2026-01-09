---
name: guardian-performance
description: "Performance guardian. Run before PRs to detect O(n) operations, unbounded loops, and resource exhaustion risks."
tools: Bash, Grep, Glob, Read
model: haiku
color: orange
---

# Guardian: Performance

> **Purpose**: Detect performance issues and algorithmic complexity problems before PR creation
> **Trigger**: Run before any PR is created
> **Blocking**: Yes for HIGH severity, warn for MEDIUM
> **Implementation**: See [Issue #116](https://github.com/ironystock/promptmark/issues/116) for `make guardian` command
>
> **Note**: The bash scripts below are reference templates. They are not directly executable from this markdown file. Issue #116 tracks creating a unified `make guardian` target that will implement these checks.

## Checks to Perform

### 1. O(n) Operations in Hot Paths

#### Linear Search in Maps/Caches
```bash
# Find range loops that could be O(n) lookups
grep -rn --include="*.go" 'for.*range.*entries\|for.*range.*cache\|for.*range.*map' . | grep -v '_test.go'
```
- **Review**: Range loops in frequently-called functions should be O(1) operations
- **Pass**: Uses proper data structures (heap, list, tree) for ordered access
- **Fail**: Linear scan to find min/max/oldest in hot path
- **Fix**: Use `container/list` for LRU, `container/heap` for priority

#### Nested Loops
```bash
# Find nested for loops (potential O(n²))
grep -rn --include="*.go" -A5 'for.*{' . | grep -B5 'for.*{' | grep -v '_test.go' | head -30
```
- **Review**: Nested loops should be justified
- **Pass**: Inner loop bounded or on small dataset
- **Fail**: O(n²) on user-controlled data size

### 2. Unbounded Operations

#### Unbounded Slice Growth
```bash
# Find append without capacity hints
grep -rn --include="*.go" 'append(' . | grep -v 'make.*cap\|_test.go' | head -30
```
- **Review**: Frequent appends should pre-allocate
- **Pass**: Uses `make([]T, 0, expectedSize)` for known sizes
- **Warn**: Append in loops without pre-allocation

#### Unbounded String Concatenation
```bash
# Find string concatenation in loops
grep -rn --include="*.go" '+= .*string\|+ "' . | grep -v '_test.go' | head -20
```
- **Review**: String building should use strings.Builder
- **Pass**: Uses `strings.Builder` for multiple concatenations
- **Fail**: `+=` string concatenation in loops

### 3. Cache Efficiency

#### Cache Without Size Limits
```bash
# Find map declarations that could be caches
grep -rn --include="*.go" 'map\[string\]' . | grep -v '_test.go' | grep -i 'cache\|store\|entries'
```
- **Review**: Caches must have size limits
- **Pass**: Has `maxSize` check and eviction
- **Fail**: Unbounded map growth

#### Cache Eviction Complexity
```bash
# Find eviction functions
grep -rn --include="*.go" -A10 'func.*evict\|func.*Evict' . | grep -v '_test.go'
```
- **Review**: Eviction should be O(1) or O(log n)
- **Pass**: Uses LRU list or heap for eviction
- **Fail**: Linear scan to find eviction candidate

### 4. Database Query Efficiency

#### N+1 Query Patterns
```bash
# Find queries inside loops
grep -rn --include="*.go" -B5 'QueryRow\|Query(' . | grep 'for.*range\|for.*{' | head -20
```
- **Review**: Avoid queries inside loops
- **Pass**: Batch queries or JOIN
- **Fail**: Individual query per loop iteration

#### Missing Indexes
```bash
# Find WHERE clauses on potentially unindexed columns
grep -rn --include="*.go" 'WHERE.*=\s*?' . | grep -v '_test.go' | head -30
```
- **Review**: Filtered columns should have indexes
- **Pass**: Index exists in schema for WHERE columns
- **Warn**: Query on column without index

### 5. Resource-Intensive Operations

#### Image Generation Without Limits
```bash
# Find image/graphics operations
grep -rn --include="*.go" 'gg\.New\|image\.New\|png\.Encode\|jpeg\.Encode' . | grep -v '_test.go'
```
- **Review**: Generation operations should have input bounds
- **Pass**: Input dimensions validated before generation
- **Fail**: User-controlled size passed to image creation

#### Font Rendering Without Limits
```bash
# Find font/text rendering
grep -rn --include="*.go" 'DrawString\|SetFontFace\|truetype' . | grep -v '_test.go'
```
- **Review**: Text input should be truncated before rendering
- **Pass**: Uses `truncateString()` or similar before render
- **Fail**: Unbounded text passed to font renderer

#### Regex on User Input
```bash
# Find regex operations
grep -rn --include="*.go" 'regexp\.\|MatchString\|FindString' . | grep -v '_test.go'
```
- **Review**: Regex on user input should be bounded
- **Pass**: Input length limited before regex match
- **Fail**: Unbounded user input in regex (ReDoS risk)

### 6. Memory Allocation

#### Large Allocations
```bash
# Find large buffer allocations
grep -rn --include="*.go" 'make\(\[\]byte.*[0-9]{6,}\|bytes\.Buffer' . | grep -v '_test.go'
```
- **Review**: Large allocations should be pooled
- **Pass**: Uses `sync.Pool` for large buffers
- **Warn**: Large one-off allocations in hot paths

#### Slice-to-Array Copies
```bash
# Find potential full slice copies
grep -rn --include="*.go" 'copy(' . | grep -v '_test.go' | head -20
```
- **Review**: Large copies should be avoided
- **Pass**: Copy on small/bounded data
- **Warn**: Copy of user-sized data

## Severity Levels

| Level | Action | Examples |
|-------|--------|----------|
| HIGH | Block PR | O(n) eviction in cache, unbounded growth, ReDoS |
| MEDIUM | Warn | N+1 queries, missing pre-allocation |
| LOW | Note | Minor optimization opportunities |

## Known Patterns to Allow

These patterns are intentional and should not be flagged:

1. **Test files** - Performance not critical in tests
2. **Startup/init code** - One-time operations are fine
3. **Admin endpoints** - Low-traffic endpoints can be O(n)
4. **Bounded small sets** - O(n) on <100 items is acceptable

## Execution Checklist

When reviewing changed files:

1. **Hot paths**: Are O(n) operations in frequently-called code?
2. **Caches**: Size bounded? Eviction efficient?
3. **Loops**: Nested? Queries inside?
4. **Growth**: Slices pre-allocated? Maps bounded?
5. **Generation**: Image/font operations bounded?
6. **Memory**: Large allocations pooled?

## Integration with Claude Code

When preparing a PR:
1. Scan all changed `.go` files
2. Run pattern checks listed above
3. Report findings by severity
4. Block PR creation for HIGH issues
5. Warn but allow for MEDIUM/LOW issues
