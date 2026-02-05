# TanStack Start Reference

TanStack Start is a full-stack React framework built on Vite, Nitro, and TanStack Router.

Official docs: https://tanstack.com/start/latest

## Project Structure

```
src/
├── routes/
│   ├── __root.tsx      # Root layout (always rendered)
│   ├── index.tsx       # Homepage route (/)
│   ├── about.tsx       # /about route
│   └── users/
│       ├── index.tsx   # /users route
│       └── $id.tsx     # /users/:id dynamic route
├── components/
├── client.tsx
├── router.tsx
└── routeTree.gen.ts    # Auto-generated route tree
```

## File-Based Routing

Routes are defined in `src/routes/` using file naming conventions:

| File | Route |
|------|-------|
| `index.tsx` | `/` |
| `about.tsx` | `/about` |
| `users/index.tsx` | `/users` |
| `users/$id.tsx` | `/users/:id` (dynamic) |
| `__root.tsx` | Root layout wrapper |

---

## createFileRoute API

### Basic Route

```typescript
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/about")({
  component: AboutPage,
});

function AboutPage() {
  return <div>About</div>;
}
```

### Route with Loader

```typescript
export const Route = createFileRoute("/users")({
  component: UsersPage,
  loader: async () => {
    const users = await fetchUsers();
    return { users };
  },
});

function UsersPage() {
  const { users } = Route.useLoaderData();
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### All Route Options

```typescript
export const Route = createFileRoute("/path")({
  // Required
  component: MyComponent,

  // Data loading
  loader: async ({ params, context, abortController }) => { ... },
  beforeLoad: async ({ params, context }) => { ... },
  loaderDeps: ({ search }) => ({ query: search.q }),

  // Loading states
  pendingComponent: () => <div>Loading...</div>,
  pendingMs: 150,        // ms before showing pendingComponent
  pendingMinMs: 200,     // minimum time to show pendingComponent

  // Error handling
  errorComponent: ({ error }) => <div>Error: {error.message}</div>,

  // Caching
  staleTime: 1000 * 60,  // 1 min - data considered fresh
  gcTime: 1000 * 60 * 5, // 5 min - keep in memory after unmount

  // Search params validation
  validateSearch: (search) => ({ page: Number(search.page) || 1 }),
});
```

### Route Options Reference

| Option | Type | Description |
|--------|------|-------------|
| `component` | `Component` | The route's UI component |
| `loader` | `async function` | Fetches data before render |
| `beforeLoad` | `async function` | Runs before loader (auth guards) |
| `pendingComponent` | `Component` | Shown while loading |
| `errorComponent` | `Component` | Shown on error |
| `staleTime` | `number` | ms data stays fresh (default: 0) |
| `gcTime` | `number` | ms to keep cached (default: 30 min) |
| `pendingMs` | `number` | Delay before pendingComponent shows |
| `validateSearch` | `function` | Validate/transform search params |
| `loaderDeps` | `function` | Dependencies that trigger reload |

---

## beforeLoad vs loader

**Execution order:**
1. `beforeLoad` runs sequentially (parent → child)
2. `loader` runs in parallel (all at once, after ALL beforeLoad complete)

```typescript
// Use beforeLoad for:
// - Auth guards (redirect if not logged in)
// - Adding context for child routes
// - Blocking navigation

export const Route = createFileRoute("/admin")({
  beforeLoad: async ({ context }) => {
    if (!context.auth.isLoggedIn) {
      throw redirect({ to: "/login" });
    }
    // Return values merge into context for child routes
    return { adminData: await fetchAdminData() };
  },
  loader: async () => {
    // Runs after beforeLoad succeeds
    return { stats: await fetchStats() };
  },
});
```

---

## Navigation

### Link Component

```typescript
import { Link } from "@tanstack/react-router";

<Link to="/users">Users</Link>
<Link to="/users/$id" params={{ id: "123" }}>User 123</Link>
<Link to="/search" search={{ q: "test" }}>Search</Link>
```

### Programmatic Navigation

```typescript
import { useNavigate, useRouter } from "@tanstack/react-router";

function MyComponent() {
  const navigate = useNavigate();
  const router = useRouter();

  // Navigate
  navigate({ to: "/users" });
  navigate({ to: "/users/$id", params: { id: "123" } });

  // Invalidate and reload current route data
  router.invalidate();
}
```

### useMatchRoute Hook

```typescript
import { useMatchRoute } from "@tanstack/react-router";

function Nav() {
  const matchRoute = useMatchRoute();

  const isActive = matchRoute({ to: "/users", fuzzy: true });
  return <Link className={isActive ? "active" : ""}>Users</Link>;
}
```

---

## Root Layout (__root.tsx)

```typescript
import { createRootRoute, Outlet } from "@tanstack/react-router";

export const Route = createRootRoute({
  component: RootLayout,
});

