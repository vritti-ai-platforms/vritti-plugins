---
name: permission-code
description: Adds a permission code end to end — cloud catalog (via the Vritti Admin MCP), the commerce-permissions lib, and the @RequirePermission on the route that enforces it. Use when adding, renaming or removing any permission.
---

# Permission Code — Full Stack

You are adding the permission(s) described in "$ARGUMENTS". A permission lives in **three places that must
agree exactly** — a mismatch fails closed, so the button silently never appears.

| Layer | Where | Authored with |
|---|---|---|
| 1. Lib constant | `vritti-core/libs/commerce-permissions/src/<feature>.ts` | edit + rebuild |
| 2. Cloud catalog | Vritti Cloud admin database | **Vritti Admin MCP** |
| 3. Enforcement | gateway controller `@RequirePermission(...)` + frontend `permission` prop | edit |

---

## Step 0: Know what is being added

Establish, before writing anything:

- **feature code** (`offerings`, `inventory-items`, …) and **which scopes** it exists in. Commerce features exist
  as ORG, LE and SITE rows under one code. A permission normally goes into every scope the feature has.
- **the action code**: bare and dotted-lowercase. `export`, `toggle`, `variants.export`. A code inside a group is
  stored as `<group>.<action>` — the group supplies the prefix, so send only the action plus `group`.
- **its prerequisite**: `add`/`edit`/`delete`/`toggle`/`export` → `view`; `<sub>.view` → `view`;
  `<sub>.{add,edit,delete}` → `<sub>.view`.
- **which plan sells it** and **which role templates grant it**. Ask if it is not obvious — this is a product
  decision, not a mechanical one.

## Step 1: Cloud catalog, in one MCP call

Read the server's `add_permission_code` prompt and the `vritti://catalog/conventions` resource first.

```
list_versions                                  → versionId
list_features (query: "<feature>")             → confirm the scopes
list_permission_groups (featureId)             → the group must already exist
list_permissions (featureId)                   → current highest sortOrder
```

Then one call, dry run first:

```
author_permissions({
  versionId,
  feature: { code: '<feature>', scopes: ['ORG', 'LE', 'SITE'] },
  permissions: [
    { code: 'export', label: 'Export', dependsOn: ['view'], platforms: ['WEB'], sortOrder: <max + 1> },
    { code: 'export', group: 'variants', label: 'Export', dependsOn: ['variants.view'], platforms: ['WEB'], sortOrder: <max + 2> },
  ],
  unlockInPlans: [{ business: { code: 'fmcg' }, plan: { code: 'essential' } }],
  grantToRoleTemplates: [{ business: { code: 'fmcg' }, roleTemplate: { code: 'org-admin' } }],
  snapshot: true,
  dryRun: true,
})
```

Read the dry run back, then re-run without `dryRun`. **All five layers matter**: the row, its `platforms`, the
plan unlock, the role grant, the snapshot. Stopping early leaves a permission that exists and does nothing —
that is the most common failure, and it is invisible until someone reports a missing button.

The response tells you what moved. `changed: false` means the catalog already looked like this. Per scope you get
`created` vs `alreadyPresent`, plus `differs` when a stored row disagrees with what you asked for — a corrected
label is **not** applied to an existing code, it is reported, so re-run with the same input is always safe.

## Step 1b: Put the new codes in their place

A new permission lands after the current maximum `sortOrder`, which means it sits at the very end of the list —
below the group blocks, away from the actions it belongs with. One call fixes every scope:

```
reorder_permissions({
  versionId,
  feature: { code: '<feature>', scopes: ['ORG', 'LE', 'SITE'] },
  order: ['view','add','edit','delete','toggle','set-tax-class','export',
          'dimensions.view','dimensions.add','dimensions.edit','dimensions.delete',
          'variants.view','variants.add','variants.edit','variants.delete','variants.export',
          'variants.bom.view','variants.bom.add','variants.bom.edit','variants.bom.delete'],
  dryRun: true,
})
```

`sortOrder` is per **feature**, not per group, so the list settles the whole feature at once. House order: the
feature's own actions first (view, add, edit, delete, then toggles and specials, then export), then each group in
turn with its `view` first and any nested sub-group last.

Two rules the tool enforces so you cannot half-apply an order:
- The list must name **every** code the feature carries. Anything omitted keeps its old number and collides — the
  tool refuses and names what is missing, per scope.
- A code a narrower scope does not hold (ORG-only actions such as `variants.bom.create-inventory-item`) comes back
  as `notInScope` and is skipped, not treated as an error.

Labels: bare verb (`View`, `Add`, `Edit`, `Delete`, `Export`) — the group already names the noun. Spell one out
only for a non-standard action (`Add from Template`, `Activate`).

## Step 2: The lib constant

In `vritti-core/libs/commerce-permissions/src/<feature>.ts`, add the key to **every** scope object
(`ORG_X`, `LE_X`, `SITE_X`), nesting it exactly as the dotted code nests:

```ts
export const ORG_OFFERINGS = {
  // …
  export: 'org.offerings.export',
  variants: {
    // …
    export: 'org.offerings.variants.export',
  },
} as const;
```

Then rebuild — consumers import the built `dist/`:

```bash
pnpm --filter @vritti/commerce-permissions build
```

## Step 3: Enforcement

**Gateway** — on the route, in all three scope controllers:

```ts
@Get('export/:format')
@RequirePermission(ORG_OFFERINGS.export)
```

`GET`→view, `POST`→add, `PATCH`→edit, `DELETE`→delete; a sub-resource path takes the nested code.

**Frontend** — add the key to the app's `…Permissions` interface, then pass it to the gating prop of the
component that performs the action (`Button` / `Switch` / `DataTable` / `RowActions` item / `Tabs` item /
`DangerZone` / `DataTable` `importExport.exportPermission`). Never hand-roll `usePermission(...).granted &&`.

## Step 4: Verify

```bash
npx tsc --noEmit -p apps/core-server/tsconfig.json
npx tsc --noEmit -p apps/web-mfs/commerce-mf/tsconfig.json
```

Then read the catalog back and confirm the new rows carry the platform you asked for, and sit where you put them:

```
list_permissions (featureId)                    → platforms include WEB, sortOrder has no duplicates
get_plan_unlocks (planId, feature: '<feature>') → the code must be listed
```

`get_plan_unlocks` defaults to the compact view — permissions by code, narrowable with `feature` / `scope` / `app`.
Only ask for `view: 'matrix'` to render a full checkbox grid; it returns the version's whole catalog.

## Known traps

- **A permission with no platform resolves for nobody.** `author_permissions` defaults to `['WEB']`; the older
  `bulk_create_permissions` path and the REST create leave every flag false unless you pass them.
- **A feature code alone is ambiguous** — the same code exists in ORG, LE and SITE, and is now rejected with the
  scopes it matches. Name the scope or pass the id.
- **`set_plan_unlocks` replaces the entire matrix.** Use `set_plan_permissions` (codes, `mode: 'add'`) or
  `author_permissions`, which merge.
- **A reorder makes the snapshot stale.** Display order lives in the catalog, so `snapshot_version` has to run
  before deployments see it.
- **The snapshot is what deployments read.** Until `snapshot_version` runs, nothing you authored is visible.
- **Deps-only PATCH on a permission 500s** — send `label` and `sortOrder` alongside.
