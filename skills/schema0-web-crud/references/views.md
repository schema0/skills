# Views (Route Components)

## List Route

**File:** `apps/web/src/routes/_auth.{entities}.tsx`

The list route is the main page for an entity. It uses `useLiveQuery` for real-time data, DataTable for display, Dialog for create/edit, and AlertDialog for delete confirmation.

**CRITICAL:** The `loader` function MUST be exported as a named export. React Router v7 requires it.

```tsx
// apps/web/src/routes/_auth.entities.tsx
import { useState } from "react";
import { useLiveQuery } from "@tanstack/react-db";
import { useCollection } from "@tanstack/react-db";
import { DataTable } from "@/components/ui/data-table";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { authContext } from "@/context";
import { entitiesCollection } from "@/query-collections/custom/entities";
import { EntitiesDialog } from "@/components/ui/data-table/custom/entities";
import { entitiesColumns } from "@/components/ui/data-table/custom/entities";
import type { Route } from "./+types/_auth.entities";
import type { z } from "zod/v4";
import type { selectEntitiesSchema } from "@template/db/schema";

type Entity = z.infer<typeof selectEntitiesSchema>;

// MUST be exported -- React Router v7 requires named export
export function loader({ context }: Route.LoaderArgs) {
  const auth = authContext.get(context);
  return { user: auth.user };
}

export default function EntitiesPage({ loaderData }: Route.ComponentProps) {
  const { user } = loaderData;

  // Live query -- reacts to optimistic updates in real time
  const { rows: entities } = useLiveQuery((query) =>
    query.from({ entities: entitiesCollection }),
  );

  const collection = useCollection(entitiesCollection);

  // Edit dialog state
  const [editEntity, setEditEntity] = useState<Entity | null>(null);
  const [editOpen, setEditOpen] = useState(false);

  // Delete confirmation state
  const [deleteEntity, setDeleteEntity] = useState<Entity | null>(null);
  const [deleteOpen, setDeleteOpen] = useState(false);

  const handleEdit = (entity: Entity) => {
    setEditEntity(entity);
    setEditOpen(true);
  };

  const handleDelete = (entity: Entity) => {
    setDeleteEntity(entity);
    setDeleteOpen(true);
  };

  const confirmDelete = () => {
    if (deleteEntity) {
      collection.delete(deleteEntity.id);
    }
    setDeleteOpen(false);
    setDeleteEntity(null);
  };

  const columns = entitiesColumns({
    onEdit: handleEdit,
    onDelete: handleDelete,
  });

  return (
    <div className="container mx-auto py-6 space-y-4">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Entities</h1>
        <EntitiesDialog mode="create" userId={user.id} />
      </div>

      <DataTable columns={columns} data={entities} />

      {/* Edit Dialog -- controlled via state */}
      {editEntity && (
        <Dialog open={editOpen} onOpenChange={setEditOpen}>
          <DialogContent>
            <DialogHeader>
              <DialogTitle>Edit Entity</DialogTitle>
            </DialogHeader>
            <EntitiesForm
              mode="edit"
              defaultValues={editEntity as Record<string, unknown>}
              onSubmit={(data) => {
                collection.update(editEntity.id, {
                  ...data,
                  updatedAt: new Date(),
                });
                setEditOpen(false);
                setEditEntity(null);
              }}
            />
          </DialogContent>
        </Dialog>
      )}

      {/* Delete Confirmation */}
      <AlertDialog open={deleteOpen} onOpenChange={setDeleteOpen}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Delete Entity</AlertDialogTitle>
            <AlertDialogDescription>
              Are you sure you want to delete "{deleteEntity?.name}"? This
              action cannot be undone.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>Cancel</AlertDialogCancel>
            <AlertDialogAction onClick={confirmDelete}>
              Delete
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  );
}
```

## Detail Route

**File:** `apps/web/src/routes/_auth.{entities}_.$id.tsx`

The detail route shows a single entity by ID. It loads the entity via the loader and displays its fields.

```tsx
// apps/web/src/routes/_auth.entities_.$id.tsx
import { authContext } from "@/context";
import { client } from "@/utils/orpc";
import type { Route } from "./+types/_auth.entities_.$id";

// MUST be exported
export async function loader({ context, params }: Route.LoaderArgs) {
  const auth = authContext.get(context);
  const entity = await client.entities.selectById({ id: params.id });
  if (!entity) {
    throw new Response("Not Found", { status: 404 });
  }
  return { user: auth.user, entity };
}

export default function EntityDetailPage({ loaderData }: Route.ComponentProps) {
  const { entity } = loaderData;

  return (
    <div className="container mx-auto py-6 space-y-4">
      <h1 className="text-2xl font-bold">{entity.name}</h1>
      <div className="grid gap-4">
        <div>
          <span className="text-muted-foreground">Email:</span> {entity.email}
        </div>
        {entity.description && (
          <div>
            <span className="text-muted-foreground">Description:</span>{" "}
            {entity.description}
          </div>
        )}
        <div>
          <span className="text-muted-foreground">Status:</span>{" "}
          {entity.active ? "Active" : "Inactive"}
        </div>
        <div>
          <span className="text-muted-foreground">Created:</span>{" "}
          {new Date(entity.createdAt).toLocaleDateString()}
        </div>
      </div>
    </div>
  );
}
```

## Sidebar Entry

After creating all files, add the entity to the sidebar:

```tsx
// apps/web/src/components/app-sidebar.tsx
// Add to the navigation items array:
{
  title: "Entities",
  url: "/entities",
  icon: SomeIcon,  // from lucide-react
}
```

## View Key Rules

1. **`loader` MUST be exported** as a named export -- React Router v7 requires this.
2. **Use `authContext.get(context)`** to access the authenticated user in loaders.
3. **Use `useLiveQuery`** from `@tanstack/react-db` for real-time data in list views.
4. **Use `useCollection`** from `@tanstack/react-db` for mutation operations (insert, update, delete).
5. **Delete uses AlertDialog** (2-step: row button opens AlertDialog, confirm button executes). NEVER use `window.confirm()`.
6. **Never perform database operations in loaders** -- loaders only call ORPC client methods.
7. **Ignore `./+types/` import errors** -- these types are generated at build time.
