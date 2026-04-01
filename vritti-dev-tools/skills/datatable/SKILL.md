---
name: datatable
description: Creates a full-stack data table — backend table/import/export endpoints + frontend DataTable page with import/export and row selection. Use when adding a new table view for a domain entity.
---

You are implementing a table-backed endpoint in the Vritti backend. Follow this pattern exactly.

A table endpoint has four parts: Response DTO → Repository method → Service method → Controller endpoint.
Always implement in this order, bottom-up.

---

## 1. Response DTO

Extend `TableResponseDto<T>` from `@vritti/api-sdk`. Use `declare` to re-type inherited fields.
Place in `dto/response/<resource>-table-response.dto.ts`.

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { TableResponseDto, type TableViewState } from '@vritti/api-sdk';
import { ZoneDto } from '../entity/zone.dto';

export class ZoneTableResponseDto extends TableResponseDto<ZoneDto> {
  @ApiProperty({ type: [ZoneDto] })
  declare result: ZoneDto[];

  @ApiProperty()
  declare count: number;

  @ApiProperty()
  declare state: TableViewState;

  @ApiPropertyOptional({ nullable: true })
  declare activeViewId: string | null;
}
```

---

## 2. Repository — `findAllForTable` using `findAllAndCount`

Use the base class `this.findAllAndCount<T>()` which supports `select`, `leftJoins`, `groupBy`, `where`, `orderBy`, `limit`, `offset`. It runs count + data queries in parallel.

**Simple case — no joins:**
```typescript
async findAllForTable(options: {
  where?: SQL;
  orderBy?: SQL[];
  limit: number;
  offset: number;
}): Promise<{ result: Zone[]; count: number }> {
  return this.findAllAndCount<Zone>(options);
}
```

**With joins and aggregations — use `leftJoins` + `groupBy` + `array_agg`:**

```typescript
import { countDistinct, eq, sql } from '@vritti/api-sdk/drizzle-orm';

export type FeatureTableRow = Feature & {
  permissions: string[];
  platforms: string[];
  appFeatureCount: number;
};

async findAllForTable(options: {
  where?: SQL;
  orderBy?: SQL[];
  limit: number;
  offset: number;
}): Promise<{ result: FeatureTableRow[]; count: number }> {
  return this.findAllAndCount<FeatureTableRow>({
    select: {
      id: features.id,
      versionId: features.versionId,
      code: features.code,
      name: features.name,
      description: features.description,
      icon: features.icon,
      isActive: features.isActive,
      sortOrder: features.sortOrder,
      createdAt: features.createdAt,
      updatedAt: features.updatedAt,
      permissions: sql<string[]>`array_remove(array_agg(distinct ${featurePermissions.type}), null)`.mapWith({
        mapFromDriverValue: (value: string) => (value === '{}' || !value ? [] : value.slice(1, -1).split(',')),
      }),
      platforms: sql<string[]>`array_remove(array_agg(distinct ${microfrontends.platform}), null)`.mapWith({
        mapFromDriverValue: (value: string) => (value === '{}' || !value ? [] : value.slice(1, -1).split(',')),
      }),
      appFeatureCount: countDistinct(appFeatures.id),
    },
    leftJoins: [
      { table: featurePermissions, on: eq(featurePermissions.featureId, features.id) },
      { table: featureMicrofrontends, on: eq(featureMicrofrontends.featureId, features.id) },
      { table: microfrontends, on: eq(microfrontends.id, featureMicrofrontends.microfrontendId) },
      { table: appFeatures, on: eq(appFeatures.featureId, features.id) },
    ],
    groupBy: [features.id],
    ...options,
  });
}
```

### `findAllAndCount` options reference

| Option | Type | Description |
|--------|------|-------------|
| `select` | `Record<string, Column \| SQL>` | Custom select fields (omit for `select *`) |
| `where` | `SQL` | WHERE clause |
| `orderBy` | `SQL[]` | ORDER BY clauses |
| `limit` | `number` | LIMIT |
| `offset` | `number` | OFFSET |
| `leftJoin` | `{ table, on }` | Single LEFT JOIN (legacy) |
| `leftJoins` | `{ table, on }[]` | Multiple LEFT JOINs |
| `groupBy` | `(Column \| SQL)[]` | GROUP BY clause |

### PostgreSQL array aggregation with `.mapWith()`

`node-postgres` returns PostgreSQL arrays as string literals (`"{A,B,C}"`), not JS arrays. Use `.mapWith()` to parse:

```typescript
sql<string[]>`array_remove(array_agg(distinct ${col}), null)`.mapWith({
  mapFromDriverValue: (value: string) => (value === '{}' || !value ? [] : value.slice(1, -1).split(',')),
})
```

- `array_agg(distinct col)` — aggregates unique values into a PG array
- `array_remove(..., null)` — removes nulls from LEFT JOIN misses
- `.mapWith()` — parses `"{A,B}"` → `['A', 'B']` at driver level

---

## 3. Service — `FIELD_MAP` + `findForTable`

`FIELD_MAP` is a static readonly whitelist. Only include fields the frontend can filter/search/sort.

```typescript
@Injectable()
export class FeatureService {
  private readonly logger = new Logger(FeatureService.name);

