# Vue.js Advanced

---

## Pinia — State Management

```typescript
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  // State (ref)
  const users = ref<User[]>([])
  const currentUserId = ref<string | null>(null)
  
  // Getters (computed)
  const currentUser = computed(() => 
    users.value.find(u => u.id === currentUserId.value)
  )
  const userCount = computed(() => users.value.length)
  
  // Actions (methods)
  async function fetchUsers() {
    users.value = await api.getUsers()
  }
  
  async function updateUser(id: string, data: Partial<User>) {
    const updated = await api.updateUser(id, data)
    const idx = users.value.findIndex(u => u.id === id)
    if (idx !== -1) users.value[idx] = updated
  }
  
  return { users, currentUserId, currentUser, userCount, fetchUsers, updateUser }
})

// In component
const store = useUserStore()
const { users, currentUser } = storeToRefs(store)  // Destructure with reactivity
await store.fetchUsers()
```

**Факт:** Pinia — официальная замена Vuex. Полная поддержка TypeScript, composition API, без mutations.

---

## Custom Directives

```typescript
// v-focus
app.directive('focus', {
  mounted(el) { el.focus() }
})

// v-permission
app.directive('permission', {
  mounted(el, binding) {
    if (!hasPermission(binding.value)) {
      el.parentNode?.removeChild(el)
    }
  }
})

// v-click-outside
app.directive('click-outside', {
  mounted(el, binding) {
    el.__clickOutside = (event: Event) => {
      if (!el.contains(event.target)) binding.value()
    }
    document.addEventListener('click', el.__clickOutside)
  },
  unmounted(el) {
    document.removeEventListener('click', el.__clickOutside)
  }
})
// Usage: <div v-click-outside="closeDropdown">...</div>
```

---

## Transitions & Animations

```vue
<template>
  <!-- Single element -->
  <Transition name="fade" mode="out-in">
    <div :key="currentView">...</div>
  </Transition>

  <!-- List animations -->
  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item.id">{{ item.name }}</li>
  </TransitionGroup>

  <!-- Page transitions with Router -->
  <RouterView v-slot="{ Component }">
    <Transition name="slide" mode="out-in">
      <component :is="Component" />
    </Transition>
  </RouterView>
</template>

<style scoped>
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from, .fade-leave-to { opacity: 0; }

.list-enter-active, .list-leave-active {
  transition: all 0.5s ease;
}
.list-enter-from, .list-leave-to {
  opacity: 0; transform: translateX(30px);
}
/* FLIP animation */
.list-move { transition: transform 0.5s ease; }
</style>
```

---

## Composables — логика переиспользования

```typescript
// useMouse.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)
  
  function update(e: MouseEvent) {
    x.value = e.pageX
    y.value = e.pageY
  }
  
  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))
  
  return { x, y }
}

// useDebounce
export function useDebounce<T>(fn: (...args: T[]) => void, delay: number) {
  let timer: ReturnType<typeof setTimeout>
  return (...args: T[]) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }
}

// In component
const { x, y } = useMouse()
```

---

## Provide/Inject

```typescript
// Parent
import { provide, ref } from 'vue'
const theme = ref('dark')
provide('theme', theme)
provide('updateTheme', (newTheme: string) => { theme.value = newTheme })

// Child (any depth)
import { inject } from 'vue'
const theme = inject('theme', ref('light'))  // Default value
const updateTheme = inject<(t: string) => void>('updateTheme')!
```

---

## Чек-лист

- [ ] Pinia: defineStore (setup syntax), storeToRefs, actions (async)
- [ ] Custom Directives: mounted, unmounted, binding.value
- [ ] Transitions: Transition (mode), TransitionGroup (FLIP), page transitions
- [ ] Composables: useMouse, useDebounce — переиспользование логики
- [ ] Provide/Inject: для глубокой передачи состояния
- [ ] Teleport: <Teleport to="body"> для модалок
- [ ] Suspense: <Suspense> для async components
