---
name: repo-scaffold
description: >
  Scaffold a fresh project repo with a consistent, production-ready tech stack. Use this skill
  whenever the user wants to start a new project, initialise a new repo, bootstrap an app, or
  create a starter template — even if they say things like "start me a new project", "set up a
  fresh repo", "I want to build a new app", or "initialise this folder". Autonomously runs all
  shell commands, writes all config files, creates the folder structure, installs dependencies,
  and leaves the user with a working dev server and a comprehensive AGENTS.md. Targets beginners
  and vibe coders who want zero friction from empty folder to running app.
---

# Repo Scaffold Skill

Autonomously scaffolds a complete, working project from an empty directory. No interactive prompts — just run commands, write files, verify the result.

## Stack

| Layer | Choice |
|---|---|
| Framework | React 19 + TypeScript |
| Bundler | Vite (latest) |
| Routing | TanStack Router (file-based, Vite plugin) |
| Data fetching | TanStack Query v5 |
| Styling | Tailwind CSS v4 (Vite plugin — no PostCSS config needed) |
| Components | shadcn/ui (new-york style, Tailwind v4) |
| Backend client | PocketBase JS SDK |
| Testing | Vitest + React Testing Library + jsdom |
| Linting | ESLint (flat config) + Prettier |

---

## Pre-flight

Before running any commands, confirm:
1. You are in the correct target directory (or create it first)
2. Node.js ≥ 20 is available (`node --version`)
3. The directory is empty or the user has confirmed overwrite

If the user hasn't named their project yet, use the current directory name as the project name.
Capture the project name — you'll need it when writing `AGENTS.md` and `README.md`.

---

## Scaffold Sequence

Execute steps in order. Do not skip steps. If a step fails, diagnose and fix before continuing.

### Step 1 — Vite + React + TypeScript baseline

```bash
npm create vite@latest . -- --template react-ts
npm install
```

### Step 2 — Write config files

> **Important:** Config files must be written _before_ running shadcn init. shadcn reads
> `vite.config.ts` and `tsconfig.json` to detect Tailwind and the `@/*` path alias.

Vite 6 generates a `tsconfig.app.json` alongside `tsconfig.json`. Remove it — it conflicts
with the single-tsconfig setup below:

```bash
rm -f tsconfig.app.json
```

Write each file below to the project root exactly as shown.

---

#### `vite.config.ts`

```ts
/// <reference types="vitest/config" />
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import { tanstackRouter } from '@tanstack/router-plugin/vite'
import path from 'path'

export default defineConfig({
  plugins: [
    // tanstackRouter MUST come before react()
    tanstackRouter({ target: 'react', autoCodeSplitting: true }),
    react(),
    tailwindcss(),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      exclude: ['node_modules/', 'src/test/', 'src/routeTree.gen.ts', '*.config.*'],
    },
  },
})
```

---

#### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] },
    "ignoreDeprecations": "6.0"
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

---

#### `tsconfig.node.json`

```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "strict": true
  },
  "include": ["vite.config.ts"]
}
```

---

#### `prettier.config.js`

```js
/** @type {import('prettier').Config} */
export default {
  semi: false,
  singleQuote: true,
  trailingComma: 'all',
  printWidth: 100,
  tabWidth: 2,
  plugins: ['prettier-plugin-tailwindcss'],
}
```

---

#### `eslint.config.js` — replace the Vite default entirely

Route files export both a `Route` constant and a component, and shadcn/ui files export
variants alongside components. The default Vite ESLint config has no overrides for these,
causing `react-refresh/only-export-components` warnings that fail `--max-warnings 0`.
Replace the generated file entirely:

```js
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import tseslint from 'typescript-eslint'

export default tseslint.config(
  { ignores: ['dist', 'src/routeTree.gen.ts', 'src/components/ui/'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
    },
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': ['warn', { allowConstantExport: true }],
    },
  },
  {
    // Route and context files export both Route definitions and components — disable HMR rule
    files: ['src/routes/**/*.{ts,tsx}', 'src/context/**/*.{ts,tsx}'],
    rules: {
      'react-refresh/only-export-components': 'off',
    },
  },
  {
    // Test files don't need HMR lint rules
    files: ['src/test/**/*.{ts,tsx}', 'src/**/__tests__/**/*.{ts,tsx}'],
    rules: {
      'react-refresh/only-export-components': 'off',
    },
  },
)
```

---

#### `package.json` — merge these scripts into the existing scripts block

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit",
    "lint": "eslint . --report-unused-disable-directives --max-warnings 0",
    "format": "prettier --write \"src/**/*.{ts,tsx,css,md}\"",
    "test": "vitest",
    "test:run": "vitest run",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage"
  }
}
```

---

#### `src/index.css` — replace Vite default entirely

```css
@import "tailwindcss";

