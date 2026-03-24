---
name: new-selector
description: Creates a full-stack selector — backend select-api endpoint + quantum-ui Selector and Filter components. Use when adding a new dropdown/select for a domain entity.
---

# New Selector — Full Stack

You are creating a new selector for the entity described in "$ARGUMENTS". This involves both backend (select-api endpoint) and frontend (quantum-ui Selector + Filter components). Follow every step exactly.

---

## Step 0: Gather Information

Before writing any code, determine:

1. **Entity name** (PascalCase): e.g., `Zone`, `Warehouse`, `Supplier`
2. **Plural name** (kebab-case): e.g., `zones`, `warehouses`, `suppliers`
3. **Which server** has the domain module: `cloud-server` or `core-server`
4. **DB table name** and which columns to expose: typically `id` (value) + `name` (label), optionally `code` (description)
5. **Filter field name** (camelCase): typically `{entity}Id` (e.g., `zoneId`, `warehouseId`)

If any of these are unclear from the user's request, **ask before proceeding**.

---

## Step 1: Backend — Domain Service Method

Add `findForSelect` to the entity's existing domain service.

**File**: `modules/domain/{entity}/services/{entity}.service.ts`

```typescript
// Returns paginated {entity} options for the select component
findForSelect(query: SelectOptionsQueryDto): Promise<SelectQueryResult> {
  this.logger.log(`Fetched {entity} select options (limit: ${query.limit}, offset: ${query.offset})`);
  return this.{entity}Repository.findForSelect({
    value: query.valueKey || 'id',
    label: query.labelKey || 'name',
    description: query.descriptionKey,
    groupId: query.groupIdKey,
    search: query.search,
    limit: query.limit,
    offset: query.offset,
    values: query.values,
    excludeIds: query.excludeIds,
    orderBy: { name: 'asc' },
  });
}
```

Import `SelectOptionsQueryDto` and `SelectQueryResult` from `@vritti/api-sdk`. The `findForSelect` method is inherited from `PrimaryBaseRepository` — no repository changes needed.

---

## Step 2: Backend — Select-API Controller

Create a new controller in the select-api module.

**File**: `modules/select-api/controllers/{entity}-select.controller.ts`

```typescript
import { Controller, Get, Logger, Query } from '@nestjs/common';
import { ApiBearerAuth, ApiTags } from '@nestjs/swagger';
import { RequireSession, SelectOptionsQueryDto, type SelectQueryResult } from '@vritti/api-sdk';
import { SessionTypeValues } from '@/db/schema';
import { {Entity}Service } from '@domain/{entity}/services/{entity}.service';

@ApiTags('Select')
@ApiBearerAuth()
@RequireSession(SessionTypeValues.CLOUD, SessionTypeValues.ADMIN)
@Controller('{entities}')
export class {Entity}SelectController {
  private readonly logger = new Logger({Entity}SelectController.name);

  constructor(private readonly {entity}Service: {Entity}Service) {}

  // Returns paginated {entity} options for the select component
  @Get()
  findForSelect(@Query() query: SelectOptionsQueryDto): Promise<SelectQueryResult> {
    this.logger.log('GET /select-api/{entities}');
    return this.{entity}Service.findForSelect(query);
  }
}
```

**Register** the controller in `modules/select-api/select.module.ts`:
- Add the domain module to `imports` (if not already imported)
- Add the controller to `controllers`

---

## Step 3: Frontend — Selector Component

Create the selector folder in quantum-ui.

**File**: `quantum-ui/lib/selects/{entity}/index.ts`
```typescript
export { {Entity}Filter, type {Entity}FilterProps } from './{Entity}Filter';
export { {Entity}Selector, type {Entity}SelectorProps } from './{Entity}Selector';
```

**File**: `quantum-ui/lib/selects/{entity}/{Entity}Selector.tsx`
```typescript
import { forwardRef } from 'react';
import { Select, type SelectProps } from '../../components/Select/Select';

export type {Entity}SelectorProps = Omit<SelectProps, 'optionsEndpoint'>;

// Pre-configured Select for {entity} selection
export const {Entity}Selector = forwardRef<HTMLButtonElement, {Entity}SelectorProps>((props, ref) => (
  <Select
    ref={ref}
    label="{Display Label}"
    placeholder="Select {display label}"
    searchable
    optionsEndpoint="select-api/{entities}"
    fieldKeys={{ valueKey: 'id', labelKey: 'name' }}
    {...props}
  />
));
{Entity}Selector.displayName = '{Entity}Selector';
```

