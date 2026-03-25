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

## 2. Repository — `findAllWithCounts`

Options object, parallel count + data queries. Defaults (`limit = 20`, `offset = 0`) in destructuring.

**Simple case — no joins:**
```typescript
type ZoneRow = Zone;

async findAllWithCounts(options: {
  where?: SQL;
  orderBy?: SQL[];
  limit?: number;
  offset?: number;
}): Promise<{ rows: ZoneRow[]; total: number }> {
  const { where, orderBy, limit = 20, offset = 0 } = options;
  const [countResult, rows] = await Promise.all([
    this.db.select({ total: count() }).from(zones).where(where).then((r) => r[0]?.total ?? 0),
    this.db
      .select(getTableColumns(zones))
      .from(zones)
      .where(where)
      .orderBy(...(orderBy?.length ? orderBy : [asc(zones.name)]))
      .limit(limit)
      .offset(offset),
  ]);
  return { rows, total: Number(countResult) };
}
```

**With a FK join (one related object):**
```typescript
type ZoneWithRegion = Zone & { region: { id: string; name: string } | null };

// In select:
region: sql<{ id: string; name: string } | null>`
  CASE WHEN ${regions.id} IS NOT NULL
    THEN json_build_object('id', ${regions.id}, 'name', ${regions.name})
    ELSE NULL
  END
`,
// After .from(zones):
.leftJoin(regions, eq(regions.id, zones.regionId))
// groupBy required when mixing aggregate-style SQL:
.groupBy(zones.id, regions.id, regions.name)
```

**With a many-to-many join (array of related objects):**
```typescript
type ZoneWithProviders = Zone & {
  providerCount: number;
  providers: Array<{ id: string; name: string }>;
};

// In select:
providerCount: count(zoneProviders.providerId),
providers: sql<Array<{ id: string; name: string }>>`
  json_agg(
    json_build_object('id', ${providers.id}, 'name', ${providers.name})
  ) FILTER (WHERE ${providers.id} IS NOT NULL)
`,
// After .from(zones):
.leftJoin(zoneProviders, eq(zoneProviders.zoneId, zones.id))
.leftJoin(providers, eq(providers.id, zoneProviders.providerId))
.groupBy(zones.id)
```

> `FILTER (WHERE ... IS NOT NULL)` prevents null sentinel values in the aggregated array.

---

## 3. Service — `FIELD_MAP` + `findForTable`

`FIELD_MAP` is a static readonly whitelist. Only include fields the frontend can filter/search/sort.

