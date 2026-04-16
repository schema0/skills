# Collections, Dialogs, Forms & Index Export

## 1. Query Collections

**File:** `apps/web/src/query-collections/custom/{entities}.ts`

Collections bridge the API router to TanStack DB for optimistic UI updates. Each collection defines `onInsert`, `onUpdate`, `onDelete` mutation handlers and a `queryFn` for fetching data.

### Full Collection Template

```typescript
// apps/web/src/query-collections/custom/entities.ts
import { queryCollectionOptions } from "@tanstack/react-db";
import { client, queryClient } from "@/utils/orpc";
import {
  selectEntitiesSchema,
  insertEntitiesSchema,
  updateEntitiesSchema,
} from "@template/db/schema";

export const entitiesCollection = queryCollectionOptions({
  id: "entities",
  queryKey: ["entities"],
  queryFn: async (ctx) => {
    return await client.entities.selectAll({});
  },
  getId: (item) => item.id,
  schema: selectEntitiesSchema,
  onInsert: async (tx) => {
    const records = tx.insertions.map((item) => ({
      id: item.id,
      name: item.name,
      description: item.description ?? null,
      email: item.email,
      price: item.price ?? null,
      active: item.active ?? true,
      userId: item.userId ?? null,
      createdAt: item.createdAt ?? new Date(),
      updatedAt: item.updatedAt ?? new Date(),
    }));
    await client.entities.insertMany(records);
    await queryClient.invalidateQueries({ queryKey: ["entities"] });
  },
  onUpdate: async (tx) => {
    const updates = tx.updates.map((u) => ({
      id: u.id,
      ...u.changes,
    }));
    await client.entities.updateMany(updates);
    await queryClient.invalidateQueries({ queryKey: ["entities"] });
  },
  onDelete: async (tx) => {
    const ids = tx.deletions.map((item) => ({ id: item.id }));
    await client.entities.deleteMany(ids);
    await queryClient.invalidateQueries({ queryKey: ["entities"] });
  },
  queryClient,
});
```

### Collection Key Rules

- `id` and `queryKey` must match each other and match the `invalidateQueries` call in tests.
- `queryFn` MUST use `client.{entity}.selectAll({})` directly -- never `fetchCustomResources`.
- `schema` uses `selectEntitiesSchema` (the select schema from the DB package).
- `onInsert` maps form data to the full record shape including system fields.
- `onUpdate` maps update data and always includes `id`.
- `onDelete` maps items to `{ id }` objects.
- Always `invalidateQueries` after mutations to refresh the cache.

---

## 2. Dialog

**File:** `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Dialog.tsx`

The Dialog wraps the Form component, handles create vs edit mode, and calls the collection's insert/update.

```tsx
// apps/web/src/components/ui/data-table/custom/entities/EntitiesDialog.tsx
import { useState } from "react";
import { useCollection } from "@tanstack/react-db";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";
import { entitiesCollection } from "@/query-collections/custom/entities";
import { EntitiesForm } from "./EntitiesForm";
import type { z } from "zod/v4";
import type {
  entitiesFormSchema,
  entitiesEditFormSchema,
} from "@template/db/schema";

interface EntitiesDialogProps {
  mode: "create" | "edit";
  entityData?: Record<string, unknown>; // existing data for edit mode
  trigger?: React.ReactNode;
  userId: string;
}

export function EntitiesDialog({
  mode,
  entityData,
  trigger,
  userId,
}: EntitiesDialogProps) {
  const [open, setOpen] = useState(false);
  const collection = useCollection(entitiesCollection);

  const handleCreate = (data: z.infer<typeof entitiesFormSchema>) => {
    collection.insert([
      {
        ...data,
        id: crypto.randomUUID(),
        userId,
        createdAt: new Date(),
        updatedAt: new Date(),
      },
    ]);
    setOpen(false);
  };

  const handleEdit = (data: z.infer<typeof entitiesEditFormSchema>) => {
    if (!entityData?.id) return;
    collection.update(entityData.id as string, {
      ...data,
      updatedAt: new Date(),
    });
    setOpen(false);
  };

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        {trigger ?? (
          <Button>{mode === "create" ? "Add Entity" : "Edit"}</Button>
        )}
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>
            {mode === "create" ? "Create Entity" : "Edit Entity"}
          </DialogTitle>
        </DialogHeader>
        <EntitiesForm
          mode={mode}
          defaultValues={mode === "edit" ? entityData : undefined}
          onSubmit={mode === "create" ? handleCreate : handleEdit}
        />
      </DialogContent>
    </Dialog>
  );
}
```

