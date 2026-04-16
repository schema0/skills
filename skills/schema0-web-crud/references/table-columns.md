# Table Columns

**File:** `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Column.tsx`

Columns define how data appears in the DataTable and include the actions column with edit/delete buttons.

## Full Columns Template

```tsx
// apps/web/src/components/ui/data-table/custom/entities/EntitiesColumn.tsx
import type { ColumnDef } from "@tanstack/react-table";
import type { z } from "zod/v4";
import type { selectEntitiesSchema } from "@template/db/schema";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import { Pencil, Trash2 } from "lucide-react";

type Entity = z.infer<typeof selectEntitiesSchema>;

interface EntitiesColumnMeta {
  onEdit?: (entity: Entity) => void;
  onDelete?: (entity: Entity) => void;
}

// Base columns -- data display without actions
export const entitiesBaseColumns: ColumnDef<Entity>[] = [
  {
    accessorKey: "name",
    header: "Name",
  },
  {
    accessorKey: "email",
    header: "Email",
  },
  {
    accessorKey: "description",
    header: "Description",
    cell: ({ row }) => {
      const value = row.getValue("description") as string | null;
      return value ?? "—";
    },
  },
  {
    accessorKey: "active",
    header: "Status",
    cell: ({ row }) => {
      const active = row.getValue("active") as boolean | null;
      return (
        <Badge variant={active ? "default" : "secondary"}>
          {active ? "Active" : "Inactive"}
        </Badge>
      );
    },
  },
  {
    accessorKey: "createdAt",
    header: "Created",
    cell: ({ row }) => {
      const date = row.getValue("createdAt") as Date;
      return date ? new Date(date).toLocaleDateString() : "—";
    },
  },
];

// Full columns with actions -- used in the list view
export function entitiesColumns(meta: EntitiesColumnMeta): ColumnDef<Entity>[] {
  return [
    ...entitiesBaseColumns,
    {
      id: "actions",
      header: "Actions",
      cell: ({ row }) => (
        <div className="flex items-center gap-2">
          <Button
            variant="ghost"
            size="icon"
            aria-label="Edit"
            onClick={() => meta.onEdit?.(row.original)}
          >
            <Pencil className="h-4 w-4" />
          </Button>
          <Button
            variant="ghost"
            size="icon"
            aria-label="Delete"
            onClick={() => meta.onDelete?.(row.original)}
          >
            <Trash2 className="h-4 w-4" />
          </Button>
        </div>
      ),
    },
  ];
}
```

## Column Customization Examples

Common cell renderers for different data types:

```tsx
// Date column
{
  accessorKey: "createdAt",
  header: "Created",
  cell: ({ row }) => {
    const date = row.getValue("createdAt") as Date;
    return date ? new Date(date).toLocaleDateString() : "—";
  },
}

// Status badge (boolean)
{
  accessorKey: "active",
  header: "Status",
  cell: ({ row }) => {
    const active = row.getValue("active") as boolean;
    return (
      <Badge variant={active ? "default" : "secondary"}>
        {active ? "Active" : "Inactive"}
      </Badge>
    );
  },
}

// Price / decimal
{
  accessorKey: "price",
  header: "Price",
  cell: ({ row }) => {
    const price = row.getValue("price") as string | null;
    return price ? `$${parseFloat(price).toFixed(2)}` : "—";
  },
}

// Email link
{
  accessorKey: "email",
  header: "Email",
  cell: ({ row }) => {
    const email = row.getValue("email") as string;
    return <a href={`mailto:${email}`} className="text-primary underline">{email}</a>;
  },
}

// URL link
{
  accessorKey: "website",
  header: "Website",
  cell: ({ row }) => {
    const url = row.getValue("website") as string | null;
    return url ? (
      <a href={url} target="_blank" rel="noopener noreferrer" className="text-primary underline">
        {new URL(url).hostname}
      </a>
    ) : "—";
  },
}

// Avatar with fallback
{
  accessorKey: "avatarUrl",
  header: "Avatar",
  cell: ({ row }) => {
    const url = row.getValue("avatarUrl") as string | null;
    const name = row.getValue("name") as string;
    return url ? (
      <img src={url} alt={name} className="h-8 w-8 rounded-full" />
    ) : (
      <div className="flex h-8 w-8 items-center justify-center rounded-full bg-muted text-xs">
        {name?.charAt(0)?.toUpperCase()}
      </div>
    );
  },
}

// Truncated text
{
  accessorKey: "description",
  header: "Description",
  cell: ({ row }) => {
    const text = row.getValue("description") as string | null;
    if (!text) return "—";
    return text.length > 50 ? `${text.slice(0, 50)}...` : text;
  },
}
```

## Column Key Rules

- Actions column uses `aria-label="Edit"` and `aria-label="Delete"` -- these labels are used by tests to find buttons.
- Delete button calls `onDelete?.(row.original)` -- passes the full object, NOT just `row.original.id`.
- NEVER use `confirm()` for delete -- use `onDelete` callback which triggers an AlertDialog in the view.
- The `meta` pattern keeps columns pure; the view provides the callbacks.
