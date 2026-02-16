---
description: "Run the full Pre-PR Guardian Cycle"
allowed-tools: Bash, Read, Grep, Glob, Task
user-invocable: true
---

# Pre-PR Guardian Cycle

Run all guardian agents to validate the codebase before creating a PR. This is a BLOCKING requirement - do not create a PR until all guardians pass or issues are addressed.

## Execution Plan

### Step 1: Build Verification
First, verify the project builds cleanly:
```bash
docker exec promptmark-dev-1 make build
```
If this fails, stop immediately and report the build errors.

### Step 2: Run Guardian Agents
Invoke each guardian agent as a subagent task. Run independent guardians in parallel where possible:

**Parallel batch 1** (can run simultaneously):
- `guardian-code` (sonnet) - Go format, vet, build, test compliance
- `guardian-docs` (sonnet) - Missing doc comments, README, CHANGELOG
- `guardian-context` (sonnet) - File sizes, codebase navigability

**Parallel batch 2** (can run simultaneously):
- `guardian-security` (opus) - SQL injection, XSS, CSRF, OWASP
- `guardian-logic` (opus) - State transitions, activity tracking, per-user DB
- `guardian-performance` (sonnet) - O(n) operations, unbounded loops

**Parallel batch 3** (if web/MCP changes detected):
- `guardian-mcp` (sonnet) - MCP server sync with web functionality

### Step 3: Unified Report
After all guardians complete, produce a summary:

```
=== Pre-PR Guardian Cycle ===

guardian-code:        PASS / FAIL (details)
guardian-security:    PASS / FAIL (details)
guardian-logic:       PASS / FAIL (details)
guardian-performance: PASS / FAIL / WARN (details)
guardian-mcp:        PASS / FAIL / SKIP (details)
guardian-docs:       PASS / WARN (details)
guardian-context:    PASS / WARN (details)

Overall: PASS / FAIL
```

**FAIL** = at least one blocking guardian failed. Do NOT proceed with PR.
**PASS** = all guardians passed (warnings are acceptable).

When invoking guardian agents, do NOT pass a `model` parameter - let them use their configured models from their YAML frontmatter.
