# Command Functions

Commands allow calling server functions from anywhere (not tied to forms). Prefer `form` when possible for graceful degradation.

## Basic Command

```typescript
// likes.remote.ts
import { z } from 'zod';
import { query, command } from '$app/server';
import * as db from '$lib/server/database';

export const getLikes = query(z.string(), async (id) => {
  const [row] = await db.sql`
    SELECT likes FROM item WHERE id = ${id}
  `;
  return row.likes;
});

export const addLike = command(z.string(), async (id) => {
  await db.sql`
    UPDATE item
    SET likes = likes + 1
    WHERE id = ${id}
  `;
});
```

```svelte
<script>
  import { getLikes, addLike } from './likes.remote';
  import { showToast } from '$lib/toast';
  
  let { item } = $props();
</script>

<button
  onclick={async () => {
    try {
      await addLike(item.id);
    } catch (error) {
      showToast('Something went wrong!');
    }
  }}
>
  add like
</button>

<p>likes: {await getLikes(item.id)}</p>
```

**Important:** Commands cannot be called during render.

## Single-Flight Mutations

By default, all queries refresh after a successful command. Optimize by specifying which queries to update.

### Server-side refresh:

```typescript
export const addLike = command(v.string(), async (id) => {
  await db.sql`
    UPDATE item SET likes = likes + 1 WHERE id = ${id}
  `;
  
  // Refresh only the specific query
  getLikes(id).refresh();
});
```

### Client-side refresh:

```typescript
try {
  await addLike(item.id).updates(getLikes(item.id));
} catch (error) {
  showToast('Something went wrong!');
}
```

### Optimistic updates:

```typescript
try {
  await addLike(item.id).updates(
    getLikes(item.id).withOverride((n) => n + 1)
  );
} catch (error) {
  showToast('Something went wrong!');
}
```

The override:
- Applies immediately
- Released when command completes
- Reverted on error

## Argument Validation

Always validate arguments using Standard Schema (Zod recommended):

```typescript
import { z } from 'zod';

// Simple string
export const deleteItem = command(z.string(), async (id) => {
  await db.delete(id);
});

// Object with validation
export const updateItem = command(
  z.object({
    id: z.string().uuid(),
    name: z.string().min(1).max(100),
    price: z.number().positive()
  }),
  async (data) => {
    await db.sql`
      UPDATE item
      SET name = ${data.name}, price = ${data.price}
      WHERE id = ${data.id}
    `;
  }
);

// Complex nested validation
export const createOrder = command(
  z.object({
    userId: z.string(),
    items: z.array(z.object({
      productId: z.string(),
      quantity: z.number().int().positive()
    })).min(1),
    shipping: z.object({
      address: z.string(),
      method: z.enum(['standard', 'express', 'overnight'])
    })
  }),
  async (order) => {
    // Process order...
  }
);
```

## No Redirects in Commands

Commands cannot redirect. If needed, return a redirect object and handle it client-side:

```typescript
export const doSomething = command(async () => {
  // ...
  return { redirect: '/success' };
});
```

```svelte
<script>
  const result = await doSomething();
  if (result.redirect) {
    goto(result.redirect);
  }
</script>
```

## Common Patterns

### Multiple query updates:

```typescript
await addLike(item.id).updates(
  getLikes(item.id),
  getStats()
);
```

### Conditional updates:

```typescript
const likes = await addLike(item.id);
if (likes > 100) {
  getPopularItems().refresh();
}
```

### Error recovery:

```typescript
try {
  await addLike(item.id).updates(
    getLikes(item.id).withOverride((n) => n + 1)
  );
} catch (error) {
  // Override is automatically reverted
  showToast('Failed to add like');
}
```