@layer base {
  :root {
    --background: oklch(1 0 0);
    --foreground: oklch(0.145 0 0);
    --card: oklch(1 0 0);
    --card-foreground: oklch(0.145 0 0);
    --primary: oklch(0.205 0 0);
    --primary-foreground: oklch(0.985 0 0);
    --secondary: oklch(0.97 0 0);
    --secondary-foreground: oklch(0.205 0 0);
    --muted: oklch(0.97 0 0);
    --muted-foreground: oklch(0.556 0 0);
    --accent: oklch(0.97 0 0);
    --accent-foreground: oklch(0.205 0 0);
    --destructive: oklch(0.577 0.245 27.325);
    --border: oklch(0.922 0 0);
    --input: oklch(0.922 0 0);
    --ring: oklch(0.708 0 0);
    --radius: 0.625rem;
  }
  .dark {
    --background: oklch(0.145 0 0);
    --foreground: oklch(0.985 0 0);
    --card: oklch(0.205 0 0);
    --card-foreground: oklch(0.985 0 0);
    --primary: oklch(0.985 0 0);
    --primary-foreground: oklch(0.205 0 0);
    --secondary: oklch(0.269 0 0);
    --secondary-foreground: oklch(0.985 0 0);
    --muted: oklch(0.269 0 0);
    --muted-foreground: oklch(0.708 0 0);
    --accent: oklch(0.269 0 0);
    --accent-foreground: oklch(0.985 0 0);
    --destructive: oklch(0.704 0.191 22.216);
    --border: oklch(1 0 0 / 10%);
    --input: oklch(1 0 0 / 15%);
    --ring: oklch(0.556 0 0);
  }
}

@layer base {
  * { @apply border-border; }
  body { @apply bg-background text-foreground; }
}
```

---

#### `src/vite-env.d.ts`

```ts
/// <reference types="vite/client" />
```

Vite generates this file during `npm create vite`, but shadcn init may remove it. Write it
explicitly so `import.meta.env` always has types.

---

#### `.env.example`

```env
# PocketBase backend URL — copy this file to .env.local and fill in your values
# Default when running PocketBase locally:
VITE_PB_URL=http://127.0.0.1:8090

# Set to "true" to enable verbose PocketBase SDK logging in development
VITE_PB_DEBUG=false
```

---

#### `.gitignore` — append to Vite default

```
.env.local
.env.*.local
coverage
html
.DS_Store
Thumbs.db
```

---

#### `.vscode/extensions.json`

```json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "bradlc.vscode-tailwindcss",
    "vitest.explorer",
    "ms-vscode.vscode-typescript-next"
  ]
}
```

---

### Step 3 — Initialise shadcn/ui

> **Note:** shadcn init rewrites `package.json`. Running it here — before the `npm install`
> steps below — ensures it cannot remove dependencies you have already installed.

```bash
npx shadcn@latest init --defaults
```

If prompted for style, select **new-york**. If prompted for base colour, select **zinc**.

Then add the starter component set:

```bash
npx shadcn@latest add button card input label badge
```

### Step 4 — Install runtime dependencies

```bash
npm install \
  @tanstack/react-router \
  @tanstack/react-query \
  @tanstack/react-query-devtools \
  pocketbase
```

### Step 5 — Install dev dependencies

```bash
npm install -D \
  @tanstack/router-plugin \
  tailwindcss \
  @tailwindcss/vite \
  prettier \
  prettier-plugin-tailwindcss \
  vitest \
  @vitest/coverage-v8 \
  @vitest/ui \
  @testing-library/react \
  @testing-library/user-event \
  @testing-library/jest-dom \
  jsdom
```

---

### Step 6 — Write source files

Write each file below to the path shown, relative to the project root.

---

#### `src/lib/pb.ts`

```ts
import PocketBase from 'pocketbase'

// TODO: Replace with your PocketBase server URL in .env.local
const pb = new PocketBase(import.meta.env.VITE_PB_URL ?? 'http://127.0.0.1:8090')

pb.autoCancellation(false)

export default pb
```

---

#### `src/lib/queryClient.ts`

```ts
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Don't retry on 4xx errors from PocketBase
      retry: (failureCount, error: unknown) => {
        if (error instanceof Error && 'status' in error) {
          const status = (error as { status: number }).status
          if (status >= 400 && status < 500) return false
        }
        return failureCount < 2
      },
      staleTime: 1000 * 60,
    },
  },
})
```

---

#### `src/context/AuthContext.tsx`

```tsx
import { createContext, useContext, useEffect, useState, type ReactNode } from 'react'
import type { RecordModel } from 'pocketbase'
import pb from '@/lib/pb'

interface AuthState {
  user: RecordModel | null
  isLoggedIn: boolean
  isLoading: boolean
}

interface AuthContextValue extends AuthState {
  logout: () => void
}

const AuthContext = createContext<AuthContextValue | null>(null)

