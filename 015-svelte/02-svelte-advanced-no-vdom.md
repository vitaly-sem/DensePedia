# Svelte Advanced & No Virtual DOM

---

## Svelte Compiler — почему нет Virtual DOM

**Факты:**
- Svelte — **compiler**, не runtime framework
- React/Vue: Virtual DOM diff → apply changes (runtime overhead)
- Svelte: **compile-time analysis** → direct DOM updates (minimal code)

```
React:
  JSX → Virtual DOM → Diff → Real DOM

Svelte:
  Template → Compiler analysis → Direct DOM update code
```

**Факт:** Svelte bundle в 2-5x меньше React/Vue (нет runtime Virtual DOM).

**Факт:** Svelte обновляет DOM **напрямую** при изменении переменной — без diff.

```javascript
// React — нужно runtime (~50KB)
// React.createElement → Virtual DOM → reconciliation

// Svelte — скомпилировано в прямые DOM операции
function update_count(value) {
  count = value
  text_node.data = value  // Direct DOM update!
  button.disabled = value <= 0
}
```

### Solid.js — ещё один подход без VDOM

**Факт:** Solid.js похож на Svelte (нет VDOM), но использует **fine-grained reactivity** через Signals:

```typescript
import { createSignal, createEffect } from 'solid-js'

const [count, setCount] = createSignal(0)
const doubled = () => count() * 2

createEffect(() => {
  console.log(`Count is ${count()}`)
})

return <button onClick={() => setCount(c => c + 1)}>{doubled()}</button>
```

**Сравнение No-VDOM фреймворков:**

| Framework | Approach | Bundle | Technique |
|---|---|---|---|
| **Svelte** | Compile-time | ~5KB | Compiler generates imperative DOM ops |
| **Solid.js** | Fine-grained | ~8KB | Signals + DOM cloning |
| **Inferno** | VDOM (optimized) | ~8KB | Very fast VDOM diff |
| **Preact** | Minimal VDOM | ~3KB | Tiny React alternative |

---

## SvelteKit — SSR Framework

**Факты:**
- SvelteKit = Next.js для Svelte (SSR, SSG, SPA)
- **File-based routing**
- **Load functions** — data fetching (server-side)

```svelte
<!-- src/routes/+page.svelte — root page -->
<script>
  import { page } from '$app/stores'
  export let data  // From load function
</script>

<h1>{data.title}</h1>
```

```typescript
// src/routes/+page.server.ts — server load
import type { PageServerLoad } from './$types'

export const load: PageServerLoad = async ({ params, fetch, cookies }) => {
  const res = await fetch(`/api/posts/${params.slug}`)
  const post = await res.json()
  
  return { post }
}

// src/routes/+page.ts — universal load (server + client)
export const load: PageLoad = async ({ fetch }) => {
  const res = await fetch('/api/posts')
  return { posts: await res.json() }
}
```

```svelte
<!-- SvelteKit routing -->
<!-- src/routes/+page.svelte        → / -->
<!-- src/routes/about/+page.svelte   → /about -->
<!-- src/routes/blog/[slug]/+page.svelte → /blog/:slug -->
<!-- src/routes/blog/[...catch]/+page.svelte → /blog/a/b/c -->
```

### Forms & Actions

```typescript
// src/routes/login/+page.server.ts
export const actions = {
  default: async ({ request, cookies }) => {
    const data = await request.formData()
    const email = data.get('email')
    const password = data.get('password')
    
    const token = await login(email, password)
    cookies.set('token', token, { path: '/' })
    
    return { success: true }
  }
}
```

---

## Transitions & Animations

```svelte
<script>
  import { fade, slide, scale, fly, blur } from 'svelte/transition'
  import { crossfade } from 'svelte/transition'
  let visible = false
</script>

<button on:click={() => visible = !visible}>Toggle</button>

{#if visible}
  <div transition:fade={{ duration: 300 }}>
    Fade
  </div>
  
  <div transition:slide={{ duration: 200, delay: 100 }}>
    Slide
  </div>
  
  <!-- In/Out separate -->
  <div in:fly={{ x: 200 }} out:fly={{ x: -200 }}>
    Fly
  </div>
  
  <!-- Keyed list animation -->
  {#each items as item (item.id)}
    <div animate:flip>{item.name}</div>
  {/each}
{/if}
```

---

## Actions

```svelte
<!-- Actions = Svelte's custom directives -->
<script>
  function clickOutside(node: HTMLElement, onOutsideClick: () => void) {
    function handleClick(event: MouseEvent) {
      if (!node.contains(event.target as Node)) {
        onOutsideClick()
      }
    }
    
    document.addEventListener('click', handleClick)
    
    return {
      destroy() {
        document.removeEventListener('click', handleClick)
      }
    }
  }
  
  let showDropdown = false
</script>

<div use:clickOutside={() => showDropdown = false}>
  <button on:click={() => showDropdown = !showDropdown}>Menu</button>
  {#if showDropdown}
    <div class="dropdown">...</div>
  {/if}
</div>
```

---

## Чек-лист

- [ ] No Virtual DOM: compile-time analysis, direct DOM ops, smaller bundle
- [ ] Solid.js: fine-grained reactivity (Signals)
- [ ] SvelteKit: file routing, load functions, SSR/SSG
- [ ] SvelteKit Forms: actions, form data, cookie management
- [ ] Transitions: fade, slide, fly, scale, crossfade, animate:flip
- [ ] Actions: use:actionName, lifecycle (destroy, update)
