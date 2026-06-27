# React Practical Tips

---

## Performance

```tsx
// 1. React.memo — skip re-render if props unchanged
export const UserCard = memo(function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>
})

// 2. useMemo — expensive computations
const sortedUsers = useMemo(() => {
  return [...users].sort((a, b) => a.name.localeCompare(b.name))
}, [users])

// 3. useCallback — stable function references
const handleClick = useCallback((id: string) => {
  setSelectedId(id)
}, [])

// 4. Virtual list for large lists
import { FixedSizeList } from 'react-window'
<FixedSizeList height={400} itemCount={10000} itemSize={50}>
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</FixedSizeList>

// 5. Code splitting with React.lazy
const HeavyComponent = lazy(() => import('./HeavyComponent'))
<Suspense fallback={<Spinner />}>
  <HeavyComponent />
</Suspense>
```

---

## Hooks Best Practices

```tsx
// 1. Custom hook for complex logic
function useUser(id: string) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  
  useEffect(() => {
    let cancelled = false
    setLoading(true)
    
    fetchUser(id).then(data => {
      if (!cancelled) {
        setUser(data)
        setLoading(false)
      }
    })
    
    return () => { cancelled = true }
  }, [id])
  
  return { user, loading }
}

// 2. useEffect dependencies — explicit
useEffect(() => {
  document.title = `User: ${user.name}`
}, [user.name])  // NOT [user] — too broad

// 3. Cleanup for subscriptions
useEffect(() => {
  const sub = service.subscribe(handleEvent)
  return () => sub.unsubscribe()
}, [])

// 4. useRef for previous value
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>()
  useEffect(() => { ref.current = value }, [value])
  return ref.current
}

// 5. useState callback for derived state
const [count, setCount] = useState(0)
setCount(prev => prev + 1)  // safe when batching
```

---

## State Management

```tsx
// 1. Context + useReducer for local state
const StateContext = createContext<State>(initialState)
const DispatchContext = createContext<React.Dispatch<Action>>(() => {})

function App() {
  const [state, dispatch] = useReducer(reducer, initialState)
  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        <Main />
      </DispatchContext.Provider>
    </StateContext.Provider>
  )
}

// 2. Zustand — simpler than Redux
import { create } from 'zustand'
const useStore = create(set => ({
  users: [],
  loading: false,
  fetchUsers: async () => {
    set({ loading: true })
    const users = await api.getUsers()
    set({ users, loading: false })
  }
}))

// 3. TanStack Query — server state
import { useQuery } from '@tanstack/react-query'
const { data, isLoading } = useQuery({
  queryKey: ['users', id],
  queryFn: () => fetchUser(id),
  staleTime: 5 * 60 * 1000,  // 5 min cache
  retry: 2
})
```

---

## Error Handling

```tsx
// 1. Error Boundary (wrap critical sections)
<ErrorBoundary fallback={<ErrorPage />}>
  <UserDashboard />
</ErrorBoundary>

// 2. React Query error handling
const { error } = useQuery({ queryKey: ['data'], queryFn: fetchData })
if (error) return <Error message={error.message} />

// 3. Async error in event handlers
async function handleSubmit() {
  try {
    await saveData(formData)
    toast.success('Saved!')
  } catch (err) {
    toast.error(err instanceof Error ? err.message : 'Failed')
  }
}

// 4. Global error boundary (React 18+)
// createRoot with error handling
```

---

## Forms

```tsx
// 1. React Hook Form — performant forms
import { useForm } from 'react-hook-form'
const { register, handleSubmit, formState: { errors } } = useForm<User>()

<form onSubmit={handleSubmit(save)}>
  <input {...register('name', { required: true })} />
  {errors.name && <span>Required</span>}
  <button type="submit">Save</button>
</form>

// 2. Controlled vs Uncontrolled
// Uncontrolled (ref, faster) — for simple inputs
// Controlled (state) — when validation, dynamic fields

// 3. Debounced search
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])
  return debounced
}
```

---

## Other Tips

```tsx
// 1. StrictMode — detect side effects (dev only)
<React.StrictMode><App /></React.StrictMode>

// 2. Keys — stable, unique, not index
{items.map(item => <Item key={item.id} />)}

// 3. Conditional rendering
{isLoading ? <Spinner /> : <Data />}
{data && <Data data={data} />}

// 4. Props spreading (careful!)
<Child {...props} />
// Better: explicit props

// 5. TypeScript — strict mode
interface Props {
  onSelect: (id: string) => void
  children: React.ReactNode
}
```