export function AuthProvider({ children }: { children: ReactNode }) {
  const [state, setState] = useState<AuthState>({
    user: pb.authStore.record ?? null,
    isLoggedIn: pb.authStore.isValid,
    isLoading: false,
  })

  useEffect(() => {
    const unsubscribe = pb.authStore.onChange((token, record) => {
      setState({
        user: record ?? null,
        isLoggedIn: !!token && pb.authStore.isValid,
        isLoading: false,
      })
    })
    return unsubscribe
  }, [])

  const logout = () => pb.authStore.clear()

  return <AuthContext.Provider value={{ ...state, logout }}>{children}</AuthContext.Provider>
}

export function useAuth() {
  const ctx = useContext(AuthContext)
  if (!ctx) throw new Error('useAuth must be used inside <AuthProvider>')
  return ctx
}
```

---

#### `src/main.tsx`

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { RouterProvider, createRouter } from '@tanstack/react-router'
import { QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { AuthProvider } from '@/context/AuthContext'
import { queryClient } from '@/lib/queryClient'
import { routeTree } from './routeTree.gen'
import '@/index.css'

const router = createRouter({ routeTree })

declare module '@tanstack/react-router' {
  interface Register { router: typeof router }
}

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <RouterProvider router={router} />
      </AuthProvider>
      {/* TODO: Remove before production */}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  </StrictMode>,
)
```

---

#### `src/routes/__root.tsx`

```tsx
import { createRootRoute, Link, Outlet } from '@tanstack/react-router'
import { useAuth } from '@/context/AuthContext'

export const Route = createRootRoute({ component: RootLayout })

function RootLayout() {
  const { isLoggedIn, user, logout } = useAuth()
  return (
    <div className="min-h-screen bg-background text-foreground">
      {/* TODO: Replace with your own nav design */}
      <nav className="border-b border-border px-6 py-3 flex items-center justify-between">
        <Link to="/" className="font-semibold text-lg tracking-tight">
          {/* TODO: Replace with your app name */}
          My App
        </Link>
        <div className="flex items-center gap-4 text-sm">
          {isLoggedIn ? (
            <>
              <span className="text-muted-foreground">{user?.email}</span>
              <button
                onClick={logout}
                className="text-muted-foreground hover:text-foreground transition-colors"
              >
                Sign out
              </button>
            </>
          ) : (
            <Link to="/login" className="text-muted-foreground hover:text-foreground transition-colors">
              Sign in
            </Link>
          )}
        </div>
      </nav>
      <main className="px-6 py-8 max-w-5xl mx-auto">
        <Outlet />
      </main>
    </div>
  )
}
```

---

#### `src/routes/index.tsx`

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'

export const Route = createFileRoute('/')({ component: HomePage })

export function HomePage() {
  return (
    <div className="flex flex-col gap-8">
      <div>
        {/* TODO: Replace with your app's hero copy */}
        <h1 className="text-3xl font-bold tracking-tight mb-2">Welcome</h1>
        <p className="text-muted-foreground">
          Your app is running. Start building in <code className="text-sm">src/routes/</code>.
        </p>
      </div>
      {/* TODO: Replace these cards with your real content */}
      <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
        <Card>
          <CardHeader><CardTitle className="text-base">Add a route</CardTitle></CardHeader>
          <CardContent className="text-sm text-muted-foreground">
            Create <code>src/routes/my-page.tsx</code> and it becomes <code>/my-page</code> automatically.
          </CardContent>
        </Card>
        <Card>
          <CardHeader><CardTitle className="text-base">Connect PocketBase</CardTitle></CardHeader>
          <CardContent className="text-sm text-muted-foreground">
            Set <code>VITE_PB_URL</code> in <code>.env.local</code>, then use <code>pb</code> from <code>@/lib/pb</code>.
          </CardContent>
        </Card>
        <Card>
          <CardHeader><CardTitle className="text-base">Auth is ready</CardTitle></CardHeader>
          <CardContent className="text-sm text-muted-foreground">
            Use <code>useAuth()</code> anywhere to get the current user, login state, and a logout function.
          </CardContent>
        </Card>
      </div>
    </div>
  )
}
```

---

#### `src/routes/login.tsx`

```tsx
import { createFileRoute, useNavigate } from '@tanstack/react-router'
import { useState } from 'react'
import pb from '@/lib/pb'
import { useAuth } from '@/context/AuthContext'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'

export const Route = createFileRoute('/login')({ component: LoginPage })

function LoginPage() {
  const { isLoggedIn } = useAuth()
  const navigate = useNavigate()
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState<string | null>(null)
  const [loading, setLoading] = useState(false)

  if (isLoggedIn) { navigate({ to: '/' }); return null }

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError(null)
    setLoading(true)
    try {
      // TODO: Change 'users' if your PocketBase auth collection has a different name
      await pb.collection('users').authWithPassword(email, password)
      navigate({ to: '/' })
    } catch {
      setError('Invalid email or password. Please try again.')
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="flex justify-center items-start pt-16">
      <Card className="w-full max-w-sm">
        <CardHeader>
          {/* TODO: Update title to match your app */}
          <CardTitle>Sign in</CardTitle>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit} className="flex flex-col gap-4">
            <div className="flex flex-col gap-1.5">
              <Label htmlFor="email">Email</Label>
              <Input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} autoComplete="email" required />
            </div>
            <div className="flex flex-col gap-1.5">
              <Label htmlFor="password">Password</Label>
              <Input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} autoComplete="current-password" required />
            </div>
            {error && <p className="text-sm text-destructive">{error}</p>}
            <Button type="submit" disabled={loading} className="w-full">
              {loading ? 'Signing in…' : 'Sign in'}
            </Button>
          </form>
        </CardContent>
      </Card>
    </div>
  )
}
```

---

#### `src/components/ProtectedRoute.tsx`

```tsx
import { useAuth } from '@/context/AuthContext'
import { Navigate } from '@tanstack/react-router'
import type { ReactNode } from 'react'

