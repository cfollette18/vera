# Next.js + Tailwind Layout Patterns

## The `h-screen` + `overflow-hidden` Trap

When building dashboard layouts with a sidebar, the naive pattern fails:

```tsx
// WRONG - content gets cut off, no scroll
<div className="flex h-screen">
  <Sidebar />
  <main className="flex-1 overflow-hidden">
    {children}  {/* can't scroll if children is tall */}
  </main>
</div>
```

The `h-screen` on the wrapper + `overflow-hidden` on main creates a **rigid stacking context**. If the main content area has internal padding that makes content taller than the viewport, the overflow is swallowed and no scrollbar appears.

### Why it happens

`h-screen` = `100vh` (fixed viewport height). `overflow-hidden` clips anything beyond that boundary. The sidebar takes its natural height, the remaining space is exactly `100vh - sidebar-height`, and if children exceed that, they're simply invisible.

### The fix

```tsx
// CORRECT - allows scrolling in main content
<div className="flex h-screen overflow-hidden">
  <Sidebar />
  <main className="flex-1 min-h-0 overflow-y-auto">
    {children}
  </main>
</div>
```

Key differences:
- `overflow-hidden` stays on the wrapper (prevents body scroll)
- `min-h-0` on main: **this is critical** — flex children default to `min-height: auto` which prevents them from shrinking below content size. Setting `min-h-0` allows the flex child to shrink and accept `overflow-y-auto`
- `overflow-y-auto` on main: lets the main area scroll independently

### Alternative pattern (scrollable wrapper)

```tsx
<div className="flex h-screen">
  <Sidebar />
  <div className="flex-1 flex flex-col min-h-0 overflow-hidden">
    <Header /> {/* fixed height header */}
    <main className="flex-1 overflow-y-auto">
      {children}
    </main>
  </div>
</div>
```

## Tailwind Dark Theme Tokens (Linear-style)

Minimum viable dark theme for professional UIs:

```css
:root {
  --background: #09090b;  /* near-black, NOT #0f172a or #1e293b */
  --foreground: #fafafa;  /* off-white, NOT #ffffff */
  --card: #18181b;        /* elevated surface */
  --border: #27272a;      /* subtle borders, NOT #334155 */
  --muted-foreground: #71717a;  /* muted text */
  --primary: #3b82f6;     /* blue accent */
  --accent: #3b82f6;      /* interactive elements */
}
```

Generic "slate-900" palettes with "slate-700" borders look cheap — this is the Linear/Shopify bar.
