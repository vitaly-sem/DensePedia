# Vue.js Version History

---

| Версия | Дата | Ключевые изменения |
|---|---|---|
| **Vue 2.0** | Sep 2016 | Virtual DOM, render functions, SSR, Vue CLI |
| **Vue 2.2** | Feb 2017 | `provide/inject`, new lifecycle hooks |
| **Vue 2.3** | Apr 2017 | Async component improvements, SSR |
| **Vue 2.4** | Jul 2017 | `$attrs`, `$listeners`, SSR improvements |
| **Vue 2.5** | Oct 2017 | TypeScript improvements, `errorCaptured` |
| **Vue 2.6** | Feb 2019 | Slots syntax (`v-slot`), error tracking, `observed` arrays |
| **Vue 2.7** | Jul 2022 | Composition API backport, `<script setup>`, `defineComponent` |
| **Vue 3.0 "One Piece"** | Sep 2020 | **Composition API**, Proxy reactivity, TypeScript, Teleport, Fragments, Suspense, Tree-shaking |
| **Vue 3.1** | Jun 2021 | Migration build |
| **Vue 3.2** | Aug 2021 | **`<script setup>`** (stable), `defineEmits`, `defineProps`, `v-memo`, `effectScope` |
| **Vue 3.3** | May 2023 | **Generics in `<script setup>`**, `defineModel`, `defineOptions` |
| **Vue 3.4** | Dec 2023 | **Parsing improvements**, reactivity perf, better TypeScript |
| **Vue 3.5** | Sep 2024 | **Reactivity improvements**, `useId()`, `watch` pause/resume |
| **Vue 3.6+** | 2025 | **Vapor Mode** (no Virtual DOM, compile-time reactivity) |

**Факт:** Vue 2 EOL — Dec 31, 2022. Vue 2.7 — последний минор.

**Факт:** Vue 3 **не является** breaking change для большинства API (Options API сохранился).

### Vue 2 → Vue 3 — ключевые различия

| Аспект | Vue 2 | Vue 3 |
|---|---|---|
| Reactivity | `Object.defineProperty` | `Proxy` (новые свойства, массивы) |
| API | Options API | Options + Composition API |
| Multiple roots | ❌ (один root) | ✅ (fragments) |
| TypeScript | Сложно | ✅ First-class |
| Teleport | ❌ (Portal Vue) | ✅ `<Teleport>` |
| Suspense | ❌ | ✅ `<Suspense>` |
| Tree-shaking | ❌ | ✅ (~50% smaller bundle) |
| IE11 | ✅ | ❌ |
