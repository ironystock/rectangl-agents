---
name: guardian-mcp
description: "MCP sync guardian. Run before PRs to ensure MCP server stays synchronized with web functionality."
tools: Bash, Grep, Glob, Read
model: opus
color: blue
---

# Guardian: MCP Sync

> **Purpose**: Ensure MCP server stays synchronized with web functionality
> **Trigger**: Run before any PR that touches prompts, collections, or templates
> **Blocking**: Yes for feature parity issues, Warn for schema differences
> **Implementation**: See [Issue #116](https://github.com/ironystock/promptmark/issues/116) for `make guardian` command
>
> **Note**: The bash scripts below are reference templates. They are not directly executable from this markdown file. Issue #116 tracks creating a unified `make guardian` target that will implement these checks.

## Core Principle

**Every CRUD operation available in the web UI must be available via MCP tools.**

Users should be able to manage their Promptmark data entirely through AI assistants using MCP, without needing the web interface.

## Checks to Perform

### 1. Feature Parity

#### Web Handlers vs MCP Tools
Compare operations available:

| Entity | Web Operations | MCP Tools Required |
|--------|----------------|-------------------|
| Prompts | List, Create, Read, Update, Delete | `list_prompts`, `create_prompt`, `get_prompt`, `update_prompt`, `delete_prompt` |
| Collections | List, Create, Read, Update, Delete | `list_collections`, `create_collection`, `get_collection`, `update_collection`, `delete_collection` |
| Templates | List, Use, Render | `list_templates`, `get_prompt_schema`, `render_prompt`, `validate_prompt_inputs` |
| Tags | List, Add, Remove | `list_tags`, `add_tag`, `remove_tag` |
| Versions | List, Restore | `get_prompt_versions`, `restore_prompt_version` |
| Remixes | List, Create | `list_remixes`, (create via `create_prompt` with `remixed_id`) |

When modifying web handlers:
```bash
git diff --cached internal/server/prompts*.go internal/server/collections.go internal/server/template_handlers.go
```
- **Check**: Does MCP need corresponding update?
- **Pass**: MCP tools cover same functionality
- **Fail**: Web-only feature without MCP equivalent

### 2. Schema Consistency

#### Request/Response Fields
MCP tool arguments should match web form fields:

```go
// Web: prompts.go
type CreatePromptRequest struct {
    Title       string
    Prompt      string
    Notes       string
    Tags        []string
    IsPublic    bool
    TypeID      string
    CollectionID string
}

// MCP: handlers_prompts.go - createPromptHandler
// Should accept same fields
```

When adding new fields to web:
```bash
git diff --cached cmd/web/types.go internal/server/prompts*.go | grep "type\|struct\|json:"
```
- **Check**: New fields added to MCP tool arguments?
- **Pass**: Same fields in both
- **Warn**: Web has fields MCP doesn't expose

#### 2.5 Database Schema Parity
When web handlers add new database columns, MCP tools should expose them:

```bash
# Check for schema changes in the PR
git diff --cached internal/database/database.go | grep "CREATE TABLE\|ALTER TABLE\|ADD COLUMN"
```

If schema changes detected:
- **Check**: Do MCP handlers query/update the new columns?
- **Pass**: New columns accessible via MCP tools
- **Warn**: New features (e.g., `is_featured`, `view_count`) not exposed in MCP

Common missed columns:
- New prompt fields (type_id, template_vars, version, etc.)
- New collection fields
- User preference fields
- Activity/analytics columns

### 3. Validation Consistency

#### Same Rules in Both Places
Validation logic should match:
- Title length limits
- Tag format/count limits
- Required fields
- Enum values (type_id, etc.)

```bash
# Find validation in web handlers
grep -rn "validation\|required\|max\|min\|limit" internal/server/prompts*.go

# Find validation in MCP handlers
grep -rn "validation\|required\|max\|min\|limit" internal/mcp/handlers_prompts.go
```
- **Pass**: Same validation rules
- **Fail**: Different limits or requirements

### 4. Error Handling

#### Consistent Error Messages
Error responses should be similar:
```go
// Web
c.JSON(http.StatusBadRequest, gin.H{"error": "title is required"})

// MCP
return &mcp.CallToolResult{
    IsError: true,
    Content: []mcp.Content{&mcp.TextContent{Text: "title is required"}},
}
```

### 5. Resource Availability

#### MCP Resources Match Data Model
```bash
# Check MCP resources registration
grep -n "AddResource" internal/mcp/resources.go
```

Resources should expose:
- `prompt://{id}` - Individual prompts
- `collection://{id}` - Collections
- `template://{id}` - Templates

### 6. Tool Registration

#### All Handlers Have Tools
```bash
# Count handlers vs tools
grep -c "Handler" internal/mcp/handlers_*.go
grep -c "RegisterTool\|AddTool" internal/mcp/tools.go
```
- **Pass**: All handlers registered as tools
- **Fail**: Orphan handler without tool registration

## Sync Checklist

When modifying these files, check MCP:

| Web File | MCP File to Check |
|----------|-------------------|
| `prompts.go`, `prompts_*.go` | `handlers_prompts.go` |
| `collections.go` | `handlers_collections.go` |
| `template_handlers.go` | `handlers_prompts.go` (template tools) |
| `testing.go`, `testing_*.go` | (No MCP equivalent yet) |

## MCP Tool Inventory

Current tools that must stay in sync:

### Prompts
- `list_prompts` - Match filters from web list view
- `get_prompt` - Return same fields as detail view
- `create_prompt` - Accept same fields as create form
- `update_prompt` - Accept same fields as edit form
- `delete_prompt` - Same soft-delete behavior

### Collections
- `list_collections` - Match web sidebar/list
- `get_collection` - Include prompt count
- `create_collection` - Name, description, icon, color
- `update_collection` - Same editable fields
- `delete_collection` - Cascade behavior matches

### Templates
- `get_prompt_schema` - Variable definitions
- `render_prompt` - Substitution logic matches
- `validate_prompt_inputs` - Same validation

### Tags
- `list_tags` - All tags with counts
- `add_tag` - Add to prompt
- `remove_tag` - Remove from prompt
- `bulk_add_tag` - Match bulk operations

### Versions
- `get_prompt_versions` - Version history
- `restore_prompt_version` - Restore functionality

## Integration with Claude Code

When preparing a PR that touches CRUD operations:
1. Identify affected entities (prompts, collections, etc.)
2. Check if MCP handlers need updates
3. Verify schema consistency
4. Test MCP tools still work after changes
5. Block PR if feature parity broken
6. Warn if new web features lack MCP support
