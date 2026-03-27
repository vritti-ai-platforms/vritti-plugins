---
name: skeleton
description: Adds a Suspense skeleton to a view page — converts the hook to useSuspenseQuery, creates a co-located skeleton component, and wraps the route with <Suspense>.
---

# View Page Skeleton

You are adding a loading skeleton to the view page described in "$ARGUMENTS". Follow every step exactly.

---

## Step 0: Gather Information

Before writing any code, determine:

1. **Page component file** — e.g., `pages/admin/regions/RegionViewPage.tsx`
2. **Data hook** — e.g., `hooks/admin/regions/useRegion.ts`
3. **Page layout** — read the page component and note:
   - Number and layout of stat cards (grid cols)
   - Whether it uses Tabs, DangerZone, or custom card sections
   - Any unique sections (provider toggles, content lists, etc.)

If any of these are unclear from the user's request, **ask before proceeding**.

---

## Step 1: Convert Hook to `useSuspenseQuery`

**File**: `hooks/admin/{entity}/use{Entity}.ts`

Replace `useQuery` with `useSuspenseQuery`. Remove the optional `options` parameter — suspense hooks don't need it.

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import type { AxiosError } from 'axios';
import type { {Entity} } from '@/schemas/admin/{entities}';
import { get{Entity} } from '../../../services/admin/{entities}.service';

// Fetches a single {entity} by ID — suspends until data is ready
export function use{Entity}(id: string) {
  return useSuspenseQuery<{Entity}, AxiosError>({
    queryKey: ['admin', '{entities}', id],
    queryFn: () => get{Entity}(id),
  });
}
```

---

## Step 2: Create Skeleton Component

**File**: `pages/admin/{entities}/{Page}Skeleton.tsx`

Compose from quantum-ui building blocks. Match the page's exact layout (grid cols, gap, padding).

### Available building blocks

| Component | Import | Props |
|-----------|--------|-------|
| `PageHeaderSkeleton` | `@vritti/quantum-ui/PageHeader` | `showDescription?`, `showActions?` |
| `CardSkeleton` | `@vritti/quantum-ui/Card` | `count?`, `children` (inner content) |
| `TabsSkeleton` | `@vritti/quantum-ui/Tabs` | `count?`, `contentHeight?`, `tabWidths?` |
| `DangerZoneSkeleton` | `@vritti/quantum-ui/DangerZone` | `showWarning?` |
| `Skeleton` | `@vritti/quantum-ui/Skeleton` | `className` for custom shapes |

### Rules

- **No shadows** — skeleton cards use `CardSkeleton`, not real `Card`
- **No colored borders** — no `border-destructive/50` on skeleton danger zones
- **Match padding** — stat card children need `p-6` to match `CardContent className="p-6"`
- **Match grid** — use the same `grid-cols-{n}` and `gap-{n}` as the real page
- **Custom sections** — use raw `Skeleton` blocks inside a card-like div (`bg-card rounded-xl border py-6`)

### Example (3 stat cards + providers card + danger zone)

```tsx
import { CardSkeleton } from '@vritti/quantum-ui/Card';
import { DangerZoneSkeleton } from '@vritti/quantum-ui/DangerZone';
import { PageHeaderSkeleton } from '@vritti/quantum-ui/PageHeader';
import { Skeleton } from '@vritti/quantum-ui/Skeleton';

export const {Page}Skeleton = () => (
  <div className="flex flex-col gap-6">
    <PageHeaderSkeleton />

    <div className="grid grid-cols-3 gap-4">
      <CardSkeleton count={3}>
        <div className="flex items-center gap-4 p-6">
          <Skeleton className="w-12 h-12 rounded-lg shrink-0" />
          <div className="flex flex-col gap-1.5">
            <Skeleton className="h-3.5 w-24" />
            <Skeleton className="h-7 w-8" />
          </div>
        </div>
      </CardSkeleton>
    </div>

    {/* Custom card section */}
    <div className="bg-card text-card-foreground rounded-xl border py-6">
      <div className="flex flex-col gap-6">
        <div className="px-6 flex flex-col gap-1.5">
          <Skeleton className="h-5 w-32" />
          <Skeleton className="h-4 w-72" />
        </div>
        <div className="px-6">
          <Skeleton className="h-48 w-full rounded-lg" />
        </div>
      </div>
    </div>

    <DangerZoneSkeleton />
  </div>
);
```

---

## Step 3: Update View Page

**File**: `pages/admin/{entities}/{Page}.tsx`

1. Remove `Spinner` import
2. Remove `isLoading` destructuring from hook — use `const { data: {entity} } = use{Entity}(id)`
3. Remove `if ({entity}Loading) return <Spinner />` branch
4. Remove `if (!{entity}) return null` guard
5. Remove optional chaining on data (`{entity}?.name` → `{entity}.name`)

---

## Step 4: Update Route

**File**: `routes.tsx`

1. Import `Suspense` from `'react'` (if not already imported)
2. Import `{Page}Skeleton` from the skeleton file
3. Wrap the page element:

```typescript
{
  path: '{entities}/:slug',
  element: (
    <Suspense fallback={<{Page}Skeleton />}>
      <{Page} />
    </Suspense>
  ),
}
```

---

## Step 5: Verify

1. **Type-check**: `pnpm exec tsc --noEmit -p apps/cloud-web/tsconfig.json`
2. **Navigate** to the view page — skeleton should display during load
3. **Error state** — handled by existing `QueryErrorBoundary` in layout

---

## Conventions Checklist

- [ ] Hook uses `useSuspenseQuery`, not `useQuery`
- [ ] Skeleton file co-located with page component
- [ ] Uses quantum-ui skeleton building blocks
- [ ] No shadows or elevation on skeleton cards
- [ ] No colored borders on skeleton danger zones
- [ ] Grid cols and gap match the real page
- [ ] `<Suspense>` wraps the page at route level
- [ ] `export const` for skeleton component