interface Props {
  children: ReactNode
  redirectTo?: string
}

// TODO: Use this to guard any route that requires login:
// export const Route = createFileRoute('/dashboard')({
//   component: () => <ProtectedRoute><Dashboard /></ProtectedRoute>,
// })
export function ProtectedRoute({ children, redirectTo = '/login' }: Props) {
  const { isLoggedIn, isLoading } = useAuth()

  if (isLoading) {
    return (
      <div className="flex justify-center items-center h-32 text-muted-foreground text-sm">
        Loading…
      </div>
    )
  }

  if (!isLoggedIn) return <Navigate to={redirectTo} />

  return <>{children}</>
}
```

---

#### `src/routes/admin.tsx`

An in-app admin dashboard for superusers. Requires the logged-in user to have an `isAdmin` (or equivalent) field set to `true` on their PocketBase user record — enforce this in PocketBase collection rules, not only here.

> **Note:** This is separate from the PocketBase built-in admin UI at `http://[pb-host]/_/`.
> That UI is for database/schema management. This route is for in-app admin features
> (user management, moderation, app-level reporting, etc.) visible to your superusers.

```tsx
import { createFileRoute, useNavigate } from '@tanstack/react-router'
import { useEffect } from 'react'
import { useQuery } from '@tanstack/react-query'
import { useAuth } from '@/context/AuthContext'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Badge } from '@/components/ui/badge'
import pb from '@/lib/pb'

export const Route = createFileRoute('/admin')({ component: AdminPage })

function AdminPage() {
  const { isLoggedIn, isLoading, user } = useAuth()
  const navigate = useNavigate()

  // TODO: Replace 'isAdmin' with whatever field you use on your PocketBase users collection
  // to designate superusers (e.g. 'role === "admin"', 'isSuperuser', etc.)
  const isAdmin = (user as Record<string, unknown> | null)?.isAdmin === true

  // Redirect non-admins — auth check is enforced here AND in PocketBase collection rules
  useEffect(() => {
    if (!isLoading && (!isLoggedIn || !isAdmin)) {
      navigate({ to: '/' })
    }
  }, [isLoading, isLoggedIn, isAdmin, navigate])

  // TODO: Replace 'users' with your actual PocketBase collection name if different
  const { data: users, isLoading: usersLoading } = useQuery({
    queryKey: ['admin', 'users'],
    queryFn: () => pb.collection('users').getList(1, 50, { sort: '-created' }),
    enabled: isLoggedIn && isAdmin,
  })

  if (isLoading || (!isLoggedIn || !isAdmin)) {
    return (
      <div className="flex justify-center items-center h-32 text-muted-foreground text-sm">
        Loading…
      </div>
    )
  }

  return (
    <div className="flex flex-col gap-8">
      <div>
        <h1 className="text-3xl font-bold tracking-tight mb-1">Admin</h1>
        <p className="text-muted-foreground text-sm">
          In-app administration. For database and schema management, use the{' '}
          {/* TODO: Update this URL if PocketBase is not running locally */}
          <a
            href={`${import.meta.env.VITE_PB_URL ?? 'http://127.0.0.1:8090'}/_/`}
            target="_blank"
            rel="noopener noreferrer"
            className="underline hover:text-foreground transition-colors"
          >
            PocketBase admin UI ↗
          </a>
          .
        </p>
      </div>

      {/* TODO: Replace this users panel with the admin features your app needs */}
      <Card>
        <CardHeader>
          <CardTitle className="text-base flex items-center gap-2">
            Users
            {!usersLoading && (
              <Badge variant="secondary">{users?.totalItems ?? 0}</Badge>
            )}
          </CardTitle>
        </CardHeader>
        <CardContent>
          {usersLoading ? (
            <p className="text-sm text-muted-foreground">Loading users…</p>
          ) : (
            <ul className="divide-y divide-border text-sm">
              {users?.items.map((u) => (
                <li key={u.id} className="py-2 flex items-center justify-between">
                  <span>{u.email}</span>
                  <span className="text-muted-foreground text-xs">
                    {new Date(u.created).toLocaleDateString()}
                  </span>
                </li>
              ))}
            </ul>
          )}
        </CardContent>
      </Card>
    </div>
  )
}
```

---

### Step 7 — Write test suite