If the entity has a `code` field worth showing, add `descriptionKey: 'code'` to `fieldKeys`.

**File**: `quantum-ui/lib/selects/{entity}/{Entity}Filter.tsx`
```typescript
import { forwardRef } from 'react';
import { SelectFilter, type SelectFilterProps } from '../../components/Select/SelectFilter';

export type {Entity}FilterProps = Omit<SelectFilterProps, 'optionsEndpoint' | 'name'> & { name?: string };

// Pre-configured SelectFilter for {entity} filtering
export const {Entity}Filter = Object.assign(
  forwardRef<HTMLButtonElement, {Entity}FilterProps>((props, ref) => (
    <SelectFilter
      ref={ref}
      name="{entityId}"
      label="{Display Label}"
      placeholder="Select {display label}"
      optionsEndpoint="select-api/{entities}"
      fieldKeys={{ valueKey: 'id', labelKey: 'name' }}
      {...props}
    />
  )),
  { displayName: '{Entity}Filter', defaultLabel: '{Display Label}' },
);
```

---

## Step 4: Frontend — Register in quantum-ui

Three files must be updated:

### 4a. Barrel export

**File**: `quantum-ui/lib/selects/index.ts` — add:
```typescript
export * from './{entity}';
```

### 4b. Package.json exports

**File**: `quantum-ui/package.json` — add in the `"exports"` object (alphabetical):
```json
"./selects/{entity}": {
  "types": "./dist/lib/selects/{entity}/index.d.ts",
  "import": "./dist/selects/{entity}.js"
},
```

### 4c. Vite entry point

**File**: `quantum-ui/vite.config.ts` — add in the `entry` object:
```typescript
'selects/{entity}': resolve(__dirname, 'lib/selects/{entity}/index.ts'),
```

---

## Step 5: Build and Verify

1. **Build quantum-ui**: `cd quantum-ui && pnpm build`
   - Verify zero errors
   - Verify `dist/selects/{entity}.js` exists in output

2. **Type-check the server**: `cd {server}/apps/{app} && npx tsc --noEmit`
   - Verify the new controller compiles

3. **Test the import** works:
   ```typescript
   import { {Entity}Selector } from '@vritti/quantum-ui/selects/{entity}';
   import { {Entity}Filter } from '@vritti/quantum-ui/selects/{entity}';
   ```

---

## Conventions Checklist

- [ ] `//` comments only (no JSDoc)
- [ ] Every method has a one-liner `//` comment
- [ ] No destructuring in Selector/Filter wrappers — defaults first, `{...props}` last
- [ ] `export const` for components
- [ ] `forwardRef` with `displayName`
- [ ] Filter uses `Object.assign` for `displayName` + `defaultLabel`
- [ ] Controller is thin: log + one service call
- [ ] Exceptions from `@vritti/api-sdk`, not `@nestjs/common`

---

## Reference: Existing Selectors

| Entity | Endpoint | fieldKeys | Filter name |
|--------|----------|-----------|-------------|
| App | `select-api/apps` | id, name, code | `appId` |
| AppCode | `select-api/app-codes` | code, name, code | `appCode` |
| CloudProvider | `select-api/cloud-providers` | id, name | `cloudProviderId` |
| Deployment | `select-api/deployments` | id, name | `deploymentId` |
| Feature | `select-api/features` | id, name, code | `featureId` |
| Industry | `select-api/industries` | id, name | `industryId` |
| Plan | `select-api/plans` | id, name | `planId` |
| Region | `select-api/regions` | id, name | `regionId` |

Static selectors (no backend): Currency, Locale, Timezone — use `options` prop instead of `optionsEndpoint`.

---

Now implement the selector for the entity the user described. Read the existing domain service and DB schema first, then build in order: service method → controller → quantum-ui components → registration → build.
