# Form Functions

Form functions enable type-safe form submissions with progressive enhancement.

## Basic Form

```typescript
// data.remote.ts
import { error, redirect } from '@sveltejs/kit';
import { query, form } from '$app/server';
import * as db from '$lib/server/database';
import * as auth from '$lib/server/auth';

export const getPosts = query(async () => { /* ... */ });

export const createPost = form(async (data) => {
  // Check authentication
  const user = await auth.getUser();
  if (!user) error(401, 'Unauthorized');

  // Validate form data
  const title = data.get('title');
  const content = data.get('content');

  if (typeof title !== 'string' || typeof content !== 'string') {
    error(400, 'Title and content are required');
  }

  const slug = title.toLowerCase().replace(/ /g, '-');

  // Insert into database
  await db.sql`
    INSERT INTO post (slug, title, content)
    VALUES (${slug}, ${title}, ${content})
  `;

  // Redirect to new page
  redirect(303, `/blog/${slug}`);
});
```

## Sensitive Fields

Protect sensitive data by prefixing field names with underscore. These fields are never sent back to the client:

```typescript
import { form } from '$app/server';
import { z } from 'zod';

export const createAccount = form(
  z.object({
    email: z.string().email(),
    username: z.string().min(3),
    _password: z.string().min(8), // Not sent back to client
    _apiKey: z.string().optional() // Not sent back to client
  }),
  async (data) => {
    // data._password is available here
    const hashedPassword = await hash(data._password);

    await db.sql`
      INSERT INTO user (email, username, password_hash)
      VALUES (${data.email}, ${data.username}, ${hashedPassword})
    `;

    redirect(303, '/login');
  }
);
```

```svelte
<form {...createAccount}>
  <input name="email" type="email" />
  <input name="username" />
  <input name="_password" type="password" />
  <input name="_apiKey" placeholder="Optional API key" />
  <button>Create Account</button>
</form>

<!-- After submission, form fields are repopulated EXCEPT _password and _apiKey -->
{#if createAccount.result}
  <!-- email and username will be available, but _password and _apiKey won't -->
  <p>Account created for {createAccount.result.email}</p>
{/if}
```

**Why use sensitive fields:**
- Prevents passwords from being sent back to client after validation errors
- Keeps API keys and secrets from appearing in form repopulation
- Security best practice for handling credentials

```svelte
<!-- +page.svelte -->
<script>
  import { createPost } from '../data.remote';
</script>

<h1>Create a new post</h1>
<form {...createPost}>
  <label>
    <h2>Title</h2>
    <input name="title" />
  </label>
  
  <label>
    <h2>Write your post</h2>
    <textarea name="content"></textarea>
  </label>
  
  <button>Publish!</button>
</form>
```

The form object contains:
- `method` and `action` - enables no-JS fallback
- `onsubmit` - progressive enhancement handler

## Single-Flight Mutations

By default, all queries on the page refresh after form submission. Optimize by specifying which queries to refresh.

### Server-side refresh:

```typescript
export const createPost = form(async (data) => {
  // form logic...
  
  // Refresh only the posts query
  getPosts().refresh();
  
  redirect(303, `/blog/${slug}`);
});
```

### Client-side refresh (via enhance):

See the enhance section below.

## Returning Data

Instead of redirecting, return data to display in the UI:

```typescript
export const createPost = form(async (data) => {
  // form logic...
  
  return { success: true };
});
```

```svelte
<form {...createPost}>
  <!-- form fields -->
</form>

{#if createPost.result?.success}
  <p>Successfully published!</p>
{/if}
```

**Note:** The `result` value is ephemeral - it vanishes on resubmit, navigation, or reload.

Use `result` for:
- Success messages
- Validation errors
- Form repopulation data

## Form Enhancement

Customize submission behavior with `.enhance()`:

```svelte
<script>
  import { createPost } from '../data.remote';
  import { showToast } from '$lib/toast';
</script>

<form {...createPost.enhance(async ({ form, data, submit }) => {
  try {
    await submit();
    form.reset();
    showToast('Successfully published!');
  } catch (error) {
    showToast('Oh no! Something went wrong');
  }
})}>
  <input name="title" />
  <textarea name="content"></textarea>
  <button>publish</button>
</form>
```

The callback receives:
- `form` - The form element
- `data` - FormData object
- `submit()` - Function to submit

### Client-side single-flight mutations:

```typescript
await submit().updates(getPosts());
```

### Optimistic updates:

```typescript
await submit().updates(
  getPosts().withOverride((posts) => [newPost, ...posts])
);
```

The override applies immediately and releases when submission completes.

## Multiple Forms with buttonProps

Use different actions for buttons in the same form:

```svelte
<script>
  import { login, register } from '$lib/auth';
</script>

<form {...login}>
  <label>
    Your username
    <input name="username" />
  </label>

  <label>
    Your password
    <input name="password" type="password" />
  </label>

  <button>login</button>
  <button {...register.buttonProps}>register</button>
</form>
```

The `buttonProps` object also has an `enhance` method for customizing submission.

## Isolated Form Instances with form.for(id)

When rendering multiple forms (e.g., in a list), use `form.for(id)` to isolate their state:

```typescript
// todos.remote.ts
import { form, query } from '$app/server';
import { z } from 'zod';

export const getTodos = query(async () => {
  return await db.sql`SELECT * FROM todo`;
});

export const deleteTodo = form(
  z.object({ id: z.string() }),
  async (data) => {
    await db.sql`DELETE FROM todo WHERE id = ${data.id}`;
    getTodos().refresh();
  }
);
```

```svelte
<!-- +page.svelte -->
<script>
  import { getTodos, deleteTodo } from './todos.remote';
  const todos = getTodos();
</script>

<ul>
  {#each await todos as todo}
    <li>
      {todo.text}

      <!-- Each form has isolated state -->
      <form {...deleteTodo.for(todo.id)}>
        <input type="hidden" name="id" value={todo.id} />
        <button>Delete</button>
      </form>

      <!-- Check state of THIS specific form -->
      {#if deleteTodo.for(todo.id).pending}
        <span>Deleting...</span>
      {/if}
    </li>
  {/each}
</ul>
```

**Without .for(id):**
- All forms share same `pending` state
- Clicking any delete button shows "Deleting..." on ALL forms

**With .for(id):**
- Each form has independent `pending`, `result`, and `error` state
- Only the clicked form shows "Deleting..."
- Better UX for lists of items

**Common use cases:**
- Delete buttons in lists
- Toggle/star buttons per item
- Quick-edit forms inline
- Any scenario with multiple identical forms

## Error Handling

If an error occurs during submission, the nearest `+error.svelte` page is rendered.

## Progressive Enhancement

Forms work without JavaScript:
- `method` and `action` enable form submission
- Page reloads with server response

With JavaScript:
- `onsubmit` handler prevents reload
- Submits via fetch
- Updates UI without full page refresh
