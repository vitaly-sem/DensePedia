# React Basics

---

## Components

```tsx
// Functional Component (современный)
interface UserProps {
  name: string
  age: number
  onUpdate: (user: User) => void
}

function UserCard({ name, age, onUpdate }: UserProps) {
  return (
    <div className="user-card">
      <h2>{name}</h2>
      <p>Age: {age}</p>
      <button onClick={() => onUpdate({ name, age: age + 1 })}>
        Birthday!
      </button>
    </div>
  )
}

// Class Component (legacy)
class Welcome extends React.Component<{ name: string }> {
  render() {
    return <h1>Hello, {this.props.name}</h1>
  }
}
```

**Факт:** Functional components + Hooks — современный стандарт. Class components: legacy.

---

## JSX

```tsx
// JSX = JavaScript XML, компилируется в React.createElement
// React 17+: new JSX transform (не требует import React)

// Conditional rendering
{isLoggedIn ? <UserPanel /> : <LoginButton />}
{loading && <Spinner />}

// Lists
{items.map((item, index) => (
  <li key={item.id}>  {/* key обязателен для списков */}
    {index}: {item.name}
  </li>
))}

// Event handling
<button onClick={(e) => handleClick(e)}>Click</button>
<form onSubmit={handleSubmit}>...</form>

// Styles
<div style={{ color: 'red', fontSize: 14 }}>Inline styles</div>
<div className={`card ${active ? 'active' : ''}`}>Classes</div>
```

**Факт:** `key` prop — критичен для производительности списков. Используйте уникальный ID, не index.

---

## Hooks — useState, useEffect

```tsx
import { useState, useEffect } from 'react'

function UserList() {
  // State
  const [users, setUsers] = useState<User[]>([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  
  // Side effects
  useEffect(() => {
    let cancelled = false
    
    async function load() {
      try {
        setLoading(true)
        const data = await fetchUsers()
        if (!cancelled) setUsers(data)
      } catch (err) {
        if (!cancelled) setError(err.message)
      } finally {
        if (!cancelled) setLoading(false)
      }
    }
    
    load()
    
    return () => { cancelled = true }  // Cleanup
  }, [])  // Empty deps = on mount
  
  if (loading) return <Spinner />
  if (error) return <Error message={error} />
  
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  )
}
```

**Факт:** Cleanup function в useEffect (return () => {}) — предотвращает обновление unmounted компонента.

**Факт:** Strict Mode (React 18+) — double-invokes useEffect в dev для detection of bugs.

---

## Hooks — useContext

```tsx
// Create context (в отдельном файле)
interface AuthContext {
  user: User | null
  login: (email: string, password: string) => Promise<void>
  logout: () => void
}

const AuthContext = createContext<AuthContext | undefined>(undefined)

// Provider
function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  
  const login = async (email: string, password: string) => {
    const user = await api.login(email, password)
    setUser(user)
  }
  
  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  )
}

// Consumer hook
function useAuth() {
  const context = useContext(AuthContext)
  if (!context) throw new Error('useAuth must be inside AuthProvider')
  return context
}

// Usage in component
function Profile() {
  const { user, logout } = useAuth()
  return <div>Welcome, {user?.name} <button onClick={logout}>Logout</button></div>
}
```

---

## useRef

```tsx
function AutoFocusInput() {
  const inputRef = useRef<HTMLInputElement>(null)
  
  useEffect(() => {
    inputRef.current?.focus()
  }, [])
  
  // Ref also holds mutable values (no re-render)
  const renderCount = useRef(0)
  renderCount.current++
  
  return <input ref={inputRef} />
}
```

---

## Чек-лист

- [ ] Components: functional (modern) vs class (legacy)
- [ ] Props: TypeScript interface, destructuring
- [ ] State: useState (immutable update)
- [ ] Effects: useEffect (deps, cleanup, mount/unmount)
- [ ] Context: createContext, Provider, custom hook for context
- [ ] Refs: useRef (DOM access, mutable values)
- [ ] Lists: key prop (ID, not index)
- [ ] Events: onClick, onSubmit, synthetic events
