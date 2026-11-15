# Svelte / SvelteKit Code Review Guide

Svelte 5 / SvelteKit review focus areas: the Runes reactivity system, Server/Client boundaries, Form Actions, Store migration, and security.

## Table of Contents

- [Runes: $state / $derived / $effect](#runes-state--derived--effect)
- [Load Functions (Server vs Client)](#load-functions-server-vs-client)
- [Form Actions](#form-actions)
- [Store Migration (→ $state)](#store-migration--state)
- [SSR vs CSR Boundaries](#ssr-vs-csr-boundaries)
- [Reactive Statement Migration ($: → Runes)](#reactive-statement-migration)
- [Performance Optimization](#performance-optimization)
- [Security Review](#security-review)
- [Review Checklist](#review-checklist)

---

## Runes: $state / $derived / $effect

### $state Basics

```svelte
<!-- ❌ Using $state for values that never change -->
<script lang="ts">
  let config = $state({ timeout: 5000 });  // Doesn't need reactivity
  const API_URL = $state('/api');           // Constants don't need $state
</script>

<!-- ✅ Declare constants directly -->
<script lang="ts">
  const config = { timeout: 5000 };
  const API_URL = '/api';

  // Use $state only for values that change
  let count = $state(0);
  let user = $state<User | null>(null);
</script>
```

### $state.raw and Large Objects

```svelte
<!-- ❌ Using deep reactivity for large immutable data -->
<script lang="ts">
  // largeData gets deeply proxied, which is expensive
  let data = $state(hugeApiResponse);
</script>

<!-- ✅ $state.raw avoids deep proxying -->
<script lang="ts">
  let data = $state.raw(hugeApiResponse);

  // Updates are triggered only when the whole value is replaced
  async function refresh() {
    data = await fetchLatestData();  // ✅ triggers reactivity
  }

  // ❌ Mutating nested properties won't trigger updates
  // data.items[0].name = 'new';  // will NOT re-render
</script>
```

### $state.snapshot for External Libraries

```svelte
<!-- ❌ Passing a $state object directly to an external library -->
<script lang="ts">
  let state = $state({ x: 0, y: 0 });

  onMount(() => {
    // External libraries may not handle Proxy objects correctly
    chartLibrary.update(state);  // state is a Proxy!
  });
</script>

<!-- ✅ $state.snapshot gets a plain object copy -->
<script lang="ts">
  let state = $state({ x: 0, y: 0 });

  onMount(() => {
    // $state.snapshot produces a plain object (Svelte 5)
    chartLibrary.update($state.snapshot(state));
  });
</script>
```

### Destructuring $state Loses Reactivity

```svelte
<!-- ❌ Destructuring a $state object loses reactivity -->
<script lang="ts">
  let state = $state({ count: 0, name: 'Svelte' });
  let { count, name } = state;  // count and name are plain values!
</script>
<p>{count}</p>  <!-- ❌ will NOT update when state.count changes -->

<!-- ✅ Access $state properties directly -->
<script lang="ts">
  let state = $state({ count: 0, name: 'Svelte' });
</script>
<p>{state.count}</p>  <!-- ✅ stays reactive -->

<!-- ✅ Or declare each $state separately -->
<script lang="ts">
  let count = $state(0);
  let name = $state('Svelte');
</script>
```

---

### Using $derived Correctly

```svelte
<!-- ❌ #1 anti-pattern: using $effect for state sync -->
<script lang="ts">
  let firstName = $state('John');
  let lastName = $state('Doe');
  let fullName = $state('');

  // Don't use $effect to sync derived state!
  $effect(() => {
    fullName = `${firstName} ${lastName}`;  // unnecessary effect
  });
</script>

<!-- ✅ Use $derived to compute derived values -->
<script lang="ts">
  let firstName = $state('John');
  let lastName = $state('Doe');
  let fullName = $derived(`${firstName} ${lastName}`);
</script>
```

### $derived Should Have No Side Effects

```svelte
<!-- ❌ Side effects inside $derived -->
<script lang="ts">
  let items = $state<Item[]>([]);
  let count = $derived(() => {
    console.log('recalculating');  // side effect!
    analytics.track('count', items.length);  // side effect!
    return items.length;
  });
</script>

<!-- ✅ Use $derived only for pure computations -->
<script lang="ts">
  let items = $state<Item[]>([]);
  let count = $derived(items.length);

  // side effects go in $effect
  $effect(() => {
    analytics.track('count', count);
  });
</script>
```

---

### Using $effect Correctly

#### $effect vs $derived

```svelte
<!-- ❌ Using $effect to sync state (the #1 anti-pattern) -->
<script lang="ts">
  let searchQuery = $state('');
  let results = $state([]);

  $effect(() => {
    results = searchQuery ? items.filter(i => i.name.includes(searchQuery)) : items;
  });
</script>

<!-- ✅ Use $derived -->
<script lang="ts">
  let searchQuery = $state('');
  let results = $derived(
    searchQuery ? items.filter(i => i.name.includes(searchQuery)) : items
  );
</script>
```

#### Infinite Loops

```svelte
<!-- ❌ Updating its own dependency inside $effect → infinite loop -->
<script lang="ts">
  let count = $state(0);

  $effect(() => {
    console.log(count);
    count++;  // modifying dependency inside effect → infinite loop!
  });
</script>

<!-- ✅ Avoid modifying tracked state inside $effect -->
<script lang="ts">
  let count = $state(0);
  let log = $state<string[]>([]);

  $effect(() => {
    // read count, write to a different state
    log = [...log, `count is ${count}`];
  });
</script>
```

#### Cleanup Functions

```svelte
<!-- ❌ Missing cleanup function → memory leak -->
<script lang="ts">
  let roomId = $state('');

  $effect(() => {
    const ws = new WebSocket(`ws://example.com/${roomId}`);
    ws.onmessage = (e) => {
      messages = [...messages, JSON.parse(e.data)];
    };
    // no cleanup! WebSocket leaks when roomId changes
  });
</script>

<!-- ✅ Return a cleanup function -->
<script lang="ts">
  let roomId = $state('');

  $effect(() => {
    const ws = new WebSocket(`ws://example.com/${roomId}`);
    ws.onmessage = (e) => {
      messages = [...messages, JSON.parse(e.data)];
    };
    return () => ws.close();  // cleanup on re-run
  });
</script>

<!-- ✅ Timer cleanup -->
<script lang="ts">
  $effect(() => {
    const id = setInterval(() => {
      console.log('tick');
    }, 1000);
    return () => clearInterval(id);
  });
</script>
```

#### Tracking Pitfalls in async $effect

```svelte
<!-- ❌ State read after await is not tracked -->
<script lang="ts">
  let userId = $state('1');
  let preference = $state('dark');

  $effect(async () => {
    const user = await fetchUser(userId);   // userId IS tracked
    const theme = preference;               // NOT tracked (read after await)!
    applyTheme(user, theme);
  });
</script>

<!-- ✅ Read all dependencies before await -->
<script lang="ts">
  let userId = $state('1');
  let preference = $state('dark');

  $effect(async () => {
    const currentPref = preference;  // read before await
    const user = await fetchUser(userId);
    applyTheme(user, currentPref);
  });
</script>
```

#### Excluding Dependencies with untrack

```svelte
<!-- ❌ Accidentally tracking an unnecessary dependency -->
<script lang="ts">
  let data = $state<Data | null>(null);
  let debugMode = $state(false);

  $effect(() => {
    if (debugMode) {  // debugMode becomes a dependency!
      console.log('data changed', data);
    }
  });
</script>

<!-- ✅ Use untrack to exclude irrelevant dependencies -->
<script lang="ts">
  import { untrack } from 'svelte';

  let data = $state<Data | null>(null);
  let debugMode = $state(false);

  $effect(() => {
    if (untrack(() => debugMode)) {  // debugMode is NOT tracked
      console.log('data changed', data);
    }
  });
</script>
```

---

## Load Functions (Server vs Client)

### +page.server.js vs +page.js

```typescript
// ❌ Accessing the database or secrets in +page.js
// src/routes/admin/+page.js
export async function load({ fetch }) {
  // universal load runs on both server and client
  const data = await db.query('SELECT * FROM users');  // db not available in browser!
  return { users: data };
}

// ✅ Put server-side logic in +page.server.js
// src/routes/admin/+page.server.js
import { db } from '$lib/server/db';

export async function load() {
  const users = await db.query('SELECT * FROM users');
  return { users };
}
```

```typescript
// ✅ Use +page.js for data also available on the client (e.g. aggregating fetches)
// src/routes/dashboard/+page.js
export async function load({ fetch, parent }) {
  const [analytics, notifications] = await Promise.all([
    fetch('/api/analytics').then(r => r.json()),
    fetch('/api/notifications').then(r => r.json())
  ]);
  return { analytics, notifications };
}
```

### await parent() Waterfalls

```typescript
// ❌ Sequentially awaiting parent → waterfall
// src/routes/blog/[slug]/+page.js
export async function load({ parent, fetch }) {
  const parentData = await parent();  // wait for parent
  const post = await fetch(`/api/posts/${parentData.blogId}`);
  return { post };
}

// ✅ Parallelize where possible and avoid unnecessary parent awaits
// src/routes/blog/[slug]/+page.js
export async function load({ parent, fetch }) {
  // only await parent if you truly need its data
  const post = await fetch('/api/posts/slug');
  return { post };
}

// ✅ If you truly need parent data, the waterfall is unavoidable, but comment it explicitly
// src/routes/blog/[slug]/+page.js
export async function load({ parent, fetch }) {
  const { blogId } = await parent();  // required: need blogId for post URL
  const post = await fetch(`/api/posts/${blogId}`);
  return { post };
}
```

### Non-Serializable Return Values

```typescript
// ❌ Returning non-serializable values from a server load
// src/routes/api/+page.server.js
export async function load() {
  return {
    stream: fs.createReadStream('data.csv'),  // not serializable!
    callback: () => console.log('hi'),        // functions not serializable!
    date: new Date(),                         // OK — devalue serializes Date/Map/Set fine
  };
}

// ✅ Return only serializable data
// src/routes/api/+page.server.js
export async function load() {
  return {
    data: await readFile('data.csv', 'utf-8'),
    timestamp: Date.now(),
  };
}
```

---

## Form Actions

### Use POST for Side Effects

```svelte
<!-- ❌ Handling side effects with GET/load functions -->
<script lang="ts">
  import { goto } from '$app/navigation';

  async function deleteUser(id: string) {
    await fetch(`/api/users/${id}`, { method: 'DELETE' });
    goto('/users');  // side effect via client navigation
  }
</script>
<button onclick={() => deleteUser(user.id)}>Delete</button>

<!-- ✅ Use form actions -->
```

```typescript
// src/routes/users/+page.server.js
import { fail, redirect } from '@sveltejs/kit';

export const actions = {
  delete: async ({ request, locals }) => {
    const formData = await request.formData();
    const id = formData.get('id');

    if (!id) return fail(400, { message: 'Missing id' });

    await locals.db.users.delete(id);
    throw redirect(303, '/users');
  }
};
```

```svelte
<!-- form with progressive enhancement -->
<script lang="ts">
  import { enhance } from '$app/forms';
</script>

<form method="POST" action="?/delete" use:enhance>
  <input type="hidden" name="id" value={user.id} />
  <button type="submit">Delete</button>
</form>
```

### Don't Expose Sensitive Information in fail()

```typescript
// ❌ Returning sensitive information in fail()
// src/routes/login/+page.server.js
export const actions = {
  default: async ({ request, locals }) => {
    const formData = await request.formData();
    const user = await locals.db.users.findByEmail(formData.get('email'));

    return fail(401, {
      password: formData.get('password'),  // ❌ exposes password in page data!
      hint: user.passwordHint,             // ❌ leaks internal data!
    });
  }
};

// ✅ Return only safe error information
export const actions = {
  default: async ({ request }) => {
    const formData = await request.formData();
    const email = formData.get('email');

    return fail(401, {
      email,                    // ✅ safe to echo back
      incorrect: true,          // ✅ generic error flag
    });
  }
};
```

### Progressive Enhancement with use:enhance

```svelte
<!-- ❌ Form without use:enhance → only native behavior (full page reload) -->
<form method="POST" action="?/create">
  <input name="title" />
  <button type="submit">Create</button>
</form>

<!-- ✅ use:enhance provides an SPA experience + progressive enhancement -->
<script lang="ts">
  import { enhance } from '$app/forms';
</script>

<form method="POST" action="?/create" use:enhance={() => {
  return ({ update }) => {
    update({ reset: false });  // customize behavior
  };
}}>
  <input name="title" />
  <button type="submit">Create</button>
</form>

<!-- ✅ With a loading state -->
<form
  method="POST"
  action="?/create"
  use:enhance={() => {
    submitting = true;
    return ({ update }) => {
      update();
      submitting = false;
    };
  }}
>
  <button type="submit" disabled={submitting}>
    {submitting ? 'Creating...' : 'Create'}
  </button>
</form>
```

---

## Store Migration (→ $state)

### writable/readable → $state

```typescript
// ❌ Legacy store pattern (Svelte 4)
// src/lib/stores/user.js
import { writable, derived } from 'svelte/store';

export const user = writable(null);
export const isLoggedIn = derived(user, $user => !!$user);

// usage with $ prefix
// $user = { name: 'John' };

// ✅ Svelte 5: shared state in .svelte.js files
// src/lib/stores/user.svelte.js
let currentUser = $state<User | null>(null);

export function getUser() {
  return currentUser;
}

export function setUser(user: User | null) {
  currentUser = user;
}

export function isLoggedIn() {
  return currentUser !== null;
}
```

### The $ Prefix Store Syntax Is Legacy

```svelte
<!-- ❌ $-prefixed store auto-subscription is a legacy pattern -->
<script lang="ts">
  import { count } from '$lib/stores/count';
  // $count is legacy syntax in Svelte 5
</script>
<p>{$count}</p>

<!-- ✅ Svelte 5 runes pattern -->
<script lang="ts">
  import { getCount } from '$lib/stores/count.svelte';

  let count = $derived(getCount());
</script>
<p>{count}</p>

<!-- ✅ Or use an exported reactive $state getter directly -->
<script lang="ts">
  // count.svelte.js exports a reactive reference
  import { counter } from '$lib/stores/count.svelte';
</script>
<p>{counter.value}</p>
```

### .svelte.js / .svelte.ts Extensions

```typescript
// ❌ Using runes in a plain .js file → compile error
// src/lib/utils.js
let state = $state(0);  // runes only work in .svelte.js files!

// ✅ Use the .svelte.js extension
// src/lib/utils.svelte.js
let state = $state(0);

export function getState() {
  return state;
}

export function setState(val: number) {
  state = val;
}
```

---

## SSR vs CSR Boundaries

### ssr=false SPA Mode

```typescript
// ❌ Disabling SSR in the root layout → everything becomes CSR
// src/routes/+layout.js
export const ssr = false;  // entire app becomes SPA

// ✅ Disable SSR only on pages that need it
// src/routes/admin/dashboard/+page.js
export const ssr = false;  // only this page skips SSR

// ✅ Better: configure per route
// src/routes/editor/+page.js
export const ssr = false;  // editor needs browser APIs, skip SSR
```

### Browser Globals During SSR

```svelte
<!-- ❌ Accessing browser APIs at the module top level -->
<script lang="ts">
  const height = window.innerHeight;        // ReferenceError during SSR!
  const prefersDark = matchMedia('(prefers-color-scheme: dark)');  // crash!
</script>

<!-- ✅ Access them in onMount or behind a browser guard -->
<script lang="ts">
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';

  let height = $state(0);

  onMount(() => {
    height = window.innerHeight;
  });

  // or conditional check
  const prefersDark = browser
    ? matchMedia('(prefers-color-scheme: dark)').matches
    : false;
</script>
```

### prerender Conflicts with actions

```typescript
// ❌ Defining actions on a prerendered page → compile error
// src/routes/contact/+page.server.js
export const prerender = true;

export const actions = {
  // Error: prerendered pages cannot have server-side form actions
  default: async ({ request }) => { /* ... */ }
};

// ✅ Prerendered pages don't use server actions
// src/routes/about/+page.server.js
export const prerender = true;
// no actions — static page

// ✅ Pages that need actions are not prerendered
// src/routes/contact/+page.server.js
export const actions = {
  default: async ({ request }) => {
    // handle form submission
  }
};
```

---

## Reactive Statement Migration

### $: → $derived / $effect

```svelte
<!-- ❌ Svelte 4 reactive statements -->
<script lang="ts">
  let count = 0;
  let doubled = 0;

  $: doubled = count * 2;              // reactive assignment
  $: if (count > 10) console.log('big');
</script>

<!-- ✅ Svelte 5 runes -->
<script lang="ts">
  let count = $state(0);
  let doubled = $derived(count * 2);   // derived value

  $effect(() => {
    if (count > 10) console.log('big');
  });
</script>
```

### export let → $props()

```svelte
<!-- ❌ Svelte 4 props -->
<script lang="ts">
  export let title: string;
  export let count = 0;
</script>

<!-- ✅ Svelte 5 $props() -->
<script lang="ts">
  let { title, count = 0 }: { title: string; count?: number } = $props();
</script>
```

### on:click → onclick

```svelte
<!-- ❌ Svelte 4 directive-style events -->
<button on:click={handleClick}>Click</button>
<button on:click={() => count++}>Increment</button>

<!-- ✅ Svelte 5 HTML attribute-style events -->
<button onclick={handleClick}>Click</button>
<button onclick={() => count++}>Increment</button>
```

### createEventDispatcher → Callback Props

```svelte
<!-- ❌ Svelte 4 event dispatch -->
<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  function handleDelete() {
    dispatch('delete', { id: 42 });
  }
</script>

<!-- ✅ Svelte 5 callback props -->
<script lang="ts">
  let { ondelete }: { ondelete?: (e: { id: number }) => void } = $props();

  function handleDelete() {
    ondelete?.({ id: 42 });
  }
</script>

<!-- parent usage -->
<Child ondelete={(e) => removeItem(e.id)} />
```

### slot → @render children()

```svelte
<!-- ❌ Svelte 4 slot -->
<!-- Card.svelte -->
<div class="card">
  <slot />
</div>

<!-- ✅ Svelte 5 snippets -->
<!-- Card.svelte -->
<script lang="ts">
  let { children } = $props();
</script>
<div class="card">
  {@render children()}
</div>

<!-- with named slots → named snippets -->
<!-- Layout.svelte -->
<script lang="ts">
  let { header, children, footer } = $props();
</script>
<div>
  <header>{@render header?.()}</header>
  <main>{@render children()}</main>
  <footer>{@render footer?.()}</footer>
</div>

<!-- parent usage -->
<Layout>
  {#snippet header()}<h1>Title</h1>{/snippet}
  <p>Body content</p>
  {#snippet footer()}<p>Footer</p>{/snippet}
</Layout>
```

### beforeUpdate / afterUpdate → $effect.pre

```svelte
<!-- ❌ Svelte 4 lifecycle hooks -->
<script lang="ts">
  import { beforeUpdate, afterUpdate } from 'svelte';

  let count = 0;

  beforeUpdate(() => {
    console.log('about to update', count);
  });

  afterUpdate(() => {
    console.log('updated', count);
    document.title = `Count: ${count}`;
  });
</script>

<!-- ✅ Svelte 5 $effect and $effect.pre -->
<script lang="ts">
  let count = $state(0);

  // $effect.pre runs before DOM updates (like beforeUpdate)
  $effect.pre(() => {
    console.log('about to update', count);
  });

  // $effect runs after DOM updates (like afterUpdate)
  $effect(() => {
    console.log('updated', count);
    document.title = `Count: ${count}`;
  });
</script>
```

---

## Performance Optimization

### $state.raw for Large Immutable Data

```svelte
<!-- ❌ Deeply proxying large immutable data -->
<script lang="ts">
  let searchResults = $state(largeResultArray);  // deep proxy on every item
</script>

<!-- ✅ $state.raw avoids deep proxying -->
<script lang="ts">
  let searchResults = $state.raw<SearchResult[]>([]);

  async function search(query: string) {
    searchResults = await fetchResults(query);  // whole-array replacement
  }
</script>
```

### Keyed {#each}

```svelte
<!-- ❌ each without a key → inefficient DOM diffing -->
{#each items as item}
  <div>{item.name}</div>
{/each}

<!-- ✅ each with a unique key -->
{#each items as item (item.id)}
  <div>{item.name}</div>
{/each}

<!-- ✅ Composite key -->
{#each items as item (`${item.category}-${item.id}`)}
  <div>{item.name}</div>
{/each}
```

### Streaming and Promises in load

```typescript
// ❌ Awaiting all data serially → page blocks
// src/routes/+page.server.js
export async function load({ params }) {
  const posts = await getPosts();       // slow
  const comments = await getComments(); // slow
  const tags = await getTags();         // slow
  return { posts, comments, tags };
}

// ✅ Load independent data in parallel
export async function load({ params }) {
  return {
    posts: getPosts(),       // return promises directly for streaming
    comments: getComments(),
    tags: getTags(),
  };
}
```

```svelte
<!-- streaming in template with {#await} -->
{#await data.posts}
  <p>Loading posts...</p>
{:then posts}
  <ul>
    {#each posts as post (post.id)}
      <li>{post.title}</li>
    {/each}
  </ul>
{:catch error}
  <p>Failed to load posts: {error.message}</p>
{/await}
```

---

## Security Review

Svelte/SvelteKit escapes template expressions automatically by default. Review focus: use of `{@html}`, `$env/static/private` leaks, and built-in CSRF protection.

> **For cross-framework XSS prevention, see the [XSS Prevention Guide](cross-cutting/xss-prevention.md)**, which includes React/Vue/Angular/Svelte examples and CSP configuration.

### Don't Expose Private Environment Variables

```typescript
// ❌ Exposing server-side secrets in a universal load
// src/routes/admin/+page.js (universal — runs on client too!)
export async function load() {
  return {
    apiKey: process.env.SECRET_API_KEY,    // exposed to client bundle!
    dbUrl: import.meta.env.DATABASE_URL,    // leaks to browser!
  };
}

// ✅ Use private environment variables only in server loads
// src/routes/admin/+page.server.js (server-only)
export async function load({ locals }) {
  // secrets stay on server
  const data = await fetch(process.env.SECRET_API_KEY + '/admin');
  return { data };  // only derived data is sent to client
}

// ✅ Use the PUBLIC_ prefix for public variables
// .env
// PUBLIC_API_URL=https://api.example.com
// SECRET_API_KEY=xxx  (no PUBLIC_ prefix = server-only)
```

### Server Isolation with $lib/server/

```typescript
// ❌ Server code placed where the client can import it
// src/lib/db.js
import { SECRET_DB_URL } from '$env/static/private';
// any client component importing this gets the secret!

// ✅ Put it in the $lib/server/ directory → client imports fail at compile time
// src/lib/server/db.js
import { SECRET_DB_URL } from '$env/static/private';

export async function query(sql: string) {
  // safe: client cannot import from $lib/server/
}

// usage in server files only
// src/routes/api/users/+server.js
import { query } from '$lib/server/db';
```

### Built-in CSRF Protection

```typescript
// ✅ SvelteKit has built-in CSRF protection
// Origin header is checked automatically for POST/PUT/DELETE/PATCH
// No additional CSRF tokens needed for form actions

// ❌ Don't disable CSRF checks (unless you have a very good reason)
// src/hooks.server.js
export const handle = sequence(
  // do NOT do this without understanding the implications
  // ({ event, resolve }) => resolve(event, { filterSerializedResponseHeaders: () => true })
);
```

### Secure Cookie Settings

```typescript
// ❌ Insecure cookie settings
// src/hooks.server.js
export async function handle({ event, resolve }) {
  const token = event.cookies.get('session');
  // cookie without httpOnly, secure, sameSite flags
  event.cookies.set('session', token, {
    path: '/',
    // missing: httpOnly, secure, sameSite
  });
}

// ✅ Secure cookie configuration
import { dev } from '$app/environment';

event.cookies.set('session', token, {
  path: '/',
  httpOnly: true,          // not accessible via JS
  secure: !dev,            // HTTPS only in production
  sameSite: 'lax',         // CSRF protection
  maxAge: 60 * 60 * 24 * 7 // 1 week, explicit expiry
});
```

---

## Review Checklist

### Runes: $state / $derived / $effect

- [ ] $state is used only for values that change; constants are declared directly
- [ ] Large immutable data uses $state.raw
- [ ] $state objects are not destructured (loses reactivity)
- [ ] $state.snapshot is used to pass plain objects to external libraries
- [ ] No side effects in $derived
- [ ] $effect is not used in place of $derived for state sync
- [ ] $effect does not modify tracked state (avoids infinite loops)
- [ ] $effect has cleanup functions (subscriptions, timers, WebSocket)
- [ ] async $effect reads all state that needs tracking before await
- [ ] untrack is used to exclude irrelevant dependencies

### Load Functions

- [ ] Server-side logic lives in +page.server.js (not +page.js)
- [ ] Unnecessary await parent() waterfalls are avoided
- [ ] Independent data is loaded in parallel (Promise.all or returning Promises directly)
- [ ] server load returns only serializable data

### Form Actions

- [ ] Side-effecting operations (create/update/delete) use form actions + POST
- [ ] fail() does not return sensitive information (passwords, internal data)
- [ ] use:enhance is used for progressive enhancement

### Store Migration

- [ ] writable/readable → $state in .svelte.js files
- [ ] Runes are not used in plain .js files
- [ ] The legacy $-prefixed store syntax is not used

### SSR vs CSR Boundaries

- [ ] SSR is not disabled globally in the root layout
- [ ] Browser APIs (window, document) are used in onMount or behind a browser guard
- [ ] Prerendered pages contain no server actions

### Svelte 4 → 5 Migration

- [ ] $: → $derived / $effect
- [ ] export let → $props()
- [ ] on:click → onclick
- [ ] createEventDispatcher → callback props
- [ ] slot → @render children()
- [ ] beforeUpdate/afterUpdate → $effect.pre / $effect

### Performance Optimization

- [ ] Large immutable data uses $state.raw
- [ ] {#each} uses unique keys
- [ ] load functions return Promises for streaming
- [ ] Independent data is loaded in parallel

### Security Review

- [ ] Private environment variables are used only in server loads
- [ ] Server code lives in the $lib/server/ directory
- [ ] Built-in CSRF protection is not disabled
- [ ] Cookies set httpOnly, secure, and sameSite
- [ ] server load does not leak secrets or internal data
