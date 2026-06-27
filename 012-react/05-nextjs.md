# Next.js

---

## Overview

**Факты:**
- Next.js — **React framework** для SSR, SSG, ISR, RSC (Server Components)
- **App Router** (Next.js 13+) — новый роутинг на основе файлов
- Server Components — **RSC**: React компоненты, которые работают на сервере (0 KB JS)

```bash
npx create-next-app@latest my-app
npm run dev    # SSR dev server
npm run build  # Production build
```

**Режимы рендеринга:**

| Mode | Strategy | When |
|---|---|---|
| **SSR** | Dynamic (per request) | User-specific, real-time data |
| **SSG** | Static (build-time) | Blogs, marketing |
| **ISR** | Static + revalidate | Blog with updates, e-commerce |
| **RSC** | Server Components | Data fetching, DB access |
| **CSR** | Client only | Dashboard, user settings |

---

## App Router (Next.js 13+)

```typescript
// app/layout.tsx — корневой layout
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <nav>...</nav>
        {children}
      </body>
    </html>
  )
}

// app/page.tsx → /
// app/about/page.tsx → /about
// app/blog/[slug]/page.tsx → /blog/:slug

// app/blog/[slug]/page.tsx — Server Component (по умолчанию)
export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await db.post.findUnique({ where: { slug: params.slug } })
  // DB access directly — RSC!
  return <article>{post.title}: {post.content}</article>
}

// app/api/users/route.ts — API Route
export async function GET() {
  const users = await db.user.findMany()
  return Response.json(users)
}

// app/loading.tsx — loading state
export default function Loading() { return <Spinner /> }

// app/error.tsx — error boundary
'use client'
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return <div>Error: {error.message} <button onClick={reset}>Retry</button></div>
}
```

---

## Data Fetching

```typescript
// Server Component — прямой доступ к БД (RSC)
export default async function Page() {
  const data = await db.query('SELECT * FROM items')
  return <div>{data.map(item => <Item key={item.id} item={item} />)}</div>
}

// ISR — incremental static regeneration
export const revalidate = 3600  // Revalidate every hour

// Dynamic params
export async function generateStaticParams() {
  const posts = await db.post.findMany()
  return posts.map(post => ({ slug: post.slug }))
}

// Client Component — fetch from API
'use client'
import useSWR from 'swr'
function ClientData() {
  const { data } = useSWR('/api/data', fetcher)
  return <div>{data}</div>
}

// Server Actions (mutations)
'use server'
export async function createUser(formData: FormData) {
  const name = formData.get('name')
  await db.user.create({ data: { name } })
  revalidatePath('/users')
}
```

---

## Layouts

```typescript
// app/layout.tsx — root layout (обязательно)
export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html>
      <body>
        <header />
        {children}
        <footer />
      </body>
    </html>
  )
}

// app/dashboard/layout.tsx — nested layout
export default function DashboardLayout({ children }: { children: ReactNode }) {
  return (
    <section>
      <Sidebar />
      <main>{children}</main>
    </section>
  )
}

// app/dashboard/settings/page.tsx — uses both layouts

// Route groups — without affecting URL
// app/(marketing)/about/page.tsx → /about
// app/(dashboard)/settings/page.tsx → /settings
```

---

## Image & Font Optimization

```typescript
import Image from 'next/image'
import { Inter } from 'next/font/google'

const inter = Inter({ subsets: ['latin'] })

export default function Page() {
  return (
    <div className={inter.className}>
      <Image
        src="/hero.jpg"
        alt="Hero"
        width={1200}
        height={600}
        priority          // Above-the-fold
        placeholder="blur" // Blur-up
        blurDataURL="data:image/webp;base64,..."
      />
    </div>
  )
}
```

---

## Чек-лист

- [ ] App Router: file-based, layout nesting, route groups
- [ ] Server Components (RSC): 0 KB JS, прямой доступ к DB
- [ ] Rendering: SSR, SSG, ISR (revalidate), CSR
- [ ] Data Fetching: RSC async, Server Actions, API Routes
- [ ] Image/Font optimization: next/image, next/font
- [ ] Loading/Error states: loading.tsx, error.tsx