Write each file below to the path shown.

---

#### `src/test/setup.ts`

```ts
import '@testing-library/jest-dom'
import { afterEach, vi } from 'vitest'
import { cleanup } from '@testing-library/react'

afterEach(() => { cleanup() })

// Provide a default PocketBase URL so the client initialises cleanly in tests
vi.stubEnv('VITE_PB_URL', 'http://localhost:8090')
```

---

#### `src/test/test-utils.tsx`

```tsx
import { type ReactElement, type ReactNode } from 'react'
import { render, type RenderOptions } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },
      mutations: { retry: false },
    },
  })
}

function AllProviders({ children, queryClient }: { children: ReactNode; queryClient?: QueryClient }) {
  const client = queryClient ?? createTestQueryClient()
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>
}

interface CustomRenderOptions extends Omit<RenderOptions, 'wrapper'> {
  queryClient?: QueryClient
}

function renderWithProviders(ui: ReactElement, options?: CustomRenderOptions) {
  const { queryClient, ...renderOptions } = options ?? {}
  return render(ui, {
    wrapper: ({ children }) => <AllProviders queryClient={queryClient}>{children}</AllProviders>,
    ...renderOptions,
  })
}

export * from '@testing-library/react'
export { renderWithProviders as render, createTestQueryClient }
```

---

#### `src/test/mocks/pb.ts`

```ts
import { vi } from 'vitest'

// Full mock of the PocketBase singleton.
// Usage: vi.mock('@/lib/pb', () => ({ default: mockPb }))
export const mockPb = {
  authStore: {
    record: null as unknown,
    isValid: false,
    token: '',
    clear: vi.fn(),
    onChange: vi.fn(() => () => {}), // returns an unsubscribe noop
  },
  collection: vi.fn().mockReturnValue({
    authWithPassword: vi.fn(),
    getList: vi.fn(),
    getOne: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    delete: vi.fn(),
    subscribe: vi.fn(),
    unsubscribe: vi.fn(),
  }),
  autoCancellation: vi.fn(),
}
```

---

#### `src/test/mocks/handlers.ts`

```ts
import { vi } from 'vitest'

// Simulates a PocketBase getList response shape
export function mockListResponse<T>(items: T[], totalItems?: number) {
  return {
    page: 1,
    perPage: items.length,
    totalItems: totalItems ?? items.length,
    totalPages: 1,
    items,
  }
}

// Minimal PocketBase record shape
export function mockRecord<T extends Record<string, unknown>>(fields: T, id = 'test_record_id') {
  return {
    id,
    collectionId: 'test_collection_id',
    collectionName: 'test_collection',
    created: '2024-01-01 00:00:00.000Z',
    updated: '2024-01-01 00:00:00.000Z',
    ...fields,
  }
}

// Minimal PocketBase auth record shape
export function mockAuthRecord(overrides: Record<string, unknown> = {}) {
  return mockRecord({
    email: 'test@example.com',
    username: 'testuser',
    verified: true,
    emailVisibility: false,
    ...overrides,
  })
}

// Make a vi.fn() that resolves with a value
export function resolveWith<T>(value: T) {
  return vi.fn().mockResolvedValue(value)
}

// Make a vi.fn() that rejects with a PocketBase-style error
export function rejectWith(message: string, status = 400) {
  return vi.fn().mockRejectedValue(
    Object.assign(new Error(message), { status, data: { message } }),
  )
}
```

---

#### `src/components/__tests__/HomePage.test.tsx`

```tsx
import { describe, it, expect } from 'vitest'
import { screen } from '@testing-library/react'
import { render } from '@/test/test-utils'
import { HomePage } from '@/routes/index'

describe('HomePage', () => {
  it('renders the welcome heading', () => {
    render(<HomePage />)
    expect(screen.getByRole('heading', { name: /welcome/i })).toBeInTheDocument()
  })

  it('shows the Add a route feature card', () => {
    render(<HomePage />)
    expect(screen.getByText(/add a route/i)).toBeInTheDocument()
  })

  it('shows the PocketBase feature card', () => {
    render(<HomePage />)
    expect(screen.getByText(/connect pocketbase/i)).toBeInTheDocument()
  })
})
```

---

#### `src/components/__tests__/ProtectedRoute.test.tsx`

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { screen } from '@testing-library/react'
import { render } from '@/test/test-utils'
import { ProtectedRoute } from '@/components/ProtectedRoute'

const mockUseAuth = vi.fn()
vi.mock('@/context/AuthContext', () => ({
  useAuth: () => mockUseAuth(),
  AuthProvider: ({ children }: { children: React.ReactNode }) => <>{children}</>,
}))

vi.mock('@tanstack/react-router', () => ({
  Navigate: ({ to }: { to: string }) => <div data-testid="navigate" data-to={to} />,
}))

