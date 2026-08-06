---
name: frontend-dev
description: Next.js App Router frontend development for this workspace
triggers:
  - working on .tsx/.ts/.css files in frontend/src/
  - Next.js App Router page/layout development
  - UI/UX improvements or fixes
  - dashboard/sidebar layout issues
---

# Frontend Development Skill

## Overview
Frontend development practices for this workspace: Next.js 16+ App Router, React, TypeScript, Tailwind CSS, shadcn/ui components.

## Key Conventions

### Path & Workspace
- App path: `/home/cfollette18/System-Design ` (note trailing space — always quote in commands)
- Workspace packages: `@sd/frontend`, `@sd/database`
- Lucide React icons at root `node_modules/lucide-react`
- shadcn/ui components at `frontend/src/components/ui/`

### "use client" Directive
Must NOT have semicolon:
```tsx
'use client'  // ✓ correct
'use client'; // ✗ wrong
```

### Dashboard Layout Pattern
For sidebar + scrollable main content:
```tsx
// layout.tsx
<div className="flex h-screen overflow-hidden">
  <Sidebar />
  <main className="flex-1 min-h-0 overflow-y-auto">
    {children}
  </main>
</div>
```
CRITICAL: The main content area needs `min-h-0` (not `min-h-screen`) to allow internal scrolling. `h-screen overflow-hidden` on wrapper prevents double scrollbars.

### Template Literals with $
When using Python `execute_code` to write files with template literals containing `$`:
```python
content = r'''template content with ${variable}'''
```
Use raw strings to avoid double-escaping.

## UI Quality Bar
User expects professional, Linear/Shopify/Stripe-inspired design:
- Dark theme with `#09090b` background, `#27272a` borders
- Blue accent `#3b82f6`
- Clean typography, proper spacing
- Subtle hover states and transitions
- Custom scrollbars matching theme

## Build Error Patterns

### Turbopack Cache Corruption
When build fails with cryptic errors (especially after file modifications):
```bash
cd "/home/cfollette18/System-Design " && rm -rf .next && npm run dev
```
Do NOT just restart — clean the `.next` directory.

### Icon Import Errors
`Database` icon does not exist in lucide-react. Use `Server` for data/entity-related icons.

## Important Files
- `frontend/src/app/(dashboard)/layout.tsx` — main dashboard layout with sidebar
- `frontend/src/app/globals.css` — design system CSS variables and theme
- `frontend/src/app/(dashboard)/dashboard/page.tsx` — dashboard home
- `frontend/src/app/(dashboard)/interview/new/page.tsx` — new interview form
- `frontend/src/app/(dashboard)/interview/[sessionId]/page.tsx` — interview room
- `frontend/src/app/(dashboard)/history/page.tsx` — session history
- `frontend/src/app/(dashboard)/settings/page.tsx` — settings

## Development Commands
```bash
cd "/home/cfollette18/System-Design " && npm run dev  # Start dev servers
```

## Verification
After any layout change, verify all routes return 200:
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/dashboard
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/interview/new
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/history
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/settings
```
