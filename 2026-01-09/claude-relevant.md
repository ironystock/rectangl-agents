**Preferred: Use slash commands** (`.claude/commands/`):
- `/build` - Build in Docker container
- `/test` - Run all tests in Docker
- `/test TestName` - Run specific test
### Frontend Interactivity (MANDATORY: Use horsehead-memelord Agent)
- **ALL web interactivity MUST go through the `horsehead-memelord` agent** (`.claude/agents/horsehead-memelord.md`)
- Before adding ANY JavaScript or dynamic behavior to templates, invoke this agent to design the HTMX solution
- **HTMX-first approach**: Use HTMX for all dynamic behavior unless it absolutely cannot perform the required function
- If JavaScript seems necessary, the horsehead-memelord agent will determine if it's truly needed
- HTMX docs: https://htmx.org/docs/ and https://htmx.org/examples/
- Key patterns:
  - `hx-get/post/put/delete` for all server interactions
  - `hx-target` and `hx-swap` for DOM updates
  - `hx-swap-oob` for updating multiple elements (e.g., hidden inputs + visible content)
  - `hx-trigger` with modifiers (`delay`, `changed`, `from`) for event handling
  - `hx-vals` for sending additional data with requests
- Exceptions (require JavaScript - must be confirmed by horsehead-memelord):
  - Complex animations that can't be achieved with CSS transitions
  - Third-party library integrations (e.g., drag-and-drop with Sortable.js)
  - Client-side-only calculations with no server round-trip justification

### Frontend Implementation (MANDATORY: Use front-ender Agent)
- **ALL frontend implementation work MUST go through the `front-ender` agent** (`.claude/agents/front-ender.md`)
- Front-ender is a sorcerer-level UX engineer with write access - they implement, not just advise
- Use front-ender for:
  - Implementing new UI components or pages
  - Updating templates to use design system patterns
  - Fixing design drift or inconsistencies
  - Any templ file modifications involving styling
- Front-ender enforces `docs/DESIGN_SYSTEM.md` and uses `cmd/web/components.templ`
- **Slash command**: `/component <Name> [type]` - Scaffold templ components

### Backend Implementation (MANDATORY: Use back-ender Agent)
- **ALL backend implementation work MUST go through the `back-ender` agent** (`.claude/agents/back-ender.md`)
- Back-ender is a 1% Golang deity with write access - they ship production code
- Use back-ender for:
  - New Gin handlers and API endpoints
  - Database schema changes and migrations
  - Query optimization and data layer work
  - Any Go code in `internal/` packages
- Back-ender enforces Go idioms, Gin patterns, and SQLite/Turso best practices
- **Slash command**: `/handler <name> <METHOD> </path>` - Scaffold Gin handlers

### Design & Aesthetics (MANDATORY: Use snooty-mcfigmaface Agent)
- **ALL design and aesthetic decisions MUST go through the `snooty-mcfigmaface` agent** (`.claude/agents/snooty-mcfigmaface.md`)
- This project uses **Neo-Brutalist design** - bold typography, hard edges, high contrast, raw honesty
- Before making ANY of these changes, invoke snooty-mcfigmaface:
  - Adding new component types (cards, modals, tooltips, etc.)
  - Introducing new colors or modifying the palette
  - Changing typography scales or weights
  - Creating new button variants or states
  - Designing empty states, error states, or loading states
  - Any deviation from established patterns
  - New page layouts or significant layout changes
- Established design patterns:
  - Borders: `border-4 border-black` (standard), `border-2 border-black` (subtle)
  - Shadows: `shadow-hard`, `shadow-hard-sm`
  - Colors (full palette):
    - `pm-yellow` (#FFF700): Selection, focus, tags, search, template actions
    - `pm-orange` (#FF5F1F): Remix, warning states, energy actions
    - `pm-pink` (#FF0044): Curate, share, navigation, branding
    - `pm-salmon` (#FF2F38): Critical emphasis, template underlines
    - `pm-green` (#00FF66): Create, add, success, resolved
    - `pm-cyan` (#00DFFF): Test, info, technical/experimental
    - `pm-red` (#DC2626): Delete, remove, error, destructive
    - `pm-muted` (#4B5563): Subdued text, labels, secondary info
    - `pm-black` (#0F141A): Borders, shadows, text
  - Typography: `font-black` for headers, `font-bold` for emphasis, `uppercase` for labels
  - Buttons: Solid backgrounds, black borders, shadow-hard, hover removes shadow + translates
- The snooty-mcfigmaface agent is consultative and will ask clarifying questions and present options

### Pre-PR Guardian Cycle (BLOCKING)
**BEFORE creating any PR**, run the guardian agent cycle to validate changes:

```bash
# Run ALL guardians before PR (required)
guardian-code       # Go format, vet, build, test compliance
guardian-security   # SQL injection, XSS, CSRF, OWASP vulnerabilities
guardian-performance # O(n) operations, unbounded loops, resource exhaustion
guardian-mcp        # MCP server sync with web functionality
guardian-docs       # Missing doc comments, README updates, CHANGELOG
```

**This is a BLOCKING requirement.** Do NOT create a PR until all guardians pass or issues are addressed. Guardian agents are in `.claude/agents/guardian-*.md`.