describe('ProtectedRoute', () => {
  beforeEach(() => { vi.clearAllMocks() })

  it('renders children when user is logged in', () => {
    mockUseAuth.mockReturnValue({ isLoggedIn: true, isLoading: false })
    render(<ProtectedRoute><div>Protected content</div></ProtectedRoute>)
    expect(screen.getByText('Protected content')).toBeInTheDocument()
  })

  it('redirects to /login when user is not logged in', () => {
    mockUseAuth.mockReturnValue({ isLoggedIn: false, isLoading: false })
    render(<ProtectedRoute><div>Protected content</div></ProtectedRoute>)
    expect(screen.getByTestId('navigate')).toHaveAttribute('data-to', '/login')
  })

  it('shows loading state while auth resolves', () => {
    mockUseAuth.mockReturnValue({ isLoggedIn: false, isLoading: true })
    render(<ProtectedRoute><div>Protected content</div></ProtectedRoute>)
    expect(screen.getByText(/loading/i)).toBeInTheDocument()
    expect(screen.queryByText('Protected content')).not.toBeInTheDocument()
  })

  it('redirects to a custom path when redirectTo is provided', () => {
    mockUseAuth.mockReturnValue({ isLoggedIn: false, isLoading: false })
    render(<ProtectedRoute redirectTo="/sign-in"><div>Protected content</div></ProtectedRoute>)
    expect(screen.getByTestId('navigate')).toHaveAttribute('data-to', '/sign-in')
  })
})
```

---

#### `src/context/__tests__/AuthContext.test.tsx`

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { renderHook, act } from '@testing-library/react'
import type { ReactNode } from 'react'
import { AuthProvider, useAuth } from '@/context/AuthContext'
import { mockAuthRecord } from '@/test/mocks/handlers'

// vi.mock is hoisted above imports by Vitest, so mockPb must be defined with vi.hoisted()
// rather than imported — otherwise it's undefined when the factory runs.
const mockPb = vi.hoisted(() => ({
  authStore: {
    record: null as unknown,
    isValid: false,
    token: '',
    clear: vi.fn(),
    onChange: vi.fn(() => () => {}),
  },
  collection: vi.fn(() => ({
    authWithPassword: vi.fn(),
    getList: vi.fn(),
    getOne: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    delete: vi.fn(),
    subscribe: vi.fn(),
    unsubscribe: vi.fn(),
  })),
  autoCancellation: vi.fn(),
}))

vi.mock('@/lib/pb', () => ({ default: mockPb }))

const wrapper = ({ children }: { children: ReactNode }) => (
  <AuthProvider>{children}</AuthProvider>
)

describe('AuthContext', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockPb.authStore.isValid = false
    mockPb.authStore.record = null
    mockPb.authStore.token = ''
  })

  it('starts with logged-out state when authStore is empty', () => {
    const { result } = renderHook(() => useAuth(), { wrapper })
    expect(result.current.isLoggedIn).toBe(false)
    expect(result.current.user).toBeNull()
  })

  it('starts with logged-in state when authStore has a valid token', () => {
    const user = mockAuthRecord()
    mockPb.authStore.isValid = true
    mockPb.authStore.record = user
    const { result } = renderHook(() => useAuth(), { wrapper })
    expect(result.current.isLoggedIn).toBe(true)
    expect(result.current.user).toEqual(user)
  })

  it('calls pb.authStore.clear() when logout is invoked', () => {
    mockPb.authStore.isValid = true
    mockPb.authStore.record = mockAuthRecord()
    const { result } = renderHook(() => useAuth(), { wrapper })
    act(() => { result.current.logout() })
    expect(mockPb.authStore.clear).toHaveBeenCalledOnce()
  })

  it('subscribes to authStore changes on mount', () => {
    renderHook(() => useAuth(), { wrapper })
    expect(mockPb.authStore.onChange).toHaveBeenCalledOnce()
  })

  it('throws when useAuth is used outside of AuthProvider', () => {
    vi.spyOn(console, 'error').mockImplementation(() => {})
    expect(() => renderHook(() => useAuth())).toThrow('useAuth must be used inside <AuthProvider>')
  })
})
```

---

### Step 8 — Write AGENTS.md

Write this file to the project root as `AGENTS.md`. Replace `[APP_NAME]` with the actual project name.

