# React Advanced

---

## Performance Optimization

```tsx
import { memo, useMemo, useCallback } from 'react'

// 1. React.memo — skip re-render if props unchanged (shallow comparison)
const ExpensiveList = memo(function ExpensiveList({ items }: { items: Item[] }) {
  return items.map(item => <ExpensiveItem key={item.id} item={item} />)
})

// 2. useMemo — memoize computed value
function Dashboard({ transactions }: { transactions: Transaction[] }) {
  const totals = useMemo(() => ({
    revenue: transactions.filter(t => t.type === 'revenue').reduce(sum, 0),
    expenses: transactions.filter(t => t.type === 'expense').reduce(sum, 0)
  }), [transactions])  // Recalc only when transactions change
  
  return <div>Revenue: {totals.revenue}, Expenses: {totals.expenses}</div>
}

// 3. useCallback — memoize function (stable reference)
function UserList({ onSelect }: { onSelect: (id: string) => void }) {
  const handleClick = useCallback((id: string) => {
    onSelect(id)
  }, [onSelect])
  
  return <List items={users} onItemClick={handleClick} />
}

// 4. useTransition — non-urgent updates (React 18)
function SearchPage() {
  const [query, setQuery] = useState('')
  const [isPending, startTransition] = useTransition()
  
  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    // Urgent: update input
    setQuery(e.target.value)
    
    // Non-urgent: filter results
    startTransition(() => {
      setFilteredResults(filter(e.target.value))
    })
  }
  
  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <Results items={filteredResults} />}
    </>
  )
}
```

**Факт:** `memo` — не бесплатно. Используйте только если компонент реально перерисовывается часто с теми же props.

**Факт:** `useCallback(fn, deps)` — идентичен `useMemo(() => fn, deps)`.

---

## Custom Hooks

```tsx
// useDebounce
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value)
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])
  
  return debouncedValue
}

// useLocalStorage
function useLocalStorage<T>(key: string, initial: T): [T, (value: T) => void] {
  const [stored, setStored] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key)
      return item ? JSON.parse(item) : initial
    } catch { return initial }
  })
  
  const setValue = useCallback((value: T) => {
    setStored(value)
    localStorage.setItem(key, JSON.stringify(value))
  }, [key])
  
  return [stored, setValue]
}

// useIntersectionObserver (infinite scroll)
function useInfiniteScroll(callback: () => void) {
  const observer = useRef<IntersectionObserver | null>(null)
  
  const lastElementRef = useCallback((node: HTMLElement | null) => {
    if (observer.current) observer.current.disconnect()
    
    observer.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) callback()
    })
    
    if (node) observer.current.observe(node)
  }, [callback])
  
  return lastElementRef
}
```

---

## useReducer — complex state logic

```tsx
type Action = 
  | { type: 'ADD_TODO'; payload: Todo }
  | { type: 'TOGGLE_TODO'; payload: number }
  | { type: 'DELETE_TODO'; payload: number }
  | { type: 'SET_FILTER'; payload: FilterType }

interface State {
  todos: Todo[]
  filter: FilterType
}

function todoReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'ADD_TODO':
      return { ...state, todos: [...state.todos, action.payload] }
    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(t =>
          t.id === action.payload ? { ...t, done: !t.done } : t
        )
      }
    case 'DELETE_TODO':
      return {
        ...state,
        todos: state.todos.filter(t => t.id !== action.payload)
      }
    default:
      return state
  }
}

function TodoApp() {
  const [state, dispatch] = useReducer(todoReducer, { todos: [], filter: 'all' })
  // dispatch({ type: 'ADD_TODO', payload: newTodo })
}
```

---

## Portals & Suspense

```tsx
// Portal — render outside parent DOM hierarchy
function Modal({ children }: { children: ReactNode }) {
  return createPortal(
    <div className="modal-overlay">{children}</div>,
    document.body  // Render to <body>, not parent
  )
}

// Suspense — loading state for async operations (code splitting, data fetching)
function App() {
  return (
    <Suspense fallback={<GlobalSpinner />}>
      <Routes>
        <Route path="/dashboard" element={
          <Suspense fallback={<DashboardSkeleton />}>
            <Dashboard />
          </Suspense>
        } />
      </Routes>
    </Suspense>
  )
}

// ErrorBoundary — catch render errors (still class component)
class ErrorBoundary extends React.Component<{ fallback: ReactNode }> {
  state = { hasError: false }
  
  static getDerivedStateFromError() { return { hasError: true } }
  
  componentDidCatch(error: Error, info: ErrorInfo) {
    logError(error, info)
  }
  
  render() {
    return this.state.hasError ? this.props.fallback : this.props.children
  }
}
```

---

## React DevTools — Profiler

```tsx
import { Profiler } from 'react'

function onRender(
  id: string,                    // Profiler id
  phase: 'mount' | 'update',    // Mount or re-render
  actualDuration: number,        // Time spent rendering
  baseDuration: number,          // Estimated time without memo
  startTime: number,
  commitTime: number,
  interactions: Set<Interaction>
) {
  // Log to analytics
  console.log(`${id} ${phase}: ${actualDuration}ms`)
}

<Profiler id="UserList" onRender={onRender}>
  <UserList />
</Profiler>
```

---

## Чек-лист

- [ ] Performance: memo, useMemo, useCallback, useTransition
- [ ] Custom Hooks: useDebounce, useLocalStorage, useInfiniteScroll
- [ ] useReducer: complex state, discriminated unions
- [ ] Portals: createPortal (modals, tooltips)
- [ ] Suspense: code splitting, data fetching (React 18)
- [ ] ErrorBoundary: catch render errors
- [ ] Profiler: измерение производительности
