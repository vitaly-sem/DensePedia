# Svelte Practical Tips

---

## Performance

```svelte
<script>
  // 1. Use $derived instead of $: for computed values (Svelte 5)
  let items = $state([])
  let total = $derived(items.reduce((s, i) => s + i.price, 0))
  
  // 2. Avoid reactive declarations for static data
  let staticData = DATA  // If DATA never changes, don't use $:
  
  // 3. {#key} — recreate component when value changes
  {#key userId}
    <UserProfile {userId} />
  {/key}
  
  // 4. Lazy load components
  import('./HeavyComponent.svelte').then(module => {
    heavyComponent = module.default
  })
  
  // 5. Use tick() for batch updates
  async function batchUpdate() {
    count = 10
    name = 'John'
    await tick()  // One DOM update for both changes
  }
</script>
```

---

## State Management Patterns

```svelte
<script>
  // 1. Reactive object — direct property assignment works
  let form = { name: '', email: '' }
  function updateField(field: string, value: string) {
    form[field] = value  // ✅ Svelte detects property assignment
  }
  
  // 2. Store pattern with modules
  // stores/cart.ts
  function createCartStore() {
    const { subscribe, update } = writable<CartItem[]>([])
    
    return {
      subscribe,
      addItem: (item: CartItem) => update(items => [...items, item]),
      removeItem: (id: number) => update(items => items.filter(i => i.id !== id)),
      clear: () => update(() => [])
    }
  }
  export const cart = createCartStore()
  
  // 3. Context for component trees
  import { setContext, getContext } from 'svelte'
  const key = Symbol('user')
  setContext(key, user)  // Parent
  const user = getContext(key)  // Child (any depth)
</script>
```

---

## Forms & Bindings

```svelte
<script>
  let name = ''
  let email = ''
  let agree = false
  let selectedOption = ''
  let selectedOptions: string[] = []
  
  function handleSubmit() {
    if (!name) return
    // Submit
  }
</script>

<form on:submit|preventDefault={handleSubmit}>
  <input bind:value={name} placeholder="Name" />
  <input type="email" bind:value={email} />
  
  <input type="checkbox" bind:checked={agree} /> I agree
  
  <!-- Select -->
  <select bind:value={selectedOption}>
    <option value="a">Option A</option>
    <option value="b">Option B</option>
  </select>
  
  <!-- Grouped checkboxes -->
  {#each options as option}
    <label>
      <input type="checkbox" bind:group={selectedOptions} value={option} />
      {option}
    </label>
  {/each}
  
  <button type="submit" disabled={!name}>Submit</button>
</form>
```

---

## Animations

```svelte
<script>
  import { fade, fly, slide } from 'svelte/transition'
  import { flip } from 'svelte/animate'
  
  let items = ['a', 'b', 'c']
  
  function shuffle() {
    items = items.sort(() => Math.random() - 0.5)
  }
</script>

<!-- List with FLIP animation -->
{#each items as item (item)}
  <div animate:flip>{{ item }}</div>
{/each}

<button on:click={shuffle}>Shuffle</button>

<!-- Crossfade — shared element transition -->
<script>
  import { crossfade } from 'svelte/transition'
  const [send, receive] = crossfade()
</script>

{#if inList}
  <div transition:send={{ key: item.id }}>
    {item.name}
  </div>
{:else}
  <div transition:receive={{ key: item.id }}>
    {item.name}
  </div>
{/if}
```

---

## Other Tips

```svelte
<script>
  // 1. Reactive destructuring
  let user = { name: 'John', age: 30 }
  $: ({ name, age } = user)  // Reactive destructuring
  
  // 2. Class shorthand
  let active = true
</script>

<!-- instead of class:active={active} -->
<div class:active>Active div</div>

<!-- Conditional classes -->
<div class:active class:highlight={isHighlighted}>Class binding</div>

<script>
  // 3. Spread props
  let userData = { name: 'John', email: 'john@test.com' }
</script>

<UserProfile {...userData} />

<script>
  // 4. Module scripts — run once, not per instance
</script>
<svelte:head>
  <title>My App</title>
</svelte:head>

<svelte:window on:keydown={handleKeydown} />
<svelte:body on:mouseenter={handleEnter} />
```

---

## Чек-лист

- [ ] $derived (Svelte 5) vs $: reactive statements
- [ ] {#key} для recreation
- [ ] Context API для component trees
- [ ] bind:value, bind:group, bind:checked
- [ ] Animations: flip, crossfade, custom transitions
- [ ] Class shorthand: class:active={condition}
- [ ] Module scripts: <script context="module">
- [ ] <svelte:head>, <svelte:window>, <svelte:body>
