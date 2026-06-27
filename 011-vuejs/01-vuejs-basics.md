# Vue.js Basics

---

## Composition API (Vue 3)

```vue
<script setup lang="ts">
import { ref, reactive, computed, watch, onMounted } from 'vue'

// Reactive state
const count = ref(0)                    // Primitive → .value
const user = reactive({                 // Object → direct access
  name: 'John',
  age: 30
})

// Computed
const doubleCount = computed(() => count.value * 2)

// Watchers
watch(count, (newVal, oldVal) => {
  console.log(`Count changed: ${oldVal} → ${newVal}`)
})

watchEffect(() => {  // Auto-tracks dependencies
  console.log(`Count is ${count.value}`)
})

// Lifecycle
onMounted(() => fetchUser())

// Methods
function increment() { count.value++ }

// Template refs
const inputRef = ref<HTMLInputElement | null>(null)
onMounted(() => inputRef.value?.focus())
</script>

<template>
  <div>
    <p>{{ count }} × 2 = {{ doubleCount }}</p>
    <button @click="increment">+1</button>
    <input ref="inputRef" v-model="user.name" />
  </div>
</template>
```

---

## Reactivity System

**Факты:**
- Vue 3 использует **Proxy** (Vue 2 использовал `Object.defineProperty`)
- **ref()** — для примитивов (оборачивает в reactive `.value`)
- **reactive()** — для объектов (глубокая реактивность)
- **shallowRef / shallowReactive** — поверхностная реактивность

```typescript
// Vue 2 vs Vue 3 reactivity:
// Vue 2: Object.defineProperty — не видит добавление/удаление свойств
this.obj.newField = 'value'  // Не реактивно (Vue 2)

// Vue 3: Proxy — видит всё
const obj = reactive({})
obj.newField = 'value'        // Реактивно!

// shallowRef — для больших объектов без глубокой реакции
const largeData = shallowRef({ items: [] })
```

---

## Directives

```vue
<template>
  <!-- v-if / v-else-if / v-else -->
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">{{ error }}</div>
  <div v-else>{{ data }}</div>

  <!-- v-for with key -->
  <li v-for="(item, index) in items" :key="item.id">
    {{ index }}: {{ item.name }}
  </li>

  <!-- v-model (two-way binding) -->
  <input v-model="search" />
  <input v-model.number="age" />           <!-- .number modifier -->
  <input v-model.trim="name" />            <!-- .trim modifier -->

  <!-- v-bind (:) and v-on (@) -->
  <img :src="avatarUrl" :alt="user.name" />
  <button @click.stop="handleClick">Save</button>    <!-- .stop propagation -->
  <form @submit.prevent="onSubmit">        <!-- .prevent default -->
  
  <!-- v-show (CSS display toggle) -->
  <div v-show="isVisible">Shown via CSS</div>

  <!-- v-html (⚠️ XSS risk) -->
  <div v-html="rawHtml"></div>
</template>
```

---

## Components & Props

```vue
<!-- ChildComponent.vue -->
<script setup lang="ts">
// Props with types
const props = defineProps<{
  title: string
  count?: number
  items: string[]
}>()

// Emits
const emit = defineEmits<{
  (e: 'update', value: string): void
  (e: 'delete', id: number): void
}>()

// Expose (for template refs)
defineExpose({ reset, validate })
</script>

<template>
  <div>
    <h2>{{ title }}</h2>
    <slot name="header" />
    <slot :item="currentItem" />          <!-- Scoped slot -->
  </div>
</template>

<!-- ParentComponent.vue -->
<script setup>
import ChildComponent from './ChildComponent.vue'
</script>
<template>
  <ChildComponent 
    title="Hello" 
    :count="5" 
    :items="list"
    @update="handleUpdate"
  >
    <template #header>Custom Header</template>
    <template #default="{ item }">{{ item.name }}</template>
  </ChildComponent>
</template>
```

---

## Router (Vue Router 4)

```typescript
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  { path: '/', redirect: '/home' },
  { path: '/home', component: () => import('./views/Home.vue') },  // Lazy
  { 
    path: '/users/:id', 
    component: () => import('./views/UserDetail.vue'),
    props: true,     // Route params as props
    beforeEnter: (to, from) => { /* guard */ }
  },
  { path: '/:pathMatch(.*)*', component: NotFound }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

// Navigation guards
router.beforeEach((to, from) => {
  if (to.meta.requiresAuth && !isAuthenticated) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
})
```

---

## Чек-лист

- [ ] Composition API: ref, reactive, computed, watch, watchEffect
- [ ] Reactivity: Proxy-based, ref vs reactive, shallowRef
- [ ] Directives: v-if, v-for (:key), v-model (.number, .trim), v-on (.stop, .prevent)
- [ ] Components: defineProps/defineEmits, slots (named, scoped)
- [ ] Router: lazy loading, guards, props: true
- [ ] Lifecycle: onMounted, onUnmounted, onUpdated
