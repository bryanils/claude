# TUI Development Patterns

## CRITICAL CONTEXT PATTERNS

## TanStack Start Framework

### File-Based Routing
- Routes live in `src/routes/`
- `__root.tsx` is root layout
- `index.tsx` = `/`
- `about.tsx` = `/about`
- Nested: `users/index.tsx` = `/users`
- Dynamic: `users/$id.tsx` = `/users/:id`

### Route Definition
```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/users')({
  component: UsersPage,
  loader: async () => {
    return await getUsers() // runs server-side
  }
})

function UsersPage() {
  const data = Route.useLoaderData()
  return <div>{/* ... */}</div>
}
```

### Server Functions
```tsx
import { createServerFn } from '@tanstack/react-start'

// GET (default)
export const getData = createServerFn().handler(async () => {
  return { data: 'server-only logic' }
})

// POST with Zod
const schema = z.object({ name: z.string() })

export const createUser = createServerFn({ method: 'POST' })
  .inputValidator(schema)
  .handler(async ({ data }) => {
    return { userId: 123 }
  })
```

**Client usage:**
```tsx
import { useServerFn } from '@tanstack/react-start'

const createUserFn = useServerFn(createUser)
await createUserFn({ data: { name: 'John' } })
```

### Environment-Specific Functions
```tsx
// Server only - crashes if called from client
import { createServerOnlyFn } from '@tanstack/react-start'
const foo = createServerOnlyFn(() => process.env.SECRET)

// Client only - crashes if called from server
import { createClientOnlyFn } from '@tanstack/react-start'
const bar = createClientOnlyFn(() => window.location.href)

// Isomorphic - different behavior per environment
import { createIsomorphicFn } from '@tanstack/react-start'
const getEnv = createIsomorphicFn()
  .server(() => 'server')
  .client(() => 'client')
```

### Middleware
```tsx
import { createMiddleware } from '@tanstack/react-start'

const authMiddleware = createMiddleware({ type: 'function' })
  .inputValidator(z.object({ token: z.string() }))
  .server(async ({ next, data }) => {
    const user = await validateToken(data.token)
    return next({ context: { user } })
  })

// Apply to server function
export const getUser = createServerFn()
  .middleware([authMiddleware])
  .handler(async ({ context }) => {
    return context.user // typed!
  })
```

### Server Routes (API endpoints)
```tsx
// routes/api/hello.ts
import { createFileRoute } from '@tanstack/react-router'
import { json } from '@tanstack/react-start'

export const Route = createFileRoute('/api/hello')({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return json({ message: 'Hello' })
      },
      POST: async ({ request }) => {
        const body = await request.json()
        return json({ received: body })
      }
    }
  }
})
```

### Key Patterns
- **No "use server" directive needed**
- Server functions auto-serialize data
- Use `.inputValidator()` for type-safe validation
- Call server functions with `useServerFn()` hook
- Loaders run server-side by default
- Use `createServerOnlyFn` for sensitive operations
- Middleware adds context to handlers