---

## 3. Form

**File:** `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Form.tsx`

The Form uses react-hook-form with zodResolver. It handles both create and edit modes via schema switching.

```tsx
// apps/web/src/components/ui/data-table/custom/entities/EntitiesForm.tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import type { z } from "zod/v4";
import {
  entitiesFormSchema,
  entitiesEditFormSchema,
} from "@template/db/schema";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Textarea } from "@/components/ui/textarea";
import { Switch } from "@/components/ui/switch";

type CreateFormData = z.infer<typeof entitiesFormSchema>;
type EditFormData = z.infer<typeof entitiesEditFormSchema>;

interface EntitiesFormProps {
  mode: "create" | "edit";
  defaultValues?: Record<string, unknown>;
  onSubmit: (data: CreateFormData | EditFormData) => void;
}

export function EntitiesForm({
  mode,
  defaultValues,
  onSubmit,
}: EntitiesFormProps) {
  const schema =
    mode === "create" ? entitiesFormSchema : entitiesEditFormSchema;

  const form = useForm({
    resolver: zodResolver(schema),
    defaultValues: (defaultValues ?? {}) as Record<string, unknown>,
  });

  const handleSubmit = form.handleSubmit(
    (data) => {
      onSubmit(data as CreateFormData | EditFormData);
    },
    (errors) => {
      // onInvalid callback -- logs form validation errors for debugging
      console.error("Form validation errors:", errors);
    },
  );

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      {/* Name field -- required in create, optional in edit */}
      <div className="space-y-2">
        <Label htmlFor="name">Name</Label>
        <Input id="name" {...form.register("name")} placeholder="Enter name" />
        {form.formState.errors.name && (
          <p className="text-sm text-destructive">
            {form.formState.errors.name.message as string}
          </p>
        )}
      </div>

      {/* Email field -- only shown in create mode (omitted from edit schema) */}
      {mode === "create" && (
        <div className="space-y-2">
          <Label htmlFor="email">Email</Label>
          <Input
            id="email"
            type="email"
            {...form.register("email")}
            placeholder="Enter email"
          />
          {form.formState.errors.email && (
            <p className="text-sm text-destructive">
              {form.formState.errors.email.message as string}
            </p>
          )}
        </div>
      )}

      {/* Description field -- optional textarea */}
      <div className="space-y-2">
        <Label htmlFor="description">Description</Label>
        <Textarea
          id="description"
          {...form.register("description")}
          placeholder="Enter description"
        />
      </div>

      {/* Active toggle -- boolean with Switch */}
      <div className="flex items-center space-x-2">
        <Switch
          id="active"
          checked={form.watch("active") as boolean}
          onCheckedChange={(checked) => form.setValue("active", checked)}
        />
        <Label htmlFor="active">Active</Label>
      </div>

      <Button type="submit">{mode === "create" ? "Create" : "Save"}</Button>
    </form>
  );
}
```

### Form Key Rules

- `zodResolver(schema)` uses the correct schema per mode: `formSchema` for create, `editFormSchema` for edit.
- `handleSubmit` MUST include the `onInvalid` callback for debugging: `form.handleSubmit(onValid, onInvalid)`.
- The edit form schema MUST NOT include `id` -- the Dialog adds it after submission.
- Default values for edit mode come from the existing row data.
- The `"Form validation errors:"` console message is used by tests (Layer 1 validation).

---

## 4. Index Export

```typescript
// apps/web/src/components/ui/data-table/custom/entities/index.ts
export { EntitiesDialog } from "./EntitiesDialog";
export { EntitiesForm } from "./EntitiesForm";
export { entitiesColumns, entitiesBaseColumns } from "./EntitiesColumn";
```