function RootLayout() {
  return (
    <html>
      <body>
        <Header />
        <Outlet />  {/* Child routes render here */}
        <Footer />
      </body>
    </html>
  );
}
```

---

## Dynamic Routes

### Single Parameter

```typescript
// src/routes/users/$id.tsx
export const Route = createFileRoute("/users/$id")({
  loader: async ({ params }) => {
    const user = await fetchUser(params.id);
    return { user };
  },
  component: UserPage,
});

function UserPage() {
  const { user } = Route.useLoaderData();
  return <div>{user.name}</div>;
}
```

### Multiple Parameters

```typescript
// src/routes/org/$orgId/project/$projectId.tsx
export const Route = createFileRoute("/org/$orgId/project/$projectId")({
  loader: async ({ params }) => {
    // params.orgId, params.projectId
  },
});
```

---

## Search Parameters

```typescript
// src/routes/search.tsx
import { z } from "zod";

const searchSchema = z.object({
  q: z.string().optional(),
  page: z.number().default(1),
});

export const Route = createFileRoute("/search")({
  validateSearch: (search) => searchSchema.parse(search),
  component: SearchPage,
});

function SearchPage() {
  const { q, page } = Route.useSearch();
  const navigate = useNavigate();

  const setPage = (newPage: number) => {
    navigate({ search: { q, page: newPage } });
  };
}
```

---

## createServerFn API

### CRITICAL: It's `inputValidator` NOT `validator`

```typescript
// CORRECT
.inputValidator(schema)

// WRONG - Does NOT exist
.validator(schema)
```

### Basic Structure

```typescript
import { createServerFn } from "@tanstack/react-start";

export const myServerFn = createServerFn({ method: "POST" })
  .inputValidator(MyZodSchema)  // NOT .validator() !!!
  .handler(async ({ data }) => {
    // data is typed from inputValidator
    return result;
  });
```

### Methods

| Option | Default | Description |
|--------|---------|-------------|
| `method` | `"GET"` | HTTP method: `"GET"` or `"POST"` |

### Chain Methods (in order)

1. `.inputValidator(schema)` - Validate input data (optional)
2. `.handler(async ({ data }) => ...)` - The server function logic (required)

### With Zod Schema

```typescript
import { z } from "zod";

const MySchema = z.object({
  name: z.string(),
  age: z.number(),
});

export const createUser = createServerFn({ method: "POST" })
  .inputValidator(MySchema)
  .handler(async ({ data }) => {
    // data is { name: string, age: number }
    return { success: true };
  });
```

### Simple Type Annotation

```typescript
export const greetUser = createServerFn({ method: "GET" })
  .inputValidator((data: { name: string }) => data)
  .handler(async ({ data }) => {
    return `Hello, ${data.name}!`;
  });
```

### No Input (GET)

```typescript
export const getData = createServerFn({ method: "GET" })
  .handler(async () => {
    return { data: "hello" };
  });
```

### Calling Server Functions

```typescript
// From component/route
const result = await myServerFn({ data: { name: "John" } });

// No input
const result = await getData();
```

---

## Common Patterns in This Codebase

### Pattern: Zod schema in separate types file

```typescript
// src/types/my-types.ts
export const MyInputSchema = z.object({ ... });
export type MyInput = z.infer<typeof MyInputSchema>;

// src/server/my-server.ts
import { MyInputSchema } from "../types/my-types";

export const myFn = createServerFn({ method: "POST" })
  .inputValidator(MyInputSchema)
  .handler(async ({ data }) => { ... });
```

### Pattern: Return typed results

```typescript
interface MyResult {
  success: boolean;
  error?: string;
}

export const myFn = createServerFn({ method: "POST" })
  .inputValidator(MySchema)
  .handler(async ({ data }): Promise<MyResult> => {
    return { success: true };
  });
```

### Pattern: Route with server function

```typescript
// src/routes/users/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { getUsers } from "../../server/users";

export const Route = createFileRoute("/users")({
  loader: async () => {
    const users = await getUsers();
    return { users };
  },
  component: UsersPage,
});
```

---

## SSR and Rendering Modes

TanStack Start renders routes on the server by default. Control this per-route:

```typescript
export const Route = createFileRoute("/client-only")({
  // Disable SSR for this route
  ssr: false,
  component: ClientOnlyPage,
});
```

---

## Error Boundaries

```typescript
export const Route = createFileRoute("/risky")({
  errorComponent: ({ error, reset }) => (
    <div>
      <p>Error: {error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  ),
  component: RiskyPage,
});
```

---

## Hooks Reference

| Hook | Description |
|------|-------------|
| `Route.useLoaderData()` | Get loader data for current route |
| `Route.useSearch()` | Get validated search params |
| `useParams()` | Get route params |
| `useNavigate()` | Programmatic navigation |
| `useRouter()` | Access router instance |
| `useMatchRoute()` | Check if route matches |
| `useRouterState()` | Access router state |