  private static readonly FIELD_MAP: FieldMap = {
    name: { column: features.name, type: 'string' },
    code: { column: features.code, type: 'string' },
  };

  constructor(
    private readonly featureRepository: FeatureRepository,
    private readonly dataTableStateService: DataTableStateService,
  ) {}

  async findForTable(userId: string): Promise<FeatureTableResponseDto> {
    const { state, activeViewId } = await this.dataTableStateService.getCurrentState(userId, 'features');
    const where = and(
      FilterProcessor.buildWhere(state.filters, FeatureService.FIELD_MAP),
      FilterProcessor.buildSearch(state.search, FeatureService.FIELD_MAP),
    );
    const orderBy = FilterProcessor.buildOrderBy(state.sort, FeatureService.FIELD_MAP);
    const { limit = 20, offset = 0 } = state.pagination ?? {};
    const { result, count } = await this.featureRepository.findAllForTable({ where, orderBy, limit, offset });
    return {
      result: result.map((r) => FeatureDto.from(r, r.appFeatureCount, r.permissions, r.platforms)),
      count,
      state,
      activeViewId,
    };
  }
}
```

**FIELD_MAP rules:**
| Scenario | Pattern |
|----------|---------|
| Own column (string/number) | `{ column: zones.name, type: 'string' }` |
| Own column (boolean) | `{ column: zones.isActive, type: 'boolean' }` |
| FK column | `{ column: zones.regionId, type: 'string' }` |
| Many-to-many join | `{ expression: (v) => sql\`...\`, type: 'string' }` |

---

## 4. Controller endpoint

Thin — log, extract `@UserId()`, return service result directly.

```typescript
@Get('table')
@ApiFindForTableFeatures()
findForTable(@UserId() userId: string): Promise<FeatureTableResponseDto> {
  this.logger.log('GET /admin-api/features/table');
  return this.featureService.findForTable(userId);
}
```

---

## 5. Frontend — Schema + Service + Hook

### Schema type alias

```typescript
import type { TableResponse } from '@vritti/quantum-ui/api-response';

export interface Feature {
  id: string;
  code: string;
  name: string;
  icon: string;
  description: string | null;
  permissions: string[];
  platforms: string[];
  appCount: number;
  canDelete: boolean;
}

export type FeaturesTableResponse = TableResponse<Feature>;
```

### Service function

```typescript
export function getFeatures(versionId: string): Promise<FeaturesTableResponse> {
  return axios.get<FeaturesTableResponse>(`admin-api/versions/${versionId}/features/table`).then((r) => r.data);
}
```

### Query hook

```typescript
export const FEATURES_QUERY_KEY = (versionId: string) => ['admin', 'features', versionId, 'table'] as const;

export function useFeatures(versionId: string) {
  return useQuery<FeaturesTableResponse>({
    queryKey: FEATURES_QUERY_KEY(versionId),
    queryFn: () => getFeatures(versionId),
  });
}
```

---

## 6. Frontend — Table Page

### Page with import/export and row selection

```typescript
import { useFeatures } from '@hooks/admin/features';
import { FEATURES_QUERY_KEY } from '@hooks/admin/features/useFeatures';
import { useQueryClient } from '@tanstack/react-query';
import { Badge } from '@vritti/quantum-ui/Badge';
import { Button } from '@vritti/quantum-ui/Button';
import { type ColumnDef, DataTable, RowActions, getSelectionColumn, useDataTable } from '@vritti/quantum-ui/DataTable';
import { Dialog } from '@vritti/quantum-ui/Dialog';
import { useDialog } from '@vritti/quantum-ui/hooks';
import { PageHeader } from '@vritti/quantum-ui/PageHeader';

const TABLE_SLUG = 'features';

export const FeaturesPage = () => {
  const queryClient = useQueryClient();
  const { versionId } = useVersionContext();
  const { data: response, isLoading } = useFeatures(versionId);
  const addDialog = useDialog();

  const { table } = useDataTable({
    columns: getColumns({ onView }),
    slug: TABLE_SLUG,
    label: 'feature',
    serverState: response,
    enableRowSelection: true,   // enables checkbox column
    enableSorting: true,
    enableMultiSort: false,
    onStatePush: () => queryClient.invalidateQueries({ queryKey: FEATURES_QUERY_KEY(versionId) }),
  });

  return (
    <DataTable
      table={table}
      isLoading={isLoading}
      searchConfig={{ columns: [{ id: 'code', label: 'Code' }, { id: 'name', label: 'Name' }], searchAll: true }}
      importExport={{
        columns: [
          { key: 'code', label: 'Code' },
          { key: 'name', label: 'Name' },
          { key: 'icon', label: 'Icon' },
          { key: 'description', label: 'Description' },
          { key: 'permissions', label: 'Permissions' },
        ],
        sampleData: [
          { code: 'products', name: 'Products', icon: 'package', description: 'Product catalog', permissions: 'VIEW,CREATE,EDIT,DELETE' },
        ],
        importEndpoint: `admin-api/versions/${versionId}/features/import`,
        exportEndpoint: `admin-api/versions/${versionId}/features/export`,
        transformExportRow: (row) => ({
          code: row.code, name: row.name, icon: row.icon,
          description: row.description ?? '',
          permissions: row.permissions.join(','),
        }),
        filename: 'features',
        onSuccess: () => queryClient.invalidateQueries({ queryKey: FEATURES_QUERY_KEY(versionId) }),
      }}
      toolbarActions={{ actions: <Button size="sm" onClick={addDialog.open}><Plus /> Add Feature</Button> }}
    />
  );
};
```

### `importExport` prop reference

| Prop | Type | Description |
|------|------|-------------|
| `columns` | `{ key, label }[]` | Column definitions — same format for import and export |
| `sampleData` | `Record<string, string>[]` | Sample rows for download template |
| `importEndpoint` | `string` | `POST` endpoint (multipart file upload, all-or-nothing) |
| `exportEndpoint` | `string` | `GET` endpoint base path — format appended as path param (`/export/xlsx`) |
| `transformExportRow` | `(row: T) => Record<string, unknown>` | Transforms table row data for Export Selected (e.g., flatten arrays to comma-separated strings) |
| `filename` | `string` | Filename prefix for exported files |
| `onSuccess` | `() => void` | Called after successful import (typically invalidate queries) |

**What DataTable handles internally:**
- Toolbar dropdown with Import / Export All (submenu: CSV, Excel, Excel 97-2004, ODS, TSV)
- Import dialog (upload → all-or-nothing → success summary or error table)
- Export Selected via selection bar (CSV / Excel dropdown)
- Sample file download in multiple formats

### Row selection — `getSelectionColumn()`

When `enableRowSelection: true`, prepend `getSelectionColumn<T>()` to the columns array:

```typescript
function getColumns({ onView }): ColumnDef<Feature, unknown>[] {
  return [
    getSelectionColumn<Feature>(),
    { accessorKey: 'name', header: 'Name' },
    // ...
  ];
}
```

### Action column — always use `RowActions`

See `table-action-conventions.md` for full reference.

---

## 7. Import/Export Endpoints

### `POST /import` — All-or-nothing import

Accepts multipart file upload. Validates all rows; if any invalid, returns errors and imports nothing. If all valid, upserts (creates new, updates existing by code).

```typescript
// Controller
@Post('import')
@HttpCode(HttpStatus.OK)
@ApiConsumes('multipart/form-data')
@ApiImportFeatures()
async importFeatures(
  @Param('versionId') versionId: string,
  @UploadedFile() file: UploadedFileResult,
): Promise<ImportResponseDto> {
  return this.featureService.importFromFile(file.buffer, versionId);
}
```

**Response type: `ImportResponseDto` from `@vritti/api-sdk`**
```typescript
// Success
{ success: true, message: 'Import complete.', created: 3, updated: 2, skipped: 1 }

// Validation errors (nothing imported)
{ success: false, message: 'Validation failed.', rows: [{ index: 1, data: {...}, valid: false, errors: ['...'] }], summary: { total: 5, valid: 3, invalid: 2 } }
```

### `GET /export/:format` — Streamed file download

Format is a path param: `xlsx`, `csv`, `xls`, `ods`, `tsv`. Streams file via `FastifyReply`.

```typescript
// Controller
@Get('export/:format')
@ApiExportFeatures()
async exportFeatures(
  @Param('versionId') versionId: string,
  @Param('format') format: ExportFormat,
  @Res() reply: FastifyReply,
): Promise<void> {
  const buffer = await this.featureService.exportToBuffer(versionId, format);
  reply.header('Content-Type', getExportMimeType(format));
  reply.header('Content-Disposition', `attachment; filename="features.${getExportExt(format)}"`);
  reply.header('Content-Length', buffer.length);
  return reply.send(buffer);
}
```

**Shared utility — `@vritti/api-sdk/xlsx`:**
```typescript
import { type ExportFormat, buildExportBuffer, getExportExt, getExportMimeType } from '@vritti/api-sdk/xlsx';
```

Converts `Record<string, unknown>[]` to a spreadsheet `Buffer` in the given format. Isolated sub-path so xlsx isn't bundled with the main `@vritti/api-sdk` import.

---

## Canonical references

**Backend — Features module** (joined table + import/export):
- `cloud-server/src/modules/domain/version/feature/root/repositories/feature.repository.ts` — `findAllForTable` with `leftJoins` + `groupBy` + `array_agg`
- `cloud-server/src/modules/domain/version/feature/root/services/feature.service.ts` — `findForTable`, `importFromFile`, `exportToBuffer`
- `cloud-server/src/modules/admin-api/version/feature/root/controllers/feature.controller.ts` — table + import + export endpoints

**Backend — Region module** (simple table, no joins):
- `cloud-server/src/modules/admin-api/region/controllers/region.controller.ts`
- `cloud-server/src/modules/domain/region/services/region.service.ts`
- `cloud-server/src/modules/domain/region/repositories/region.repository.ts`

**Frontend — Features page** (import/export + row selection):
- `cloud-web/src/pages/admin/versions/features/FeaturesPage.tsx`

**Frontend — Industries page** (simple table):
- `cloud-web/src/pages/admin/industries/IndustriesPage.tsx`

**Shared SDK types:**
- `api-sdk/src/database/repositories/primary-base.repository.ts` — `findAllAndCount` with `leftJoins` + `groupBy`
- `api-sdk/src/database/dto/table-response.dto.ts` — `TableResponseDto<T>`
- `api-sdk/src/database/filter/filter.processor.ts` — `FilterProcessor`, `FieldMap`
- `@vritti/api-sdk/xlsx` — `buildExportBuffer`, `ExportFormat`, `getExportMimeType`, `getExportExt`
- `cloud-server/src/utils/validate-import-rows.ts` — `validateImportRows`, `ImportResult`

---

Now implement the table endpoint for the resource the user described. Read the existing schema and entity DTO first, then build bottom-up: backend (DTO → repository → service → controller), then frontend (schema → service → hook → page).