```typescript
import { Injectable } from '@nestjs/common';
import { and, sql } from '@vritti/api-sdk/drizzle-orm';
import { FilterProcessor, NotFoundException, SuccessResponseDto, type FieldMap } from '@vritti/api-sdk';
import { zones, zoneProviders } from '@/db/schema';
import { TableViewService } from '../../table-view/services/table-view.service';
import { ZoneDto } from '../dto/entity/zone.dto';
import { ZoneTableResponseDto } from '../dto/response/zone-table-response.dto';
import { ZoneRepository } from '../repositories/zone.repository';

@Injectable()
export class ZoneService {
  private readonly logger = new Logger(ZoneService.name);

  private static readonly FIELD_MAP: FieldMap = {
    name:     { column: zones.name,     type: 'string'  },
    code:     { column: zones.code,     type: 'string'  },
    isActive: { column: zones.isActive, type: 'boolean' },
    regionId: { column: zones.regionId, type: 'string'  },
    // Many-to-many join filter — use expression, not column
    providerId: {
      expression: (value) =>
        sql`${zones.id} IN (SELECT zone_id FROM zone_providers WHERE provider_id = ${String(value)})`,
      type: 'string',
    },
  };

  constructor(
    private readonly zoneRepository: ZoneRepository,
    private readonly tableViewService: TableViewService,
  ) {}

  // Returns paginated zones with current filter/sort/search/pagination state applied
  async findForTable(userId: string): Promise<ZoneTableResponseDto> {
    const { state, activeViewId } = await this.tableViewService.getCurrentState(userId, 'zones');
    const where = and(
      FilterProcessor.buildWhere(state.filters, ZoneService.FIELD_MAP),
      FilterProcessor.buildSearch(state.search, ZoneService.FIELD_MAP),
    );
    const { limit = 20, offset = 0 } = state.pagination ?? {};
    const { rows, total } = await this.zoneRepository.findAllWithCounts({
      where,
      orderBy: FilterProcessor.buildOrderBy(state.sort, ZoneService.FIELD_MAP),
      limit,
      offset,
    });
    return {
      result: rows.map((z) => ZoneDto.from(z)),
      count: total,
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
| Own column (boolean) | `{ column: zones.isActive, type: 'boolean' }` — auto-converts `'true'`/`1` |
| FK column | `{ column: zones.regionId, type: 'string' }` |
| Many-to-many join | `{ expression: (v) => sql\`...\`, type: 'string' }` |

Unknown fields are **silently skipped** (security whitelist — prevents SQL injection).
Expression fields are skipped by `buildSearch` and `buildOrderBy` (column-only operations).

---

## 4. Controller endpoint

Thin — log, extract `@UserId()`, return service result directly (no `return await`).

```typescript
// Returns paginated zones for the data table
@Get('table')
@ApiFindForTableZones()
findForTable(@UserId() userId: string): Promise<ZoneTableResponseDto> {
  this.logger.log('GET /admin-api/zones/table');
  return this.zoneService.findForTable(userId);
}
```

Create the Swagger decorator in `docs/zone.docs.ts`:
```typescript
export function ApiFindForTableZones() {
  return applyDecorators(
    ApiOperation({ summary: 'Get zones for data table' }),
    ApiResponse({ status: 200, type: ZoneTableResponseDto }),
  );
}
```

---

## Canonical reference

The **Region module** is the complete working example:
- `cloud-server/src/modules/admin-api/region/controllers/region.controller.ts`
- `cloud-server/src/modules/admin-api/region/services/region.service.ts`
- `cloud-server/src/modules/admin-api/region/repositories/region.repository.ts`
- `cloud-server/src/modules/admin-api/region/dto/response/regions-response.dto.ts`

Shared SDK types:
- `api-sdk/src/database/dto/table-response.dto.ts` — `TableResponseDto<T>` base class
- `api-sdk/src/database/filter/filter.processor.ts` — `FilterProcessor`, `FieldMap`, `FieldDefinition`

---

## 5. Frontend — Schema + Service + Hook

### Schema type alias

```typescript
// In schemas/admin/zones.ts
import type { TableResponse } from '@vritti/quantum-ui/api-response';

export interface Zone {
  id: string;
  name: string;
  code: string;
  // ...
}

export type ZonesTableResponse = TableResponse<Zone>;
```

### Service function

```typescript
// In services/admin/zones.service.ts
export function getZonesTable(): Promise<ZonesTableResponse> {
  return axios.get<ZonesTableResponse>('admin-api/zones/table').then((r) => r.data);
}
```

### Query hook

```typescript
// In hooks/admin/zones/useZones.ts
export const ZONES_TABLE_KEY = ['admin', 'zones', 'table'] as const;

export function useZones() {
  return useQuery<ZonesTableResponse>({
    queryKey: ZONES_TABLE_KEY,
    queryFn: getZonesTable,
  });
}
```

---

## 6. Frontend — Table Page

### Page structure

```typescript
import { useDeleteZone, useZones } from '@hooks/admin/zones';
import { ZONES_TABLE_KEY } from '@hooks/admin/zones/useZones';
import { useQueryClient } from '@tanstack/react-query';
import { Badge } from '@vritti/quantum-ui/Badge';
import { Button } from '@vritti/quantum-ui/Button';
import { type ColumnDef, DataTable, RowActions, useDataTable } from '@vritti/quantum-ui/DataTable';
import { Dialog } from '@vritti/quantum-ui/Dialog';
import { useConfirm, useDialog } from '@vritti/quantum-ui/hooks';
import { PageHeader } from '@vritti/quantum-ui/PageHeader';
import { buildSlug } from '@vritti/quantum-ui/utils/slug';
import { Eye, MapPin, Pencil, Plus, Trash2 } from 'lucide-react';
import { useNavigate } from 'react-router-dom';
import type { Zone } from '@/schemas/admin/zones';
import { AddZoneForm } from './forms/AddZoneForm';
import { EditZoneForm } from './forms/EditZoneForm';

const TABLE_SLUG = 'zones';

export const ZonesPage = () => {
  const navigate = useNavigate();
  const queryClient = useQueryClient();
  const { data: response, isLoading } = useZones();
  const confirm = useConfirm();
  const addDialog = useDialog();

  const deleteMutation = useDeleteZone();

  async function handleDelete(zone: Zone) {
    const confirmed = await confirm({
      title: `Delete ${zone.name}?`,
      description: 'This action cannot be undone.',
      confirmLabel: 'Delete',
      variant: 'destructive',
    });
    if (confirmed) deleteMutation.mutate(zone.id);
  }

  const { table } = useDataTable({
    columns: getColumns({
      onDelete: handleDelete,
      onView: (zone) => navigate(`/zones/${buildSlug(zone.name, zone.id)}`),
    }),
    slug: TABLE_SLUG,
    label: 'zone',
    serverState: response,
    enableRowSelection: false,
    enableSorting: true,
    enableMultiSort: false,
    onStatePush: () => queryClient.invalidateQueries({ queryKey: ZONES_TABLE_KEY }),
  });

  return (
    <div className="flex flex-col gap-6">
      <PageHeader title="Zones" description="Manage deployment zones" />
      <DataTable
        table={table}
        isLoading={isLoading}
        searchConfig={{
          columns: [
            { id: 'name', label: 'Name' },
            { id: 'code', label: 'Code' },
          ],
          searchAll: true,
        }}
        toolbarActions={{
          actions: (
            <Button startAdornment={<Plus className="size-4" />} size="sm" onClick={addDialog.open}>
              Add Zone
            </Button>
          ),
        }}
        emptyStateConfig={{
          icon: MapPin,
          title: 'No zones found',
          description: 'Create your first zone to get started.',
          action: (
            <Button size="sm" onClick={addDialog.open}>
              <Plus className="size-4" />
              Add Zone
            </Button>
          ),
        }}
      />
      <Dialog
        open={addDialog.isOpen}
        onOpenChange={(v) => { if (!v) addDialog.close(); }}
        title="Add Zone"
        description="Enter the details for the new zone."
        content={(close) => <AddZoneForm onSuccess={close} onCancel={close} />}
      />
    </div>
  );
};
```

### Action column — always use `RowActions`

`RowActions` auto-layouts based on visible action count:
- 0 → nothing
- 1 → single icon button
- 2 → two icon buttons side-by-side
- 3+ → first action as icon button + `⋮` overflow dropdown for the rest

Use `hidden` for feature-flag visibility. The component re-layouts automatically.

```typescript
interface ColumnActions {
  onDelete: (zone: Zone) => void;
  onView: (zone: Zone) => void;
}

function getColumns({ onDelete, onView }: ColumnActions): ColumnDef<Zone, unknown>[] {
  return [
    { accessorKey: 'name', header: 'Name' },
    {
      accessorKey: 'code',
      header: 'Code',
      cell: ({ row }) => (
        <Badge variant="outline" className="font-mono text-[10px] font-medium">
          {row.original.code}
        </Badge>
      ),
    },
    {
      id: 'actions',
      header: '',
      cell: ({ row }) => (
        <RowActions
          actions={[
            { id: 'view', icon: Eye, label: 'View', onClick: () => onView(row.original) },
            {
              id: 'edit',
              icon: Pencil,
              label: 'Edit',
              dialog: {
                title: 'Edit Zone',
                description: 'Update zone details.',
                content: (close) => <EditZoneForm zone={row.original} onSuccess={close} onCancel={close} />,
              },
            },
            {
              id: 'delete',
              icon: Trash2,
              label: 'Delete',
              variant: 'destructive',
              disabled: !row.original.canDelete,
              onClick: () => onDelete(row.original),
            },
          ]}
        />
      ),
      enableSorting: false,
      enableHiding: false,
    },
  ];
}
```

### RowAction props reference

| Prop | Type | Description |
|------|------|-------------|
| `id` | `string` | Unique identifier |
| `icon` | `LucideIcon` | Icon component |
| `label` | `string` | Accessible label / dropdown text |
| `onClick` | `() => void` | Click handler (mutually exclusive with `dialog`) |
| `dialog` | `{ title, description, content }` | Opens a dialog instead |
| `variant` | `'default' \| 'destructive'` | Styling variant |
| `disabled` | `boolean` | Disables the action |
| `hidden` | `boolean` | Hides the action (for feature flags) |

---

## Canonical references

**Backend** — the Region module:
- `cloud-server/src/modules/admin-api/region/controllers/region.controller.ts`
- `cloud-server/src/modules/domain/region/services/region.service.ts`
- `cloud-server/src/modules/domain/region/repositories/region.repository.ts`
- `cloud-server/src/modules/admin-api/region/dto/response/regions-response.dto.ts`

**Frontend** — the Industries page:
- `cloud-web/src/pages/admin/industries/IndustriesPage.tsx`
- `cloud-web/src/services/admin/industries.service.ts`
- `cloud-web/src/hooks/admin/industries/useIndustries.ts`
- `cloud-web/src/schemas/admin/industries.ts`

Shared SDK types:
- `api-sdk/src/database/dto/table-response.dto.ts` — `TableResponseDto<T>` base class
- `api-sdk/src/database/filter/filter.processor.ts` — `FilterProcessor`, `FieldMap`, `FieldDefinition`

---

Now implement the table endpoint for the resource the user described. Read the existing schema and entity DTO first, then build bottom-up: backend (DTO → repository → service → controller), then frontend (schema → service → hook → page).
