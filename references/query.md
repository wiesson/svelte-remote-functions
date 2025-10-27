# Query Functions

Query functions allow type-safe data fetching from the server. They run on the server and can access server-only modules.

## Basic Query

```typescript
// data.remote.ts
import { query } from '$app/server';
import * as db from '$lib/server/database';

export const getPosts = query(async () => {
  const posts = await db.sql`
    SELECT title, slug FROM post
    ORDER BY published_at DESC
  `;
  return posts;
});
```

```svelte
<!-- +page.svelte -->
<script>
  import { getPosts } from './data.remote';
</script>

<ul>
  {#each await getPosts() as { title, slug }}
    <li><a href="/blog/{slug}">{title}</a></li>
  {/each}
</ul>
```

## Batch Queries (N+1 Prevention)

Use `query.batch()` to prevent N+1 query problems when fetching multiple items:

```typescript
import { query } from '$app/server';
import { z } from 'zod';
import * as db from '$lib/server/database';

// Instead of individual queries per item
export const getAuthor = query.batch(
  z.string(), // Validation schema for each argument
  async (authorIds) => {
    // Receives array of all requested IDs
    const authors = await db.sql`
      SELECT * FROM author WHERE id = ANY(${authorIds})
    `;

    // Return results in same order as input
    return authorIds.map(id =>
      authors.find(author => author.id === id)
    );
  }
);
```

```svelte
<!-- Client: +page.svelte -->
<script>
  import { getAuthor } from './data.remote';
  let { posts } = $props();
</script>

<!-- Each call batches into single DB query -->
{#each posts as post}
  <article>
    <h2>{post.title}</h2>
    <p>By {await getAuthor(post.author_id).name}</p>
  </article>
{/each}
```

**How it works:**
- Multiple `getAuthor()` calls in the same tick are batched
- Server receives all IDs at once: `['id1', 'id2', 'id3']`
- Single database query fetches all authors
- Results distributed back to each call site

**When to use:**
- Rendering lists where each item needs related data
- Preventing N+1 query problems
- Fetching user profiles, categories, or any relational data in loops

## Query with Arguments

Validate arguments using Standard Schema libraries (Zod recommended):

```typescript
import { z } from 'zod';
import { error } from '@sveltejs/kit';
import { query } from '$app/server';
import * as db from '$lib/server/database';

export const getPost = query(z.string(), async (slug) => {
  const [post] = await db.sql`
    SELECT * FROM post WHERE slug = ${slug}
  `;
  
  if (!post) error(404, 'Not found');
  return post;
});
```

```svelte
<script>
  import { getPost } from '../data.remote';
  let { params } = $props();
</script>

<h1>{await getPost(params.slug).title}</h1>
<div>{@html await getPost(params.slug).content}</div>
```

Or store the query:

```svelte
<script>
  import { getPost } from '../data.remote';
  let { params } = $props();
  
  const post = getPost(params.slug);
</script>

<h1>{await post.title}</h1>
<div>{@html await post.content}</div>
```

## Alternative: Properties API

Instead of `{#each await query()}`, you can use properties:

```svelte
<script>
  import { getPosts } from './data.remote';
  const posts = getPosts();
</script>

{#if posts.error}
  <p>oops!</p>
{:else if posts.loading}
  <p>loading...</p>
{:else}
  <ul>
    {#each posts.current as { title, slug }}
      <li><a href="/blog/{slug}">{title}</a></li>
    {/each}
  </ul>
{/if}
```

## Using svelte:boundary

For better error handling and loading states with await:

```svelte
<script>
  import { getPost } from '../data.remote';
  let { params } = $props();
</script>

<svelte:boundary>
  {#await getPost(params.slug) then post}
    <h1>{post.title}</h1>
    <div>{@html post.content}</div>
  {/await}
  
  {#snippet pending()}
    <p>Loading post...</p>
  {/snippet}
  
  {#snippet failed(error)}
    <p>Error: {error.message}</p>
  {/snippet}
</svelte:boundary>
```

## Refreshing Queries

Manually refresh a query:

```svelte
<script>
  import { getPosts } from './data.remote';
  const posts = getPosts();
</script>

<button onclick={() => posts.refresh()}>
  Check for new posts
</button>
```

**Important:** Queries are cached while on the page, so `getPosts() === getPosts()`. You can either store a reference or call `getPosts().refresh()` directly.

## Data Serialization

Both arguments and return values are serialized with [devalue](https://github.com/sveltejs/devalue), supporting:
- JSON types
- `Date` objects
- `Map` and `Set`
- Custom types via transport hook

## Using getRequestEvent

Access the current request context inside queries:

```typescript
import { getRequestEvent, query } from '$app/server';
import { findUser } from '$lib/server/database';

export const getProfile = query(async () => {
  const { cookies, locals } = getRequestEvent();
  const user = await getUser();
  
  return {
    name: user.name,
    avatar: user.avatar
  };
});

function getUser() {
  const { cookies, locals } = getRequestEvent();
  locals.userPromise ??= findUser(cookies.get('session_id'));
  return await locals.userPromise;
}
```

**Note:** Inside remote functions, `RequestEvent` differs:
- No `params` or `route.id`
- Cannot set headers (except cookies in `form`/`command`)
- `url.pathname` is always `/`

## Redirects

Use `redirect()` inside queries:

```typescript
import { query, redirect } from '$app/server';

export const getProtectedData = query(async () => {
  const user = await getUser();
  if (!user) redirect(303, '/login');
  
  return user.data;
});
```

## Validation Error Handling

By default, invalid arguments return 400 Bad Request. Customize via `handleValidationError` hook:

```typescript
// hooks.server.ts
export function handleValidationError({ event, issues }) {
  return {
    message: 'Invalid request'
  };
}
```

To skip validation (not recommended):

```typescript
export const getStuff = query('unchecked', async ({ id }: { id: string }) => {
  // No validation - handle security manually
});
```

## Complex Validation Examples

```typescript
import { z } from 'zod';

// String with constraints
export const getUser = query(
  z.string().email(),
  async (email) => { }
);

// Number with range
export const getItems = query(
  z.number().min(1).max(100),
  async (limit) => { }
);

// Object with nested validation
export const search = query(
  z.object({
    query: z.string().min(1),
    filters: z.object({
      category: z.enum(['tech', 'design', 'business']),
      minPrice: z.number().optional(),
      maxPrice: z.number().optional()
    }).optional(),
    page: z.number().default(1)
  }),
  async (params) => { }
);

// Array validation
export const getBulkUsers = query(
  z.array(z.string().uuid()),
  async (userIds) => { }
);

// Union types
export const getResource = query(
  z.union([z.string(), z.number()]),
  async (id) => { }
);
```
