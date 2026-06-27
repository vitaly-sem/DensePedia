# Vue.js Practical Tips

---

## Performance

```vue
<script setup>
// 1. v-memo — memoize template parts (Vue 3.2+)
// Template re-renders only when deps change
</script>
<template>
  <div v-memo="[item.id, item.active]">
    {{ item.name }} - {{ item.price }}
  </div>
</template>

<script setup>
// 2. shallowRef для больших объектов (без глубокой реакции)
const users = shallowRef<User[]>([])
const updateUsers = (data: User[]) => { users.value = data }

// 3. computed — кэшируется
const total = computed(() => items.value.reduce((s, i) => s + i.price, 0))

// 4. v-once — статический контент (рендерится один раз)
<div v-once>{{ staticContent }}</div>

// 5. KeepAlive для сохранения состояния табов
<KeepAlive>
  <component :is="currentTab" />
</KeepAlive>

// 6. Ленивая загрузка компонентов
const HeavyComponent = defineAsyncComponent(() => import('./Heavy.vue'))
```

---

## Composition API Patterns

```typescript
// 1. Composables — переиспользование логики с lifecycle
export function usePolling(fn: () => Promise<void>, interval: number) {
  const isActive = ref(false)
  let timer: ReturnType<typeof setInterval>
  
  const start = () => {
    isActive.value = true
    timer = setInterval(() => fn(), interval)
  }
  
  const stop = () => {
    isActive.value = false
    clearInterval(timer)
  }
  
  onUnmounted(stop)
  
  return { isActive, start, stop }
}

// 2. watchEffect — авто-отслеживание зависимостей
watchEffect(() => {
  localStorage.setItem('theme', theme.value) // auto-tracks theme
})

// 3. Сброс формы
function resetForm() {
  Object.assign(form, initialFormState())
}

// 4. Debounced input
const search = ref('')
watch(search, useDebounce((val) => fetchResults(val), 300))

// 5. Template refs с Composition API
const inputRef = ref<HTMLInputElement | null>(null)
onMounted(() => inputRef.value?.focus())
```

---

## State Management

```typescript
// 1. Pinia — модульная структура
export const useUserStore = defineStore('users', () => {
  const users = ref<User[]>([])
  const loading = ref(false)
  
  async function fetchUsers() {
    loading.value = true
    try { users.value = await api.getUsers() }
    finally { loading.value = false }
  }
  
  return { users, loading, fetchUsers }
})

// 2. Сброс store
function reset() {
  const store = useUserStore()
  store.$reset() // Reset to initial state
}

// 3. Persist Pinia state (плагин)
// npm i pinia-plugin-persistedstate
const store = defineStore('cart', () => { ... }, {
  persist: { key: 'cart-storage', storage: localStorage }
})

// 4. watch в store
watch(() => cart.items, (items) => {
  localStorage.setItem('cart', JSON.stringify(items))
}, { deep: true })
```

---

## Router

```vue
<script setup>
// 1. Navigation guards с composition API
import { onBeforeRouteLeave, onBeforeRouteUpdate } from 'vue-router'

onBeforeRouteLeave((to, from) => {
  const answer = window.confirm('Unsaved changes?')
  if (!answer) return false
})

// 2. Lazy loading routes
const routes = [
  { path: '/admin', component: () => import('./Admin.vue') }
]

// 3. Router scroll behavior
const router = createRouter({
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition
    return { top: 0 }
  }
})

// 4. Navigation failure handling
const result = await router.push('/admin')
if (result === NavigationFailureType.redirected) {
  // handle
}
</script>
```

---

## Other Tips

```typescript
// 1. Teleport для модалок (в body)
<Teleport to="body">
  <Modal v-if="showModal" />
</Teleport>

// 2. Provide/Inject с symbols
export const THEME_KEY = Symbol('theme')
provide(THEME_KEY, theme)

// 3. Строгая типизация emits
const emit = defineEmits<{
  (e: 'update:id', value: number): void
  (e: 'delete'): void
}>()

// 4. CSS v-bind — реактивные CSS переменные
// <style scoped>
// .card { color: v-bind(themeColor); }

// 5. Suspense для async setup
// <Suspense><AsyncComponent /></Suspense>
```
