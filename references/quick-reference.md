# Quick Reference

## File Naming

Remote functions must be in `.remote.js` or `.remote.ts` files.

## Query

```typescript
// Server: data.remote.ts
import { query } from '$app/server';
import { z } from 'zod';

// No arguments
export const getAll = query(async () => {
  return await db.getAll();
});

// With validation
export const getOne = query(z.string(), async (id) => {
  return await db.getById(id);
});

// Batch queries (prevents N+1)
export const getAuthor = query.batch(
  z.string(),
  async (authorIds) => {
    const authors = await db.getAuthorsByIds(authorIds);
    return authorIds.map(id => authors.find(a => a.id === id));
  }
);
```

```svelte
<!-- Client: +page.svelte -->
<script>
  import { getAll, getOne } from './data.remote';
</script>

<!-- Using await directly -->
{#each await getAll() as item}
  <div>{item.name}</div>
{/each}

<!-- Using await with svelte:boundary -->
<svelte:boundary>
  <p>{await getOne(id)}</p>
  {#snippet pending()}
    <p>loading...</p>
  {/snippet}
  {#snippet failed(error)}
    <p>error: {error.message}</p>
  {/snippet}
</svelte:boundary>

<!-- Using properties -->
<script>
  const query = getAll();
</script>

{#if query.loading}
  Loading...
{:else if query.error}
  Error!
{:else}
  {#each query.current as item}
    <div>{item.name}</div>
  {/each}
{/if}

<!-- Manual refresh -->
<button onclick={() => query.refresh()}>
  Refresh
</button>
```

## Form

```typescript
// Server: data.remote.ts
import { form } from '$app/server';
import { redirect } from '@sveltejs/kit';
import { z } from 'zod';

export const create = form(async (data) => {
  const title = data.get('title');
  await db.insert(title);

  // Option 1: Redirect
  redirect(303, '/success');

  // Option 2: Return data
  // return { success: true };
});

// With validation and sensitive fields
export const signup = form(
  z.object({
    email: z.string().email(),
    _password: z.string().min(8) // Not sent back to client
  }),
  async (data) => {
    await db.createUser(data.email, data._password);
    redirect(303, '/login');
  }
);
```

```svelte
<!-- Client: +page.svelte -->
<script>
  import { create } from './data.remote';
</script>

<!-- Basic -->
<form {...create}>
  <input name="title" />
  <button>Submit</button>
</form>

<!-- With enhance -->
<form {...create.enhance(async ({ form, submit }) => {
  await submit();
  form.reset();
})}>
  <input name="title" />
  <button>Submit</button>
</form>

<!-- Isolated form instances (for lists) -->
{#each items as item}
  <form {...deleteItem.for(item.id)}>
    <button>Delete</button>
    {#if deleteItem.for(item.id).pending}
      Deleting...
    {/if}
  </form>
{/each}

<!-- Show result -->
{#if create.result?.success}
  Success!
{/if}
```

## Command

```typescript
// Server: data.remote.ts
import { command } from '$app/server';
import { z } from 'zod';

export const doAction = command(z.string(), async (id) => {
  await db.update(id);
});
```

```svelte
<!-- Client: +page.svelte -->
<script>
  import { doAction } from './data.remote';
</script>

<button onclick={() => doAction(item.id)}>
  Do it
</button>
```

## Invalidation

```typescript
// Server-side (in form/command)
export const create = form(async (data) => {
  await db.insert(data);
  getAll().refresh(); // ← Refresh specific query
  redirect(303, '/');
});
```

```svelte
<!-- Client-side (form) -->
<form {...create.enhance(async ({ submit }) => {
  await submit().updates(getAll());
})}>
</form>

<!-- Client-side (command) -->
<button onclick={async () => {
  await doAction(id).updates(getAll());
}}>
</button>
```

## Optimistic Updates

```svelte
<script>
  import { getTodos, addTodo } from './todos.remote';
  const todos = getTodos();
</script>

<form {...addTodo.enhance(async ({ data, submit }) => {
  await submit().updates(
    todos.withOverride((current) => [
      ...current,
      { text: data.get('text') }
    ])
  );
})}>
  <input name="text" />
  <button>Add</button>
</form>
```

## Validation Schemas

```typescript
import { z } from 'zod';

// String
query(z.string(), async (str) => { })

// Number
query(z.number(), async (num) => { })

// Email
query(z.string().email(), async (email) => { })

// UUID
query(z.string().uuid(), async (id) => { })

// Object
query(z.object({
  id: z.string(),
  name: z.string()
}), async (obj) => { })

// Sensitive fields (not sent back to client)
form(z.object({
  username: z.string(),
  _password: z.string().min(8),
  _apiKey: z.string().optional()
}), async (data) => { })

// Array
query(z.array(z.string()), async (items) => { })

// Optional
query(z.string().optional(), async (str) => { })

// With defaults
query(z.object({
  limit: z.number().default(10),
  offset: z.number().default(0)
}), async (params) => { })

// Enum
query(z.enum(['active', 'archived', 'deleted']), async (status) => { })

// Union
query(z.union([z.string(), z.number()]), async (id) => { })

// Complex nested
query(z.object({
  user: z.object({
    email: z.string().email(),
    age: z.number().min(18)
  }),
  preferences: z.object({
    theme: z.enum(['light', 'dark']),
    notifications: z.boolean()
  }).optional()
}), async (data) => { })

// Skip validation (not recommended)
query('unchecked', async (data: any) => { })
```

## Custom Validation Error Handling

```typescript
// hooks.server.ts
export function handleValidationError({ event, issues }) {
  // Hide details from client (recommended for security)
  return {
    message: 'Invalid request',
    code: 'VALIDATION_ERROR'
  };
}
```

## Common Patterns

### Create → Redirect + Refresh
```typescript
export const create = form(async (data) => {
  await db.insert(data);
  getAll().refresh();
  redirect(303, '/list');
});
```

### Update → Refresh Multiple
```typescript
await update(id, data).updates(
  getOne(id),
  getAll()
);
```

### Delete → Optimistic Remove
```typescript
await deleteItem(id).updates(
  getAll().withOverride((items) =>
    items.filter((i) => i.id !== id)
  )
);
```

### Like → Optimistic Increment
```typescript
await addLike(id).updates(
  getLikes(id).withOverride((n) => n + 1)
);
```
