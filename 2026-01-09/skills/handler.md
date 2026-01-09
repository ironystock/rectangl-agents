---
description: Scaffold a new Gin handler with standard patterns (auth, params, body, response)
argument-hint: <handlerName> <METHOD> </path>
allowed-tools: Read, Edit, Write, Grep, Glob
---

Create a new Gin handler following Promptmark's established patterns.

Arguments: $ARGUMENTS
- Handler name (e.g., `archivePrompt`)
- HTTP method (GET, POST, PUT, DELETE)
- Route path (e.g., `/api/prompts/:id/archive`)

## Handler Template

```go
// {handlerName}Handler handles {METHOD} {path}
func (s *Server) {handlerName}Handler(c *gin.Context) {
    // 1. Auth check (if protected endpoint)
    userID := c.GetString("userID")
    if userID == "" {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Unauthorized"})
        return
    }

    // 2. Parse path params (if any :params in path)
    id := c.Param("id")
    if id == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "ID required"})
        return
    }

    // 3. Parse request body (for POST/PUT)
    // var req RequestType
    // if err := c.ShouldBindJSON(&req); err != nil {
    //     c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid request"})
    //     return
    // }

    // 4. Business logic / database call
    // result, err := s.db.DoSomething(...)
    // if err != nil {
    //     log.Printf("Error doing something: %v", err)
    //     c.JSON(http.StatusInternalServerError, gin.H{"error": "Operation failed"})
    //     return
    // }

    // 5. Response (HTMX or JSON)
    if isHTMXRequest(c) {
        // _ = components.SomeComponent(result).Render(c.Request.Context(), c.Writer)
        return
    }
    c.JSON(http.StatusOK, gin.H{"success": true})
}
```

## Reference Files

Check these for patterns:
- @internal/server/social.go - Like/favorite handlers (simple CRUD)
- @internal/server/prompts_update.go - Complex update handlers
- @internal/server/admin_curation.go - Admin-protected handlers

## Steps

1. Determine which file this handler belongs in (or create new file)
2. Write the handler following the template above
3. Add the route in `internal/server/routes.go`
4. If new DB method needed, add to `internal/database/` and `mocks/database_mock.go`

Now scaffold the handler for: $ARGUMENTS
