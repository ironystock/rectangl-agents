---
description: Scaffold a new templ component following the design system
argument-hint: <ComponentName> [type: button|card|input|section|row]
allowed-tools: Read, Edit, Write, Grep, Glob
---

Create a new templ component following Promptmark's Neo-Brutalist design system.

Arguments: $ARGUMENTS
- Component name (PascalCase, e.g., `StatusBadge`)
- Optional type hint: button, card, input, section, row

## Design System Reference

BEFORE creating any component, review:
- @docs/DESIGN_SYSTEM.md - The law
- @cmd/web/components.templ - Existing reusable components

## Core Patterns

### Button
```go
templ ButtonPrimary(label string) {
    <button class="bg-pm-green text-black font-black py-3 px-8 border-4 border-black shadow-hard hover:shadow-none hover:translate-x-1 hover:translate-y-1 transition-all uppercase">
        { label }
    </button>
}
```

### Card
```go
templ Card(title string, protocol string, color string) {
    <div class="bg-surface border-4 border-border shadow-hard">
        <div class="bg-pm-black text-white px-4 py-3 border-b-4 border-border font-mono text-sm">
            <span class={ "font-bold", templ.KV("text-pm-green", color == "green"), templ.KV("text-pm-pink", color == "pink") }>
                { protocol }://
            </span>
            { title }
        </div>
        <div class="p-6">
            { children... }
        </div>
    </div>
}
```

### Input with Label
```go
templ InputField(id string, label string, value string, inputType string) {
    <div>
        <label for={ id } class="font-black uppercase text-xs text-text-muted mb-2 tracking-wide block">
            { label }
        </label>
        <input
            type={ inputType }
            id={ id }
            name={ id }
            value={ value }
            class="w-full border-4 border-border p-3 font-bold bg-surface focus:outline-none focus:shadow-hard-sm transition-shadow"
        />
    </div>
}
```

### Section
```go
templ Section(title string, headerColor string) {
    <div class="bg-surface border-4 border-border shadow-hard">
        <div class={ "px-6 py-4 border-b-4 border-border font-black uppercase tracking-wide", templ.KV("bg-pm-black text-white", headerColor == "black"), templ.KV("bg-pm-pink text-white", headerColor == "pink") }>
            { title }
        </div>
        <div class="p-6">
            { children... }
        </div>
    </div>
}
```

### List Row (with actions)
```go
templ ListRow(id string) {
    <div id={ id } class="flex items-center justify-between p-4 hover:bg-pm-pink/10 transition-colors border-b-4 border-border last:border-b-0">
        <div class="flex items-center gap-4">
            { children... }
        </div>
        <div class="flex items-center gap-2">
            // Action buttons here
        </div>
    </div>
}
```

## Forbidden

- NO `rounded-*` classes (Neo-Brutalist = sharp edges)
- NO `focus:ring-*` (use `focus:shadow-hard-sm`)
- NO colors outside `pm-*` palette
- NO `shadow-md/lg` (use `shadow-hard` only)
- NO light font weights for headers (`font-black` required)

## Steps

1. Determine if component should go in:
   - `cmd/web/components.templ` (if reusable across pages)
   - Domain-specific file (e.g., `cmd/web/prompts/card.templ`)
2. Check if similar component exists (reuse/extend if so)
3. Write component following patterns above
4. Run `/build` to verify it compiles

Now scaffold the component: $ARGUMENTS
