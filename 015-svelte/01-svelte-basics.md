# Svelte Basics

---

## Reactivity

**Факты:**
- Svelte — **compiler** (не runtime). Код компилируется в vanilla JS
- Нет Virtual DOM — reactivity через прямые DOM-операции
- Svelte 5 использует **Runes** (`$state`, `$derived`, `$effect`)

```svelte
<script>
  // Vars are reactive (Svelte 4)
  let count = 0
  let name = 'World'
  
  function increment() { count += 1 }
  
  // Reactive declarations
  $: doubled = count * 2              // Auto-recalculates
  $: console.log(`count is ${count}`) // Side effect
  
  // Svelte 5 Runes (new syntax)
  let count5 = $state(0)
  let doubled5 = $derived(count5 * 2)
  $effect(() => console.log(count5))
</script>

<h1>Hello {name}!</h1>
<p>Count: {count} (doubled: {doubled})</p>
<button on:click={increment}>+1</button>
```

**Факт:** Svelte перемещает работу **на этап компиляции** — меньше JS в runtime, меньше bundle.

**Факт:** Svelte — самый быстрый фреймворк по результатам JS Framework Benchmark (без Virtual DOM).

---

## Components

```svelte
<!-- UserCard.svelte -->
<script>
  export let name: string
  export let age: number = 0  // Default value
  
  // Events
  import { createEventDispatcher } from 'svelte'
  const dispatch = createEventDispatcher<{ update: { name: string; age: number } }>()
  
  function handleClick() {
    dispatch('update', { name, age: age + 1 })
  }
</script>

<div class="card">
  <h2>{name}</h2>
  <p>Age: {age}</p>
  <button on:click={handleClick}>Birthday!</button>
</div>

<style>
  .card { border: 1px solid #ddd; padding: 16px; border-radius: 8px; }
  h2 { color: #333; }
</style>

<!-- Usage -->
<script>
  import UserCard from './UserCard.svelte'
  
  function handleUpdate(event: CustomEvent<{ name: string; age: number }>) {
    console.log(event.detail)
  }
</script>

<UserCard name="John" age={30} on:update={handleUpdate} />
```

---

## Reactivity Fundamentals

```svelte
<script>
  // Svelte 4 reactivity — assignment triggers re-render
  
  let items = ['a', 'b', 'c']
  
  // ❌ Mutation doesn't trigger
  items.push('d')
  
  // ✅ Assignment triggers
  items = [...items, 'd']
  // or
  items[items.length] = 'd'  // Svelte detects array assignment
  
  // Same for objects
  let user = { name: 'John', age: 30 }
  user.age = 31  // ✅ Triggers (Svelte detects property assignment)
  
  // Reactive statements ($:)
  $: valid = name.length > 0 && age > 0
  $: {
    console.log(`User: ${name}, Age: ${age}`)
  }
  
  // Reactive with dependencies
  $: fullName = `${firstName} ${lastName}`
</script>
```

---

## Stores

```svelte
<!-- store.ts -->
import { writable, derived, readable } from 'svelte/store'

export const count = writable(0)
export const doubled = derived(count, $count => $count * 2)
export const time = readable(new Date(), (set) => {
  const interval = setInterval(() => set(new Date()), 1000)
  return () => clearInterval(interval)
})

// Custom store
function createTodos() {
  const { subscribe, update, set } = writable<Todo[]>([])
  
  return {
    subscribe,
    add: (todo: Todo) => update(todos => [...todos, todo]),
    toggle: (id: number) => update(todos => todos.map(t =>
      t.id === id ? { ...t, done: !t.done } : t
    )),
    reset: () => set([])
  }
}

export const todos = createTodos()
```

```svelte
<!-- Component -->
<script>
  import { count, doubled, todos } from './store'
  import { onDestroy } from 'svelte'
  
  // Auto-subscribe with $
  // Unsubscribe auto on destroy
</script>

<p>Count: {$count} (doubled: {$doubled})</p>
<button on:click={() => $count++}>+1</button>

{#each $todos as todo (todo.id)}
  <div>
    <input type="checkbox" checked={todo.done} on:change={() => todos.toggle(todo.id)} />
    {todo.text}
  </div>
{/each}
```

---

## Lifecycle & Events

```svelte
<script>
  import { onMount, onDestroy, beforeUpdate, afterUpdate, tick } from 'svelte'
  
  let element: HTMLDivElement
  
  onMount(() => {
    console.log('Mounted', element)
    return () => console.log('Cleanup')  // onDestroy alternative
  })
  
  onDestroy(() => console.log('Destroyed'))
  beforeUpdate(() => console.log('Before update'))
  afterUpdate(() => console.log('After update'))
  
  // tick() — resolve after pending DOM updates
  async function update() {
    count++
    await tick()
    // DOM is now updated
    element.scrollTop = element.scrollHeight
  }
  
  // Event forwarding
  function handleInnerEvent() {
    // Create event — will be forwarded to parent
  }
</script>

<div bind:this={element}>
  <!-- Event modifiers: preventDefault, stopPropagation, once, capture -->
  <button on:click|preventDefault={handleSubmit}>Submit</button>
  <button on:click|once={handleOnce}>Once</button>
</div>
```

---

## Чек-лист

- [ ] Compiler-based (no Virtual DOM) — Svelte компилирует в vanilla JS
- [ ] Reactivity: assignment triggers, `$:` reactive statements
- [ ] Components: `export let` props, `createEventDispatcher`
- [ ] Stores: writable, derived, readable, custom stores, `$` auto-subscribe
- [ ] Lifecycle: onMount, onDestroy, beforeUpdate, afterUpdate, tick()
- [ ] Events: on:click, modifiers (preventDefault, stopPropagation, once)
- [ ] Bindings: bind:value, bind:this, bind:group