```markdown
# AGENTS.md

> Instructions for AI coding agents working on **[APP_NAME]**.
> Keep this file accurate. Outdated instructions are worse than none.

## Project overview

[APP_NAME] is a client-side React SPA backed by a self-hosted PocketBase server.
The frontend talks directly to PocketBase from the browser — there is no Node.js middleware layer.

## Tech stack

| Layer | Choice | Key files |
|---|---|---|
| Framework | React 19 + TypeScript | `src/` |
| Bundler | Vite | `vite.config.ts` |
| Routing | TanStack Router (file-based) | `src/routes/` · `src/routeTree.gen.ts` (auto-generated) |
| Data fetching | TanStack Query v5 | `src/lib/queryClient.ts` |
| Styling | Tailwind CSS v4 | `src/index.css` |
| Components | shadcn/ui (new-york) | `src/components/ui/` |
| Backend client | PocketBase JS SDK | `src/lib/pb.ts` |
| Auth | PocketBase authStore + React context | `src/context/AuthContext.tsx` |
| Testing | Vitest + React Testing Library | `src/test/` |
| Lint/Format | ESLint + Prettier | `prettier.config.js` |

## Essential commands

```bash
npm run dev           # Start dev server → http://localhost:5173
npm run build         # Type-check + production build
npm run typecheck     # Type-check only
npm run lint          # Lint — must pass with 0 warnings
npm run format        # Auto-format all source files
npm run test:run      # Run tests once (CI)
npm run test:coverage # Generate coverage report
```

**Always run `npm run typecheck && npm run lint && npm run test:run` before finishing any task.**

## Project structure

```
src/
├── routes/           # One file = one route. File name = URL path.
│   ├── __root.tsx    # Root layout (shared nav/shell)
│   ├── index.tsx     # → /
│   ├── login.tsx     # → /login
│   └── admin.tsx     # → /admin  (requires isAdmin on user record)
├── components/
│   ├── ui/           # shadcn/ui — DO NOT edit directly, use CLI to update
│   └── *.tsx         # Your shared components
├── context/
│   └── AuthContext.tsx
├── lib/
│   ├── pb.ts         # PocketBase singleton — import `pb` from here
│   └── queryClient.ts
├── hooks/            # Custom hooks
└── test/
    ├── setup.ts
    ├── test-utils.tsx  # Use render() from here, not from @testing-library/react
    └── mocks/
        ├── pb.ts       # PocketBase mock
        └── handlers.ts # Response factories
```

## Routing rules

- **New page:** Create `src/routes/my-page.tsx` → available at `/my-page`. No config needed.
- **Nested routes:** `src/routes/settings/profile.tsx` → `/settings/profile`
- **Route params:** `src/routes/items/$id.tsx` → `/items/:id` with typed params via `Route.useParams()`
- **Protected page:** Wrap route component with `<ProtectedRoute>` from `@/components/ProtectedRoute`
- **`routeTree.gen.ts` is auto-generated** — never edit it manually

## Admin route vs PocketBase admin UI

There are two separate admin surfaces — do not confuse them:

| Surface | URL | Purpose |
|---|---|---|
| **PocketBase admin UI** | `http://[pb-host]/_/` | Database schema, collections, raw records, users, auth settings, file storage, logs, backups. Access requires PocketBase superuser credentials. Never linked from the app UI in production. |
| **In-app admin route** | `/admin` | App-level admin features for superusers (user listing, moderation, reporting). Requires `isAdmin: true` on the PocketBase user record. Enforced in `src/routes/admin.tsx` AND in PocketBase collection rules. |

**Security rules for the admin route:**
- The `isAdmin` field check in `src/routes/admin.tsx` is a UX guard — it redirects non-admins in the browser
- The real security enforcement must be set in PocketBase collection rules (restrict list/view to `@request.auth.isAdmin = true`)
- Never expose PocketBase superuser credentials in the frontend or `.env` files
- The PocketBase admin UI link in `src/routes/admin.tsx` reads from `VITE_PB_URL` — update that env var when deploying

## PocketBase patterns

```ts
import pb from '@/lib/pb'

// List records (use inside useQuery)
await pb.collection('posts').getList(1, 20, { filter: 'published = true' })

// Single record
await pb.collection('posts').getOne(id)

// Create / update / delete
await pb.collection('posts').create({ title: 'Hello' })
await pb.collection('posts').update(id, { title: 'Updated' })
await pb.collection('posts').delete(id)

// Real-time (remember to unsubscribe on unmount)
pb.collection('posts').subscribe('*', (e) => console.log(e.action, e.record))
pb.collection('posts').unsubscribe()
```

**Auth collection name:** Default is `users`. If changed in PocketBase admin, update `src/routes/login.tsx`.

## TanStack Query patterns

```ts
// Fetch a list
const { data, isLoading, error } = useQuery({
  queryKey: ['posts'],
  queryFn: () => pb.collection('posts').getList(1, 50),
})

// Mutate and invalidate cache
const queryClient = useQueryClient()
const mutation = useMutation({
  mutationFn: (data: NewPost) => pb.collection('posts').create(data),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['posts'] }),
})
```

**Query key convention:** `[collectionName]` for lists, `[collectionName, id]` for single records.

## Auth patterns

```tsx
const { user, isLoggedIn, logout } = useAuth()
```

## Testing rules

- **Import `render` from `@/test/test-utils`**, not from `@testing-library/react`
- **Mock PocketBase** in every test using `@/lib/pb`: `vi.mock('@/lib/pb', () => ({ default: mockPb }))`
- Use helpers from `@/test/mocks/handlers.ts` to build test data (`mockRecord`, `resolveWith`, etc.)
- Test user-observable behaviour — prefer `screen.getByRole` over `getByTestId`
- All tests must pass before a task is complete: `npm run test:run`

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `VITE_PB_URL` | Yes | PocketBase server URL |
| `VITE_PB_DEBUG` | No | `"true"` for verbose SDK logging |

