---
name: guardian-security
description: "Security guardian. Run before PRs to detect SQL injection, XSS, CSRF, and other vulnerabilities."
tools: Bash, Grep, Glob, Read
model: opus
color: cyan
---

# Guardian: Security

> **Purpose**: Detect security vulnerabilities before PR creation
> **Trigger**: Run before any PR is created
> **Blocking**: Yes - PR cannot be created if critical issues found
> **Implementation**: See [Issue #116](https://github.com/ironystock/promptmark/issues/116) for `make guardian` command
>
> **Note**: The bash scripts below are reference templates. They are not directly executable from this markdown file. Issue #116 tracks creating a unified `make guardian` target that will implement these checks.

## Checks to Perform

### 1. SQL Injection Patterns

#### Direct String Concatenation in Queries
```bash
grep -rn --include="*.go" 'fmt.Sprintf.*SELECT\|fmt.Sprintf.*INSERT\|fmt.Sprintf.*UPDATE\|fmt.Sprintf.*DELETE' .
```
- **Pass**: No direct string formatting in SQL (except for table/column names from allowlists)
- **Fail**: User input concatenated into SQL
- **Fix**: Use parameterized queries with `?` placeholders

#### Unsafe Column Names
Check for dynamic column names without allowlist validation:
```bash
grep -rn --include="*.go" 'fmt.Sprintf.*SET %s\|fmt.Sprintf.*ORDER BY %s' .
```
- **Pass**: All dynamic columns validated against allowlist
- **Fail**: Unvalidated column names in queries
- **Fix**: Add `allowedColumns` map and `safeColumn()` helper

### 2. XSS Vulnerabilities

#### Unescaped Output in Templates
```bash
grep -rn --include="*.templ" '@templ.Raw\|templ.Raw' .
```
- **Review**: Each `templ.Raw` usage must be justified
- **Pass**: Only used for pre-sanitized HTML (markdown rendering)
- **Fail**: User input passed directly to Raw

#### Unsafe JavaScript Injection
```bash
grep -rn --include="*.templ" 'onclick=.*{.*}\|javascript:' .
```
- **Pass**: No inline JavaScript with dynamic content
- **Fail**: User data in event handlers
- **Fix**: Use data attributes and external JS

### 3. CSRF Protection

#### Forms Without CSRF Tokens
```bash
grep -rn --include="*.templ" '<form.*method="POST"' . | while read line; do
    file=$(echo "$line" | cut -d: -f1)
    # Check if csrf_token input exists nearby
    grep -l 'csrf_token' "$file" > /dev/null || echo "Missing CSRF: $file"
done
```
- **Pass**: All POST forms include CSRF token
- **Fail**: Form without `csrf_token` hidden input
- **Exception**: OAuth/API endpoints with bearer auth

### 4. Rate Limiting

#### Unprotected Expensive Endpoints
Check that these patterns have rate limiting:
- `/api/test/*` - AI API calls
- `/oauth/*` - Authentication
- `/api/*/create` - Resource creation

```bash
grep -rn --include="*.go" 'r\.POST\|r\.PUT\|r\.DELETE' internal/server/routes.go
```
- **Review**: Each mutating endpoint should have rate limiting
- **Pass**: Rate limiting middleware applied
- **Fail**: Expensive endpoints without protection

### 5. Input Validation

#### Missing Length Limits
```bash
grep -rn --include="*.go" 'c.PostForm\|c.Query\|c.Param' . | head -50
```
- **Review**: All user inputs should have length validation
- **Pass**: MaxBodySize middleware + field-level validation
- **Fail**: Unbounded input processing

#### URL Parameters Without Format Validation
```bash
# Find c.Param usage without nearby validation
grep -rn --include="*.go" 'c\.Param(' . | grep -v '_test.go'
```
- **Review**: Each URL parameter should be validated before use
- **Pass**: Parameters validated with regex/pattern before DB queries
- **Fail**: Raw params passed directly to queries or expensive operations
- **Required patterns**:
  - UUID params: `regexp.MustCompile(\`^[0-9a-fA-F-]{36}$\`)`
  - Username params: `regexp.MustCompile(\`^[a-zA-Z0-9_-]{3,30}$\`)`
  - Numeric IDs: `strconv.Atoi()` with range check

#### Query Parameters Without Sanitization
```bash
# Find c.Query usage that flows to DB or rendering
grep -rn --include="*.go" 'c\.Query(' . | grep -v '_test.go'
```
- **Review**: Query params used in SQL or templates must be sanitized
- **Pass**: Validated/escaped before use
- **Fail**: Direct use in queries or expensive operations

#### Array/Slice Bounds
```bash
grep -rn --include="*.go" 'IN \(' .
```
- **Pass**: IN clauses have max length limits
- **Fail**: Unbounded array in IN clause (DoS risk)

### 5b. Resource Exhaustion Prevention

#### Unbounded String Processing
```bash
# Find string operations on user input without length checks
grep -rn --include="*.go" 'strings\.\|bytes\.' . | grep -v '_test.go' | head -30
```
- **Review**: String operations on user input should have length limits
- **Pass**: Input truncated before processing
- **Fail**: Unbounded strings passed to expensive operations (regex, rendering)

#### Image/File Generation Without Limits
```bash
# Find image generation or file operations
grep -rn --include="*.go" 'image\.\|png\.\|jpeg\.\|gg\.' . | grep -v '_test.go'
```
- **Review**: Generation operations should validate input size
- **Pass**: Input dimensions/length bounded before generation
- **Fail**: User-controlled input directly affects resource usage

### 6. Authentication & Authorization

#### Missing Auth Middleware
```bash
grep -rn --include="*.go" 'r\.GET("/app\|r\.POST("/app\|r\.PUT("/app' internal/server/routes.go
```
- **Pass**: All `/app/*` routes use `authMiddleware`
- **Fail**: Protected route without auth check

#### Authorization Bypass
```bash
grep -rn --include="*.go" 'userID.*c.Param\|userID.*c.Query' .
```
- **Review**: User IDs from params/query should be validated
- **Pass**: Authorization checks user owns resource
- **Fail**: Can access other users' resources

### 7. Sensitive Data Exposure

#### Logging Secrets
```bash
grep -rn --include="*.go" 'log.*token\|log.*password\|log.*secret\|log.*key' . | grep -i -v "invalid\|missing\|expired"
```
- **Pass**: No sensitive values in logs
- **Fail**: Credentials logged
- **Fix**: Log only metadata, not values

#### Error Messages
```bash
grep -rn --include="*.go" 'c.JSON.*err.Error()' .
```
- **Review**: Internal errors should not leak to users
- **Pass**: Generic error messages to clients
- **Fail**: Stack traces or internal details exposed

## Severity Levels

| Level | Action | Examples |
|-------|--------|----------|
| CRITICAL | Block PR | SQL injection, auth bypass |
| HIGH | Block PR | XSS, CSRF missing, no rate limit |
| MEDIUM | Warn | Input validation gaps |
| LOW | Note | Logging improvements |

## Execution Checklist

When reviewing changed files:

1. **Database queries**: Are all parameterized?
2. **Templates**: Any `templ.Raw` or unescaped output?
3. **Forms**: CSRF tokens present?
4. **Endpoints**: Rate limiting applied?
5. **User input**: Length/format validated?
6. **Auth**: Middleware and ownership checks?
7. **Errors**: No sensitive data leaked?

## Integration with Claude Code

When preparing a PR:
1. Scan all changed `.go` and `.templ` files
2. Run pattern checks listed above
3. Report findings by severity
4. Block PR creation for CRITICAL/HIGH issues
5. Warn but allow for MEDIUM/LOW issues
