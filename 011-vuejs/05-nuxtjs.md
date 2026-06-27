# Nuxt.js

---

## Overview

**Факты:**
- Nuxt — **Vue framework** для SSR, SSG, SPA, ISR
- Автоматическая конфигурация: routing, meta tags, middleware, modules
- **Nuxt 3** (Vue 3 + Nitro + Vite)

```bash
npx nuxi init my-app
npm run dev    # SSG (static)
npm run build  # export static
```

**Режимы:**

| Mode | Rendering | Description |
|---|---|---|
| **SSR** (universal) | Server + Client | Default. SEO + SPA |
| **SSG** (static) | Build-time | Static files, CDN |
| **SPA** | Client only | Like Vue SPA |
| **ISR** (incremental) | Hybrid | Static + revalidate |

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: true,  // SSR mode
  nitro: {
    preset: 'vercel'  // Deploy target
  },
  routeRules: {
    '/': { prerender: true },                    // Static
    '/blog/**': { swr: 3600 },                   // ISR (revalidate 1h)
    '/api/**': { proxy: '/api' },                // Proxy
    '/admin/**': { ssr: false }                  // SPA mode
  }
})
```

---

## File-based Routing

```vue
// pages/
//   index.vue          → /
//   about.vue          → /about
//   users/
//     [id].vue         → /users/:id
//     [id]/profile.vue → /users/:id/profile
//   blog/
//     [...slug].vue    → /blog/a/b/c (catch-all)
//   admin/
//     index.vue        → /admin (client-only)

// pages/users/[id].vue
<script setup lang="ts">
const route = useRoute()
const { data: user, pending } = await useFetch(`/api/users/${route.params.id}`)

// Navigation
const router = useRouter()
function goBack() { router.back() }

definePageMeta({
  layout: 'default',
  middleware: 'auth',
  title: 'User Profile'
})
</script>

<template>
  <div v-if="pending">Loading...</div>
  <div v-else>
    <h1>{{ user.name }}</h1>
  </div>
</template>
```

---

## Data Fetching

```typescript
// useFetch — SSR-friendly fetch
const { data, pending, error, refresh } = await useFetch('/api/users', {
  key: 'users',      // Cache key
  pick: ['id', 'name'],  // Pick fields
  transform: (data) => data.map(transformUser),
  headers: { Authorization: `Bearer ${token}` },
  server: true        // Fetch on server during SSR
})

// useAsyncData — for custom fetcher
const { data } = await useAsyncData('users', () => $fetch('/api/users'))

// useLazyFetch — non-blocking
const { data } = useLazyFetch('/api/users')  // Page renders without waiting

// Refresh on demand
function refreshUsers() { refresh() }

// Error handling
if (error.value) {
  console.error(error.value.message)
}
```

---

## Auto-imports & Composables

```typescript
// composables/useCounter.ts — auto-imported
export const useCounter = () => {
  const count = ref(0)
  const increment = () => count.value++
  return { count, increment }
}

// components/ — auto-imported
// components/MyButton.vue → <MyButton />

// middleware/auth.ts — auto-imported
export default defineNuxtRouteMiddleware((to, from) => {
  const { user } = useUserSession()
  if (!user.value) return navigateTo('/login')
})

// plugins/ — auto-registered
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.provide('analytics', analytics)
})
```

---

## Modules

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@nuxtjs/tailwindcss',
    '@nuxtjs/i18n',
    '@nuxt/image',
    '@vueuse/nuxt',
    '@pinia/nuxt',
    '@nuxt/content'  // Markdown CMS
  ],
  
  // Module configuration
  i18n: {
    locales: ['en', 'ru'],
    defaultLocale: 'en'
  },
  
  image: {
    provider: 'cloudinary'
  }
})
```

---

## Чек-лист

- [ ] Режимы: SSR, SSG, SPA, ISR
- [ ] File-based routing: [param], [...slug], middleware
- [ ] Data fetching: useFetch, useAsyncData, useLazyFetch
- [ ] Auto-imports: composables, components, middleware, plugins
- [ ] Modules: Tailwind, i18n, Pinia, Content
- [ ] Deployment: Nitro (Vercel, Netlify, Node, Deno)