- Set in `.env.local` (never committed)
- All frontend vars must be prefixed `VITE_`
- Access via `import.meta.env.VITE_*`

## What NOT to do

- Do not edit `src/components/ui/` — managed by shadcn/ui CLI
- Do not edit `src/routeTree.gen.ts` — auto-generated
- Do not commit `.env.local`
- Do not import from `@testing-library/react` directly — use `@/test/test-utils`
- Do not add SSR — this is a client-only SPA
- Do not use `useEffect` for data fetching — use `useQuery`
- Do not write raw `fetch()` to PocketBase — use the `pb` client

## Adding features — checklist

1. **New page:** `src/routes/my-page.tsx` with `createFileRoute('/my-page')({ component: MyPage })`
2. **New data hook:** `src/hooks/useMyCollection.ts` using `useQuery` + `pb.collection(...)`
3. **New mutation:** `useMutation` + invalidate cache on success
4. **New component:** `src/components/MyComponent.tsx` + sibling `__tests__/MyComponent.test.tsx`
5. **New UI primitive:** `npx shadcn@latest add [name]`
6. **Protected page:** Wrap in `<ProtectedRoute>`
7. **Admin-only feature:** Add to `src/routes/admin.tsx` AND add a matching PocketBase collection rule (`@request.auth.isAdmin = true`)

## Getting unstuck

- Route not appearing? Check filename case in `src/routes/` — TanStack Router is case-sensitive
- PocketBase 401? Token expired or user not logged in — check `pb.authStore.isValid`
- Type errors in route params? Use `Route.useParams()` — fully typed from the filename
- Test failing with "no QueryClient"? Use `render` from `@/test/test-utils`
- shadcn component missing? Run `npx shadcn@latest add [component-name]`
```

---

### Step 9 — Write README.md

Overwrite the Vite default `README.md` with the content below. Replace `[APP_NAME]` with the actual project name.

```markdown
# [APP_NAME]

> TODO: One sentence describing what this app does.

## Stack

React 19 · Vite · TanStack Router · TanStack Query · Tailwind CSS v4 · shadcn/ui · PocketBase

## Prerequisites

- [Node.js](https://nodejs.org/) ≥ 20
- A running [PocketBase](https://pocketbase.io/) server

## Getting started

```bash
npm install
cp .env.example .env.local   # then fill in VITE_PB_URL
npm run dev                   # → http://localhost:5173
```

## PocketBase setup

1. [Download PocketBase](https://pocketbase.io/docs/)
2. Run it: `./pocketbase serve`
3. Open the **PocketBase admin UI** at `http://127.0.0.1:8090/_/` to manage your database schema, collections, users, auth settings, file storage, and backups
4. Create your collections and set your collection rules there

## Admin access

| Surface | URL | Who can access |
|---|---|---|
| PocketBase admin UI | `http://[pb-host]/_/` | PocketBase superusers only — for DB/schema management |
| In-app admin route | `/admin` | App users with `isAdmin: true` on their user record |

To grant a user in-app admin access, set `isAdmin = true` on their record in the PocketBase admin UI, and ensure your PocketBase collection rules enforce this field server-side.

## Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start dev server |
| `npm run build` | Production build |
| `npm run typecheck` | Type-check only |
| `npm run lint` | Lint source |
| `npm run format` | Auto-format source |
| `npm run test:run` | Run tests once |
| `npm run test:coverage` | Coverage report |

## AI coding agents

See [AGENTS.md](./AGENTS.md) for instructions on how AI coding agents should work with this codebase.
```

---

### Step 10 — Git init

```bash
git init
git add -A
git commit -m "chore: initial scaffold"
```

### Step 11 — Verify

```bash
npm run typecheck
npm run lint
npm run test:run
npm run build
```

All four commands must exit 0. Fix any errors before finishing.

---

## Final message to user

After a successful scaffold, print exactly this (with the real project name substituted):

```
✅ [APP_NAME] scaffolded successfully.

Stack: React 19 · Vite · TanStack Router · TanStack Query · Tailwind v4 · shadcn/ui · PocketBase

Next steps:
  1. Start PocketBase: ./pocketbase serve  (default: http://127.0.0.1:8090)
  2. Copy .env.example → .env.local and set VITE_PB_URL
  3. npm run dev  →  http://localhost:5173

Admin surfaces:
  • http://[pb-host]/_/   — PocketBase admin UI (DB, schema, users, auth settings)
  • /admin                — In-app admin route (set isAdmin=true on a user record to access)

Key files:
  • AGENTS.md                        — read this before asking an AI to help
  • src/lib/pb.ts                    — PocketBase client (import pb from here)
  • src/routes/                      — add new pages here (filename = URL)
  • src/routes/admin.tsx             — in-app admin dashboard
  • src/context/AuthContext.tsx      — useAuth() hook
  • src/test/test-utils.tsx          — use render() from here in all tests
```
