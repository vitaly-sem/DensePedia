# React Version History

---

| Версия | Дата | Ключевые изменения |
|---|---|---|
| **React 0.14** | Oct 2015 | Split: `react` + `react-dom`. Stateless functional components |
| **React 15** | Apr 2016 | SVG support, `document.createElement` |
| **React 16.0** | Sep 2017 | **Fiber** (new reconciliation engine), Fragments, Error Boundaries, Portals, `render()` return array |
| **React 16.3** | Mar 2018 | **Context API** (stable), `createRef`, `forwardRef`, lifecycle deprecations (componentWillMount) |
| **React 16.4** | May 2018 | Pointer Events |
| **React 16.6** | Oct 2018 | **React.lazy** (code splitting), `Suspense`, `memo`, `lazy` |
| **React 16.8** | Feb 2019 | **Hooks** (BREAKTHROUGH): `useState`, `useEffect`, `useContext`, `useReducer`, `useCallback`, `useMemo`, `useRef` |
| **React 17** | Oct 2020 | **No new features** — gradual upgrade path, new JSX transform, event delegation to root |
| **React 18** | Mar 2022 | **Concurrent features**: `useTransition`, `useDeferredValue`, `Suspense` for data fetching, Automatic batching, SSR `hydrateRoot`, `useId`, `useSyncExternalStore` |
| **React 19** | Apr 2024 | **Actions** (`useActionState`, `<form action>`), **Server Components**, `use()` hook, `useOptimistic`, Document Metadata, Asset Loading |

**Факт:** React 17 — upgrade release (без новых фич, но с breaking changes для библиотек).

**Факт:** React 19 приносит Server Components (RSC) — компоненты, выполняющиеся на сервере, без JS бандла.

### Key Concepts Timeline

```
2013: Initial release (v0.3)
2015: React Native
2017: Fiber, Fragments, Portals, Error Boundaries
2018: Context API (stable), React.lazy, Suspense
2019: Hooks (GAMECHANGER)
2020: React 17 Upgrade Release
2022: Concurrent Features (React 18)
2024: Server Components, Actions (React 19)
```

### Breaking Changes — важные

| From | To | Что делать |
|---|---|---|
| `componentWillMount` | `UNSAFE_componentWillMount` (16.3+) | Migrate to `componentDidMount` |
| `componentWillReceiveProps` | `getDerivedStateFromProps` | Or use hooks (`useEffect`) |
| `componentWillUpdate` | `getSnapshotBeforeUpdate` | Or hooks |
| Legacy Context (childContextTypes) | New Context API | `createContext` + `useContext` |
| `ReactDOM.render` | `createRoot` (18+) | Для Concurrent Features |
| JSX transform (old) | New JSX transform (17+) | `"jsx": "react-jsx"` в tsconfig |

**Факт:** Facebook/React team recommends: **Hooks for new code, keep class components for existing complex logic.**
