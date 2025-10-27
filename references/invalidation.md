# Invalidation & Refetch Patterns

Complete guide to refreshing and invalidating query data in SvelteKit Remote Functions.

## Manual Query Refresh

Call `.refresh()` on a query:

```svelte
<script>
  import { getPosts } from './data.remote';
  const posts = getPosts();
</script>

<button onclick={() => posts.refresh()}>
  Check for new posts
</button>
```

**Remember:** Queries are cached, so `getPosts() === getPosts()`. You can store a reference or call `getPosts().refresh()` directly.

## Automatic Refresh After Mutations

### Default Behavior

By default, ALL queries on the page (and load functions) refresh after:
- Successful form submission
- Successful command execution

This is inefficient but safe - everything stays up-to-date.

## Single-Flight Mutations (Recommended)

Specify exactly which queries to refresh. This:
- Reduces unnecessary requests
- Returns updated data in the same response
- Avoids second server round-trip

### Server-side (Form)

```typescript
export const createPost = form(async (data) => {
  // Insert post...
  
  // Refresh specific query on server
  getPosts().refresh();
  
  redirect(303, `/blog/${slug}`);
});
```

### Server-side (Command)

```typescript
export const addLike = command(v.string(), async (id) => {
  await db.sql`
    UPDATE item SET likes = likes + 1 WHERE id = ${id}
  `;
  
  // Refresh specific query
  getLikes(id).refresh();
});
```

### Client-side (Form with enhance)

```svelte
<form {...createPost.enhance(async ({ submit }) => {
  // Specify which queries to update
  await submit().updates(getPosts());
})}>
  <!-- form fields -->
</form>
```

### Client-side (Command)

```typescript
await addLike(item.id).updates(getLikes(item.id));
```

## Optimistic Updates

Update UI immediately, revert on error:

### Form Example

```svelte
<script>
  import { getTodos, addTodo } from './todos.remote';
  
  const todos = getTodos();
</script>

<form {...addTodo.enhance(async ({ data, submit }) => {
  const newTodo = { text: data.get('text'), done: false };
  
  await submit().updates(
    todos.withOverride((current) => [...current, newTodo])
  );
})}>
  <input name="text" />
  <button>Add</button>
</form>

<ul>
  {#each await todos as todo}
    <li>{todo.text}</li>
  {/each}
</ul>
```

### Command Example

```typescript
try {
  await addLike(item.id).updates(
    getLikes(item.id).withOverride((n) => n + 1)
  );
} catch (error) {
  // Override automatically reverted
  showToast('Failed to add like');
}
```

## Multiple Query Updates

Update several queries at once:

```svelte
<form {...createPost.enhance(async ({ submit }) => {
  await submit().updates(
    getPosts(),
    getStats(),
    getRecentActivity()
  );
})}>
  <!-- form fields -->
</form>
```

```typescript
await addLike(item.id).updates(
  getLikes(item.id),
  getStats(),
  getTrendingItems()
);
```

## Query Override Patterns

### Append to list:

```typescript
todos.withOverride((current) => [...current, newTodo])
```

### Prepend to list:

```typescript
posts.withOverride((current) => [newPost, ...current])
```

### Update item in list:

```typescript
todos.withOverride((current) =>
  current.map((todo) =>
    todo.id === updatedId ? { ...todo, done: true } : todo
  )
)
```

### Remove from list:

```typescript
todos.withOverride((current) =>
  current.filter((todo) => todo.id !== deletedId)
)
```

### Increment number:

```typescript
getLikes(id).withOverride((n) => n + 1)
```

## Common Invalidation Scenarios

### After creating an item:

```typescript
// Server-side
export const createPost = form(async (data) => {
  // ... create post
  getPosts().refresh();
  redirect(303, `/blog/${slug}`);
});

// Or client-side
await submit().updates(getPosts());
```

### After updating an item:

```typescript
// Refresh both list and detail
await updatePost(id, data).updates(
  getPosts(),
  getPost(id)
);
```

### After deleting an item:

```typescript
// Optimistic removal
await deletePost(id).updates(
  getPosts().withOverride((posts) =>
    posts.filter((p) => p.id !== id)
  )
);
```

### Conditional refresh:

```typescript
export const addLike = command(v.string(), async (id) => {
  const newLikes = await db.incrementLikes(id);
  
  // Only refresh popular items if threshold crossed
  if (newLikes >= 100) {
    getPopularItems().refresh();
  }
  
  getLikes(id).refresh();
});
```

## Performance Tips

1. **Use single-flight mutations** - Avoid refreshing all queries
2. **Be specific** - Only refresh queries that changed
3. **Use optimistic updates** - Better UX, instant feedback
4. **Batch updates** - Update multiple queries in one call
5. **Server-side refresh when possible** - Reduces client complexity

## Debugging

Check if queries are refreshing unnecessarily:

```svelte
<script>
  import { getPosts } from './data.remote';
  
  $effect(() => {
    console.log('Posts refreshed:', getPosts());
  });
</script>
```
