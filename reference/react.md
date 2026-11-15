# React Code Review Guide

React review focus: Hooks rules, appropriate (not excessive) performance optimization, component design, and modern React 19/RSC patterns.

## Table of Contents

- [Basic Hooks Rules](#basic-hooks-rules)
- [useEffect Patterns](#useeffect-patterns)
- [useMemo / useCallback](#usememo--usecallback)
- [Component Design](#component-design)
- [Error Boundaries & Suspense](#error-boundaries--suspense)
- [Server Components (RSC)](#server-components-rsc)
- [React 19 Actions & Forms](#react-19-actions--forms)
- [Suspense & Streaming SSR](#suspense--streaming-ssr)
- [TanStack Query v5](#tanstack-query-v5)
- [Review Checklists](#review-checklists)

---

## Basic Hooks Rules

```tsx
// ❌ Conditionally calling Hooks — violates the Rules of Hooks
function BadComponent({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null);  // Error!
  }
  return <div>...</div>;
}

// ✅ Hooks must be called at the top level of the component
function GoodComponent({ isLoggedIn }) {
  const [user, setUser] = useState(null);
  if (!isLoggedIn) return <LoginPrompt />;
  return <div>{user?.name}</div>;
}
```

---

## useEffect Patterns

```tsx
// ❌ Missing or incomplete dependency array
function BadEffect({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, []);  // Missing userId dependency!
}

// ✅ Complete dependency array
function GoodEffect({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    let cancelled = false;
    fetchUser(userId).then(data => {
      if (!cancelled) setUser(data);
    });
    return () => { cancelled = true; };  // cleanup function
  }, [userId]);
}

// ❌ useEffect used for derived state (anti-pattern)
function BadDerived({ items }) {
  const [filteredItems, setFilteredItems] = useState([]);
  useEffect(() => {
    setFilteredItems(items.filter(i => i.active));
  }, [items]);  // Unnecessary effect + extra render
  return <List items={filteredItems} />;
}

// ✅ Compute directly during render, or use useMemo
function GoodDerived({ items }) {
  const filteredItems = useMemo(
    () => items.filter(i => i.active),
    [items]
  );
  return <List items={filteredItems} />;
}

// ❌ useEffect used to respond to events
function BadEventEffect() {
  const [query, setQuery] = useState('');
  useEffect(() => {
    if (query) {
      analytics.track('search', { query });  // Should be in the event handler
    }
  }, [query]);
}

// ✅ Perform side effects in the event handler
function GoodEvent() {
  const [query, setQuery] = useState('');
  const handleSearch = (q: string) => {
    setQuery(q);
    analytics.track('search', { query: q });
  };
}
```

---

## useMemo / useCallback

```tsx
// ❌ Over-optimization — constants don't need useMemo
function OverOptimized() {
  const config = useMemo(() => ({ timeout: 5000 }), []);  // Pointless
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);  // Pointless unless passed to a memoized component
}

// ✅ Optimize only when needed
function ProperlyOptimized() {
  const config = { timeout: 5000 };  // Define simple objects directly
  const handleClick = () => console.log('clicked');
}

// ❌ useCallback dependency always changes
function BadCallback({ data }) {
  // data is a new object on every render, so useCallback is ineffective
  const process = useCallback(() => {
    return data.map(transform);
  }, [data]);
}

// ✅ Use useMemo + useCallback together with React.memo
const MemoizedChild = React.memo(function Child({ onClick, items }) {
  return <div onClick={onClick}>{items.length}</div>;
});

function Parent({ rawItems }) {
  const items = useMemo(() => processItems(rawItems), [rawItems]);
  const handleClick = useCallback(() => {
    console.log(items.length);
  }, [items]);
  return <MemoizedChild onClick={handleClick} items={items} />;
}
```

---

## Component Design

```tsx
// ❌ Defining a component inside a component — creates a new component on every render
function BadParent() {
  function ChildComponent() {  // A new function on every render!
    return <div>child</div>;
  }
  return <ChildComponent />;
}

// ✅ Define components outside
function ChildComponent() {
  return <div>child</div>;
}
function GoodParent() {
  return <ChildComponent />;
}

// ❌ Props are always new object references
function BadProps() {
  return (
    <MemoizedComponent
      style={{ color: 'red' }}  // New object on every render
      onClick={() => {}}         // New function on every render
    />
  );
}

// ✅ Stable references
const style = { color: 'red' };
function GoodProps() {
  const handleClick = useCallback(() => {}, []);
  return <MemoizedComponent style={style} onClick={handleClick} />;
}
```

---

## Error Boundaries & Suspense

```tsx
// ❌ No error boundary
function BadApp() {
  return (
    <Suspense fallback={<Loading />}>
      <DataComponent />  {/* An error will crash the entire app */}
    </Suspense>
  );
}

// ✅ Error Boundary wrapping Suspense
function GoodApp() {
  return (
    <ErrorBoundary fallback={<ErrorUI />}>
      <Suspense fallback={<Loading />}>
        <DataComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## Server Components (RSC)

```tsx
// ❌ Using client-side features in a Server Component
// app/page.tsx (Server Component by default)
function BadServerComponent() {
  const [count, setCount] = useState(0);  // Error! No hooks in RSC
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// ✅ Extract interactive logic into a Client Component
// app/counter.tsx
'use client';
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// app/page.tsx (Server Component)
async function GoodServerComponent() {
  const data = await fetchData();  // Can await directly
  return (
    <div>
      <h1>{data.title}</h1>
      <Counter />  {/* Client component */}
    </div>
  );
}

// ❌ Misplaced 'use client' — the whole tree becomes client-side
// layout.tsx
'use client';  // This makes every child component a client component
export default function Layout({ children }) { ... }

// ✅ Use 'use client' only in components that need interactivity
// Isolate client logic into leaf components
```

---

## React 19 Actions & Forms

React 19 introduces the Actions system and new form-handling Hooks that simplify async operations and optimistic updates.

### useActionState

```tsx
// ❌ Traditional approach: multiple state variables
function OldForm() {
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [data, setData] = useState(null);

  const handleSubmit = async (formData: FormData) => {
    setIsPending(true);
    setError(null);
    try {
      const result = await submitForm(formData);
      setData(result);
    } catch (e) {
      setError(e.message);
    } finally {
      setIsPending(false);
    }
  };
}

// ✅ React 19: useActionState manages it all in one place
import { useActionState } from 'react';

function NewForm() {
  const [state, formAction, isPending] = useActionState(
    async (prevState, formData: FormData) => {
      try {
        const result = await submitForm(formData);
        return { success: true, data: result };
      } catch (e) {
        return { success: false, error: e.message };
      }
    },
    { success: false, data: null, error: null }
  );

  return (
    <form action={formAction}>
      <input name="email" />
      <button disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
      {state.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

### useFormStatus

```tsx
// ❌ Passing form state down through props
function BadSubmitButton({ isSubmitting }) {
  return <button disabled={isSubmitting}>Submit</button>;
}

// ✅ useFormStatus reads the parent <form> state (no props needed)
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  // Note: must be used in a child component inside the <form>
  return (
    <button disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

// ❌ useFormStatus called in a component at the same level as the form — doesn't work
function BadForm() {
  const { pending } = useFormStatus();  // Can't get the status here!
  return (
    <form action={action}>
      <button disabled={pending}>Submit</button>
    </form>
  );
}

// ✅ useFormStatus must be in a child component of the form
function GoodForm() {
  return (
    <form action={action}>
      <SubmitButton />  {/* useFormStatus is called in here */}
    </form>
  );
}
```

### useOptimistic

```tsx
// ❌ Waiting for the server response before updating the UI
function SlowLike({ postId, likes }) {
  const [likeCount, setLikeCount] = useState(likes);
  const [isPending, setIsPending] = useState(false);

  const handleLike = async () => {
    setIsPending(true);
    const newCount = await likePost(postId);  // waiting...
    setLikeCount(newCount);
    setIsPending(false);
  };
}

// ✅ useOptimistic gives instant feedback and rolls back automatically on failure
import { useOptimistic } from 'react';

function FastLike({ postId, likes }) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    likes,
    (currentLikes, increment: number) => currentLikes + increment
  );

  const handleLike = async () => {
    addOptimisticLike(1);  // Update the UI immediately
    try {
      await likePost(postId);  // Sync in the background
    } catch {
      // React automatically rolls back to the original likes value
    }
  };

  return <button onClick={handleLike}>{optimisticLikes} likes</button>;
}
```

### Server Actions (Next.js 15+)

```tsx
// ❌ Calling an API from the client
'use client';
function ClientForm() {
  const handleSubmit = async (formData: FormData) => {
    const res = await fetch('/api/submit', {
      method: 'POST',
      body: formData,
    });
    // ...
  };
}

// ✅ Server Action + useActionState
// actions.ts
'use server';
export async function createPost(prevState: any, formData: FormData) {
  const title = formData.get('title');
  await db.posts.create({ title });
  revalidatePath('/posts');
  return { success: true };
}

// form.tsx
'use client';
import { createPost } from './actions';

function PostForm() {
  const [state, formAction, isPending] = useActionState(createPost, null);
  return (
    <form action={formAction}>
      <input name="title" />
      <SubmitButton />
    </form>
  );
}
```

---

## Suspense & Streaming SSR

Suspense and Streaming are core features of React 18+, widely used in 2025 frameworks such as Next.js 15.

### Basic Suspense

```tsx
// ❌ Traditional loading-state management
function OldComponent() {
  const [data, setData] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetchData().then(setData).finally(() => setIsLoading(false));
  }, []);

  if (isLoading) return <Spinner />;
  return <DataView data={data} />;
}

// ✅ Suspense for declarative loading states
function NewComponent() {
  return (
    <Suspense fallback={<Spinner />}>
      <DataView />  {/* Internally uses use() or Suspense-enabled data fetching */}
    </Suspense>
  );
}
```

### Multiple Independent Suspense Boundaries

```tsx
// ❌ Single boundary — everything loads together
function BadLayout() {
  return (
    <Suspense fallback={<FullPageSpinner />}>
      <Header />
      <MainContent />  {/* slow */}
      <Sidebar />      {/* fast */}
    </Suspense>
  );
}

// ✅ Independent boundaries — each part streams independently
function GoodLayout() {
  return (
    <>
      <Header />  {/* Shown immediately */}
      <div className="flex">
        <Suspense fallback={<ContentSkeleton />}>
          <MainContent />  {/* Loads independently */}
        </Suspense>
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />      {/* Loads independently */}
        </Suspense>
      </div>
    </>
  );
}
```

### Next.js 15 Streaming

```tsx
// app/page.tsx - automatic Streaming
export default async function Page() {
  // This await does not block the entire page
  const data = await fetchSlowData();
  return <div>{data}</div>;
}

// app/loading.tsx - automatic Suspense boundary
export default function Loading() {
  return <Skeleton />;
}
```

### use() Hook (React 19)

```tsx
// ✅ Reading a Promise in a component
import { use } from 'react';

function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);  // Triggers Suspense automatically
  return (
    <ul>
      {comments.map(c => <li key={c.id}>{c.text}</li>)}
    </ul>
  );
}

// Parent component creates the Promise, child component consumes it
function Post({ postId }) {
  const commentsPromise = fetchComments(postId);  // No await
  return (
    <article>
      <PostContent id={postId} />
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments commentsPromise={commentsPromise} />
      </Suspense>
    </article>
  );
}
```

---

## TanStack Query v5

TanStack Query is the most popular data-fetching library in the React ecosystem; v5 is the current stable version.

### Basic Configuration

```tsx
// ❌ Inappropriate default configuration
const queryClient = new QueryClient();  // Defaults may not be suitable

// ✅ Recommended production configuration
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // Data is considered fresh for 5 minutes
      gcTime: 1000 * 60 * 30,    // Garbage-collected after 30 minutes (renamed in v5)
      retry: 3,
      refetchOnWindowFocus: false,  // Decide based on requirements
    },
  },
});
```

### queryOptions (new in v5)

```tsx
// ❌ Duplicated queryKey and queryFn definitions
function Component1() {
  const { data } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  });
}

function prefetchUser(queryClient, userId) {
  queryClient.prefetchQuery({
    queryKey: ['users', userId],  // Duplicated!
    queryFn: () => fetchUser(userId),  // Duplicated!
  });
}

// ✅ queryOptions defines it once, type-safe
import { queryOptions } from '@tanstack/react-query';

const userQueryOptions = (userId: string) =>
  queryOptions({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  });

function Component1({ userId }) {
  const { data } = useQuery(userQueryOptions(userId));
}

function prefetchUser(queryClient, userId) {
  queryClient.prefetchQuery(userQueryOptions(userId));
}

// getQueryData is type-safe too
const user = queryClient.getQueryData(userQueryOptions(userId).queryKey);
```

### Common Pitfalls

```tsx
// ❌ staleTime of 0 causes excessive requests
useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  // staleTime defaults to 0, so it refetches on every component mount
});

// ✅ Set a reasonable staleTime
useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  staleTime: 1000 * 60,  // No refetch within 1 minute
});

// ❌ Using unstable references in queryFn
function BadQuery({ filters }) {
  useQuery({
    queryKey: ['items'],  // queryKey doesn't include filters!
    queryFn: () => fetchItems(filters),  // Changes to filters won't trigger a refetch
  });
}

// ✅ queryKey includes every parameter that affects the data
function GoodQuery({ filters }) {
  useQuery({
    queryKey: ['items', filters],  // filters is part of the queryKey
    queryFn: () => fetchItems(filters),
  });
}
```

### useSuspenseQuery

> **Important limitation**: useSuspenseQuery differs significantly from useQuery; understand its limitations before choosing it.

#### Limitations of useSuspenseQuery

| Feature | useQuery | useSuspenseQuery |
|---------|----------|------------------|
| `enabled` option | ✅ Supported | ❌ Not supported |
| `placeholderData` | ✅ Supported | ❌ Not supported |
| `data` type | `T \| undefined` | `T` (guaranteed to have a value) |
| Error handling | `error` property | Thrown to Error Boundary |
| Loading state | `isLoading` property | Suspends to Suspense |

#### Alternatives to the Unsupported `enabled` Option

```tsx
// ❌ Using useQuery + enabled for conditional queries
function BadSuspenseQuery({ userId }) {
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    enabled: !!userId,  // useSuspenseQuery doesn't support enabled!
  });
}

// ✅ Use component composition for conditional rendering
function GoodSuspenseQuery({ userId }) {
  // useSuspenseQuery guarantees data is T, not T | undefined
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  return <UserProfile user={data} />;
}

function Parent({ userId }) {
  if (!userId) return <NoUserSelected />;
  return (
    <Suspense fallback={<UserSkeleton />}>
      <GoodSuspenseQuery userId={userId} />
    </Suspense>
  );
}
```

#### Error Handling Differences

```tsx
// ❌ useSuspenseQuery has no error property
function BadErrorHandling() {
  const { data, error } = useSuspenseQuery({...});
  if (error) return <Error />;  // error is always null!
}

// ✅ Use an Error Boundary to handle errors
function GoodErrorHandling() {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<Loading />}>
        <DataComponent />
      </Suspense>
    </ErrorBoundary>
  );
}

function DataComponent() {
  // Errors are thrown to the Error Boundary
  const { data } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: fetchData,
  });
  return <Display data={data} />;
}
```

#### When to Choose useSuspenseQuery

```tsx
// ✅ Good fit:
// 1. The data is always needed (unconditional query)
// 2. The component must have data to render
// 3. Using React 19 Suspense patterns
// 4. Server components + client hydration

// ❌ Poor fit:
// 1. Conditional queries (triggered by user actions)
// 2. placeholderData or initial data is needed
// 3. loading/error states need to be handled inside the component
// 4. Multiple queries depend on each other

// ✅ Use useSuspenseQueries for multiple independent queries
function MultipleQueries({ userId }) {
  const [userQuery, postsQuery] = useSuspenseQueries({
    queries: [
      { queryKey: ['user', userId], queryFn: () => fetchUser(userId) },
      { queryKey: ['posts', userId], queryFn: () => fetchPosts(userId) },
    ],
  });
  // Both queries run in parallel; the component renders once both complete
  return <Profile user={userQuery.data} posts={postsQuery.data} />;
}
```

### Optimistic Updates (simplified in v5)

```tsx
// ❌ Optimistic updates with manual cache management (complex)
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });
    const previousTodos = queryClient.getQueryData(['todos']);
    queryClient.setQueryData(['todos'], (old) => [...old, newTodo]);
    return { previousTodos };
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context.previousTodos);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});

// ✅ v5 simplification: use variables for optimistic UI
function TodoList() {
  const { data: todos } = useQuery(todosQueryOptions);
  const { mutate, variables, isPending } = useMutation({
    mutationFn: addTodo,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return (
    <ul>
      {todos?.map(todo => <TodoItem key={todo.id} todo={todo} />)}
      {/* Optimistically show the todo being added */}
      {isPending && <TodoItem todo={variables} isOptimistic />}
    </ul>
  );
}
```

### v5 Status Field Changes

```tsx
// v4: isLoading means initial load or subsequent fetch
// v5: isPending means no data yet; isLoading = isPending && isFetching

const { data, isPending, isFetching, isLoading } = useQuery({...});

// isPending: no data in the cache (initial load)
// isFetching: request in flight (including background refresh)
// isLoading: isPending && isFetching (initial load in progress)

// ❌ Migrating v4 code as-is
if (isLoading) return <Spinner />;  // Behavior may differ in v5

// ✅ Make the intent explicit
if (isPending) return <Spinner />;  // Show loading when there is no data
// or
if (isLoading) return <Spinner />;  // Initial load in progress
```

---

## Review Checklists

### Hooks Rules

- [ ] Hooks are called at the top level of components/custom Hooks
- [ ] No Hooks called inside conditions/loops
- [ ] useEffect dependency arrays are complete
- [ ] useEffect has a cleanup function (subscriptions/timers/requests)
- [ ] useEffect is not used to compute derived state

### Performance Optimization (in moderation)

- [ ] useMemo/useCallback used only where truly needed
- [ ] React.memo paired with stable props references
- [ ] No child components defined inside components
- [ ] No new objects/functions created in JSX (unless passed to non-memoized components)
- [ ] Long lists use virtualization (react-window/react-virtual)

### Component Design

- [ ] Components have a single responsibility and stay under 200 lines
- [ ] Logic is separated from presentation (custom Hooks)
- [ ] Props interfaces are clear and typed with TypeScript
- [ ] Props drilling avoided (consider Context or composition)

### State Management

- [ ] State is kept close to where it's used (minimum necessary scope)
- [ ] useReducer for complex state
- [ ] Context or a state library for global state
- [ ] Unnecessary state avoided (derive > store)

### Error Handling

- [ ] Critical areas have Error Boundaries
- [ ] Suspense is used together with Error Boundaries
- [ ] Async operations have error handling

### Server Components (RSC)

- [ ] 'use client' used only for components that need interactivity
- [ ] Server Components don't use Hooks/event handlers
- [ ] Client components are placed at leaf nodes where possible
- [ ] Data fetching happens in Server Components

### React 19 Forms

- [ ] useActionState used instead of multiple useState calls
- [ ] useFormStatus called in a child component of the form
- [ ] useOptimistic not used for critical business operations (payments, etc.)
- [ ] Server Actions correctly marked with 'use server'

### Suspense & Streaming

- [ ] Suspense boundaries divided according to user-experience needs
- [ ] Each Suspense has a corresponding Error Boundary
- [ ] Meaningful fallbacks provided (skeleton screens > spinners)
- [ ] Avoid awaiting slow data at the layout level

### TanStack Query

- [ ] queryKey includes every parameter that affects the data
- [ ] A reasonable staleTime is set (not the default 0)
- [ ] useSuspenseQuery does not use enabled
- [ ] Related queries are invalidated after a successful mutation
- [ ] The difference between isPending and isLoading is understood

### Testing

- [ ] Use @testing-library/react
- [ ] Query elements via screen
- [ ] Use userEvent instead of fireEvent
- [ ] Prefer *ByRole queries
- [ ] Test behavior, not implementation details
