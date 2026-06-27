# DensePedia — База знаний Senior-инженера

> **Привет, коллега! 👋**
>
> Ты держишь в руках (ну, или открыл в браузере) **DensePedia** —
> место, где C# встречается с древнегреческой философией, Kubernetes спорит с
> психологией, а DevOps-инженер внезапно узнаёт, что такое эпистемология.
>
> Здесь собрано **всё, что нужно Senior-инженеру**: от внутренностей GC и паттернов
> проектирования до софт-скиллов, истории мира и рецепта правильного завтрака.
> Потому что хороший программист — это не только код, но и широкий кругозор.
>
> **41 раздел, 200+ статей.** Читай, удивляйся, становись лучше. 🚀

---

## 📖 Удобный просмотрщик

Для комфортного чтения используйте [`index.html`](index.html) — одностраничное веб-приложение со всеми статьями:

- 🌓 **Светлая / тёмная / системная тема**
- 🎨 **8 цветовых схем** для чтения (белый, кремовый, сепия, зелёный, серый, тёмный, синий, чёрный)
- 🔤 **Выбор шрифта** (Serif, Sans, Mono, Book) и **размер текста** (75%–150%)
- 🔍 **Поиск** по названиям статей и разделов
- 📱 **Адаптивная вёрстка** для мобильных устройств
- 💾 **Запоминает** последнюю открытую статью, тему и настройки
- 🧮 **Поддержка математических формул** (KaTeX) и **подсветка кода** (Prism.js)
- 🔗 **Навигация с browser history** — прямые ссылки на статьи через `#путь/к/файлу.md`

```bash
# Запуск (требуется любой HTTP-сервер):
cd DensePedia
python3 -m http.server 8080
# Открыть http://localhost:8080
```

---

## 🎯 О проекте

DensePedia — сборник статей для подготовки к **техническим интервью** и расширения кругозора.
Материалы разделены на 4 тематические группы; каждая секция имеет формат `NXX`,
где `N` — номер группы, `XX` — номер внутри группы.

| Группа | Секции | О чём |
|--------|--------|-------|
| **💻 Программирование** | 001–018 | .NET, Web, БД, архитектура |
| **🚀 Развитие и инфраструктура** | 101–107 | Стартапы, здоровье, финансы, DevOps, софт-скиллы |
| **🧠 Наука и AI** | 201–204 | Наука XX-XXI века, AI/ML |
| **📚 Гуманитарные науки** | 301–312 | История, литература, мифы, религии, психология, философия |

> ⬇️ **Кликните на группу ниже**, чтобы перейти к нужным разделам.

---

# 🗂️ Навигация по группам

<details>
<summary><strong>💻 Программирование</strong> (секции 001–018)</summary>

| Секция | Статей | О чём |
|--------|--------|-------|
| 🔒 [001 — Безопасность](001-security/README.md) | 5 | OWASP, JWT, OAuth, AES, PBKDF2, CORS, mTLS |
| 🖥️ [002 — WPF (Продвинутые вопросы)](002-wpf-advanced/README.md) | 9 | VisualTree, DP, RoutedEvents, Binding, MVVM |
| 🧵 [003 — Многопоточность](003-multithreading/README.md) | 8 | async/await, Task, lock-free, Dataflow, Memory Model |
| 🛡️ [004 — Защита данных и DLP](004-dlp/README.md) | 6 | Классификация, TDE, шифрование, DLP, аудит, защита данных |
| 🗄️ [005 — Коллекции и алгоритмы](005-collections-algorithms/README.md) | 5 | Dictionary, Big O, LINQ, Span, Memory |
| 🏗️ [006 — Шаблоны проектирования](006-design-patterns/README.md) | 6 | SOLID, GoF (12), CQRS, Saga, Clean Architecture |
| ⚡ [007 — Оптимизация и тестирование](007-optimization-testing/README.md) | 6 | BenchmarkDotNet, GC, SIMD, xUnit, TestContainers |
| 🧠 [008 — Advanced GC и Memory](008-gc-memory/README.md) | 5 | Mark & Compact, SOS, LOH, NUMA, HeapHardLimit |
| 🔐 [009 — Authorization Libraries](009-auth-libraries/README.md) | 1 | IdentityServer, Entra ID, Auth0, OPA |
| 🅰️ [010 — Angular](010-angular/README.md) | 5 | Components, NgRx, Signals, SSR |
| 💚 [011 — Vue.js](011-vuejs/README.md) | 5 | Composition API, Pinia, Nuxt.js |
| ⚛️ [012 — React](012-react/README.md) | 5 | Hooks, memo, Next.js, Suspense |
| 🌐 [013 — HTML5 & CSS3](013-html5-css3/README.md) | 8 | HTML5, CSS3, Tailwind, Bootstrap, Material, PWA, анимации |
| 🎮 [014 — Game Engines](014-game-engines/README.md) | 1 | Three.js, Babylon, Phaser, Unity |
| 🟠 [015 — Svelte](015-svelte/README.md) | 3 | Compiler, No VDOM, SvelteKit |
| 🤖 [016 — Code Review & AI Coding](016-modern-code-review/README.md) | 1 | Copilot, Agentic Workflows |
| 🗄️ [017 — Базы данных](017-databases/README.md) | 3 | SQL, NoSQL, NewSQL, проектирование, шардинг, репликация |
| 🏗️ [018 — Системный дизайн и архитектура](018-system-design-architecture/README.md) | 3 | Load balancing, caching, микросервисы, масштабирование, CAP |

</details>

<details>
<summary><strong>🚀 Развитие и инфраструктура</strong> (секции 101–107)</summary>

| Секция | Статей | О чём |
|--------|--------|-------|
| 🎨 [101 — Diffusion Models](101-diffusion-models/README.md) | 1 | SD, DALL-E, Midjourney, Flux, Sora |
| 🚀 [102 — Start-up Theory](102-startup-theory/README.md) | 4 | PMF, Lean Startup, успех/провал, ресурсы |
| 🥗 [103 — Здоровый образ жизни](103-healthy-lifestyle/README.md) | 5 | Питание, сон, спорт, биохакинг, экономика |
| 🖥️ [104 — Avalonia & XAML Mobile](104-avalonia-xaml-mobile/README.md) | 1 | Avalonia, MAUI, Uno |
| 💼 [105 — Финансовые корпорации](105-financial-corporations/README.md) | 7 | История, власть, скандалы, технологии, корп. культура |
| 🛠️ [106 — DevOps, инфраструктура и CI/CD](106-devops-cicd/README.md) | 4 | CI/CD, Docker, K8s, облака, мониторинг |
| 🧑‍💼 [107 — Софт-скиллы для Senior](107-soft-skills-senior/README.md) | 3 | Коммуникация, менторинг, конфликты, решения |

</details>

<details>
<summary><strong>🧠 Наука и AI</strong> (секции 201–204)</summary>

| Секция | Статей | О чём |
|--------|--------|-------|
| 🔬 [201 — Научные открытия XX века](201-science-20th-century/README.md) | 1 | Кванты, ДНК, космос |
| 🔬 [202 — Научные открытия XXI века](202-science-21st-century/README.md) | 1 | AI, CRISPR, гравитация |
| 🔬 [203 — Теории происхождения](203-origin-theories/README.md) | 4 | Научные, псевдонаучные, философские |
| 🤖 [204 — Искусственный интеллект и ML](204-ai-ml/README.md) | 3 | ML, LLM, NLP, RAG, AI-агенты, MLOps |

</details>

<details>
<summary><strong>📚 Гуманитарные науки</strong> (секции 301–312)</summary>

| Секция | Статей | О чём |
|--------|--------|-------|
| 📜 [301 — История 1810–1900](301-history-1810-1900/README.md) | 2 | События, конфликты XIX века |
| 📜 [302 — История 1901–1930](302-history-1901-1930/README.md) | 2 | Мирные войны, революции |
| 📜 [303 — История 1931–1945](303-history-1931-1945/README.md) | 2 | Вторая мировая война |
| 📜 [304 — История 1946–1980](304-history-1946-1980/README.md) | 2 | Холодная война, космос |
| 📜 [305 — История 1981–1993](305-history-1981-1993/README.md) | 2 | PC революция, распад СССР |
| 📜 [306 — История 1993–2012](306-history-1993-2012/README.md) | 2 | Интернет, Web 2.0, кризисы |
| 📜 [307 — История 2013–present](307-history-2013-present/README.md) | 2 | AI, войны, пандемия |
| 📖 [308 — Мировая литература](308-world-literature/README.md) | 11 | Классика, фантастика, детектив, антиутопия |
| 🏛️ [309 — Мифы и легенды](309-myths-legends/README.md) | 7 | Греция, Скандинавия, Египет, Славяне и др. |
| 🕍 [310 — Мировые религии](310-world-religions/README.md) | 7 | Христианство, ислам, буддизм, иудаизм |
| 🧠 [311 — Современная психология](311-modern-psychology/README.md) | 11 | Психология, зависимости, конфликты, счастье |
| 🧠 [312 — Философия и критическое мышление](312-philosophy-critical-thinking/README.md) | 3 | Логика, научный метод, этика |

</details>

---

# 📚 Полное содержание по разделам

### [🔒 001 — Безопасность](001-security/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | OWASP Top 10 (SQL Injection, XSS, CSRF, XXE, Deserialization) | [01-owasp-top10.md](001-security/01-owasp-top10.md) |
| 2 | Аутентификация и авторизация (JWT, OAuth 2.0, Identity, Claims) | [02-authentication-authorization.md](001-security/02-authentication-authorization.md) |
| 3 | Шифрование и хеширование (AES-GCM, RSA, DPAPI, PBKDF2, argon2) | [03-encryption-hashing.md](001-security/03-encryption-hashing.md) |
| 4 | Безопасность API (Rate Limiting, CORS, Validation, Audit) | [04-api-security.md](001-security/04-api-security.md) |
| 5 | Конфигурация и транспорт (Secrets, mTLS, K8s, Dependency Confusion) | [05-configuration-transport.md](001-security/05-configuration-transport.md) |

---

### [🖥️ 002 — WPF (Продвинутые вопросы)](002-wpf-advanced/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | VisualTree vs LogicalTree | [01-visual-tree.md](002-wpf-advanced/01-visual-tree.md) |
| 2 | Dependency Properties (приоритет, coercion, attached) | [02-dependency-properties.md](002-wpf-advanced/02-dependency-properties.md) |
| 3 | Routed Events и Commands (Bubbling/Tunneling, ICommand) | [03-routed-events-commands.md](002-wpf-advanced/03-routed-events-commands.md) |
| 4 | Data Binding (PriorityBinding, MultiBinding, Validation) | [04-data-binding.md](002-wpf-advanced/04-data-binding.md) |
| 5 | Templating (DataTemplate, ControlTemplate, Styles) | [05-templating.md](002-wpf-advanced/05-templating.md) |
| 6 | Производительность (Virtualization, Freezable, ShaderEffects) | [06-performance.md](002-wpf-advanced/06-performance.md) |
| 7 | Dispatcher (InvokeAsync, Priority, Background Operations) | [07-dispatcher-threading.md](002-wpf-advanced/07-dispatcher-threading.md) |
| 8 | WeakEvent и MVVM (CommunityToolkit.Mvvm, Messenger) | [08-weak-event-mvvm.md](002-wpf-advanced/08-weak-event-mvvm.md) |
| 9 | Attached Behaviours (NumericInput, DragDrop, Event-to-Command) | [09-attached-behaviours.md](002-wpf-advanced/09-attached-behaviours.md) |

---

### [🧵 003 — Многопоточность](003-multithreading/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Thread vs Task vs ThreadPool | [01-thread-task-threadpool.md](003-multithreading/01-thread-task-threadpool.md) |
| 2 | async/await — State Machine и SynchronizationContext | [02-async-await.md](003-multithreading/02-async-await.md) |
| 3 | Task — WhenAll, TaskCompletionSource, CancellationToken | [03-task-advanced.md](003-multithreading/03-task-advanced.md) |
| 4 | Синхронизация (lock, SemaphoreSlim, RWLock, Interlocked CAS) | [04-synchronization.md](003-multithreading/04-synchronization.md) |
| 5 | Concurrent Collections (Dict, Queue, Channel, Immutable) | [05-concurrent-collections.md](003-multithreading/05-concurrent-collections.md) |
| 6 | Parallel и PLINQ | [06-parallel-plinq.md](003-multithreading/06-parallel-plinq.md) |
| 7 | TLS и TPL Dataflow (AsyncLocal, Pipeline, Backpressure) | [07-tls-dataflow.md](003-multithreading/07-tls-dataflow.md) |
| 8 | Memory Model (volatile, DCL, ABA, барьеры) | [08-memory-model.md](003-multithreading/08-memory-model.md) |

---

### [🛡️ 004 — Защита данных, DLP и кибербезопасность](004-dlp/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Основы DLP и классификация данных (PII, PCI, Data Discovery) | [01-dlp-basics-classification.md](004-dlp/01-dlp-basics-classification.md) |
| 2 | Шифрование (TDE, Always Encrypted, mTLS, Anonymization) | [02-encryption-at-rest-transit.md](004-dlp/02-encryption-at-rest-transit.md) |
| 3 | Защита от утечек (маскирование логов, API, SecureString) | [03-data-leakage-prevention.md](004-dlp/03-data-leakage-prevention.md) |
| 4 | DLP в Azure (Purview, SQL Ledger, Immutable Blob, Policy) | [04-azure-dlp.md](004-dlp/04-azure-dlp.md) |
| 5 | Аудит и мониторинг (SIEM, Chain Hashing, Incident Response) | [05-audit-monitoring.md](004-dlp/05-audit-monitoring.md) |
| 6 | Как защитить свои данные от хакеров и мошенников | [06-personal-data-protection.md](004-dlp/06-personal-data-protection.md) |

---

### [🗄️ 005 — Коллекции и алгоритмы](005-collections-algorithms/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Внутреннее устройство коллекций (List, Dictionary, HashSet) | [01-dotnet-collections.md](005-collections-algorithms/01-dotnet-collections.md) |
| 2 | Big O, сортировка, поиск, два указателя, скользящее окно | [02-complexity-sorting.md](005-collections-algorithms/02-complexity-sorting.md) |
| 3 | Сравнение коллекций (Generic vs Concurrent vs Immutable) | [03-collections-comparison.md](005-collections-algorithms/03-collections-comparison.md) |
| 4 | LINQ Internals (deferred, streaming, IQueryable) | [04-linq-internals.md](005-collections-algorithms/04-linq-internals.md) |
| 5 | Span, Memory, stackalloc, ArrayPool | [05-spans-memory.md](005-collections-algorithms/05-spans-memory.md) |

---

### [🏗️ 006 — Шаблоны проектирования](006-design-patterns/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | SOLID и принципы (SRP, OCP, LSP, ISP, DIP, DRY, YAGNI) | [01-solid-principles.md](006-design-patterns/01-solid-principles.md) |
| 2 | GoF Порождающие (Singleton, Factory, Builder, Prototype) | [02-gof-creational.md](006-design-patterns/02-gof-creational.md) |
| 3 | GoF Структурные (Adapter, Decorator, Proxy, Facade, Composite) | [03-gof-structural.md](006-design-patterns/03-gof-structural.md) |
| 4 | GoF Поведенческие (Strategy, Observer, Command, State, Chain) | [04-gof-behavioral.md](006-design-patterns/04-gof-behavioral.md) |
| 5 | Enterprise (Repository, CQRS, Outbox, Saga, Specification) | [05-enterprise-patterns.md](006-design-patterns/05-enterprise-patterns.md) |
| 6 | Clean Architecture и DI (Lifetimes, Composition Root, Scrutor) | [06-clean-architecture-di.md](006-design-patterns/06-clean-architecture-di.md) |

---

### [⚡ 007 — Оптимизация и тестирование](007-optimization-testing/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Профилирование (BenchmarkDotNet, dotnet-trace, PerfView) | [01-profiling-benchmarking.md](007-optimization-testing/01-profiling-benchmarking.md) |
| 2 | GC и память (Generations, LOH, Struct vs Class, Pooling) | [02-memory-gc.md](007-optimization-testing/02-memory-gc.md) |
| 3 | Производительность кода (allocation-free, SIMD, inlining) | [03-code-performance.md](007-optimization-testing/03-code-performance.md) |
| 4 | Unit-тестирование (xUnit, NSubstitute, TDD, Code Coverage) | [04-unit-testing.md](007-optimization-testing/04-unit-testing.md) |
| 5 | Интеграционное тестирование (WebApplicationFactory, TestContainers) | [05-integration-testing.md](007-optimization-testing/05-integration-testing.md) |
| 6 | Качество кода (Roslyn, .editorconfig, SonarQube, Code Review) | [06-code-quality.md](007-optimization-testing/06-code-quality.md) |

---

### [🧠 008 — Advanced GC и Memory Management](008-gc-memory/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | GC Internals (Mark & Compact, Generations, Segments, Finalization) | [01-gc-internals.md](008-gc-memory/01-gc-internals.md) |
| 2 | GC Modes (Workstation vs Server, Background, DATAS, Latency) | [02-gc-modes-config.md](008-gc-memory/02-gc-modes-config.md) |
| 3 | Диагностика (dotnet-dump, SOS, PerfView, ETW, Leak Investigation) | [03-memory-diagnostics.md](008-gc-memory/03-memory-diagnostics.md) |
| 4 | Native Memory (SafeHandle, GC.AddMemoryPressure, P/Invoke, NativeMemory) | [04-native-memory.md](008-gc-memory/04-native-memory.md) |
| 5 | GC Tuning (ASP.NET, Desktop, Real-time, NUMA, Containers, HeapHardLimit) | [05-gc-tuning.md](008-gc-memory/05-gc-tuning.md) |

---

### [🔐 009 — Authorization Libraries & Best Practices](009-auth-libraries/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Authorization Libraries (IdentityServer, Entra ID, Auth0, Keycloak, OPA) | [01-auth-libraries-best-practices.md](009-auth-libraries/01-auth-libraries-best-practices.md) |

---

### [🅰️ 010 — Angular](010-angular/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Angular Basics (Components, DI, Routing, Forms, Directives) | [01-angular-basics.md](010-angular/01-angular-basics.md) |
| 2 | Angular Advanced (Change Detection, NgRx, SSR, Signals) | [02-angular-advanced.md](010-angular/02-angular-advanced.md) |
| 3 | Angular Version History (v2-v19) | [03-angular-version-history.md](010-angular/03-angular-version-history.md) |

---

### [💚 011 — Vue.js](011-vuejs/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Vue.js Basics (Composition API, Reactivity, Directives) | [01-vuejs-basics.md](011-vuejs/01-vuejs-basics.md) |
| 2 | Vue.js Advanced (Pinia, Composables, Transitions, Custom Directives) | [02-vuejs-advanced.md](011-vuejs/02-vuejs-advanced.md) |
| 3 | Vue.js Version History (v2-v3) | [03-vuejs-version-history.md](011-vuejs/03-vuejs-version-history.md) |

---

### [⚛️ 012 — React](012-react/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | React Basics (Hooks, JSX, Components, Context, Events) | [01-react-basics.md](012-react/01-react-basics.md) |
| 2 | React Advanced (memo, useMemo, useReducer, Portals, Suspense, Custom Hooks) | [02-react-advanced.md](012-react/02-react-advanced.md) |
| 3 | React Version History (v0.14-v19) | [03-react-version-history.md](012-react/03-react-version-history.md) |

---

### [🌐 013 — HTML5 & CSS3](013-html5-css3/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | HTML5 CSS3 Classics (Semantic HTML, Flexbox, Grid, Animations) | [01-html5-css3-classics.md](013-html5-css3/01-html5-css3-classics.md) |
| 2 | HTML5 New Features Part 1 (Canvas, SVG, WebSocket, Workers, IndexedDB) | [02-html5-new-features-part1.md](013-html5-css3/02-html5-new-features-part1.md) |
| 3 | HTML5 New Features Part 2 (Web Components, Container Queries, WebGPU, Wasm, PWA) | [03-html5-new-features-part2.md](013-html5-css3/03-html5-new-features-part2.md) |
| 4 | Tailwind CSS | [04-tailwind-css.md](013-html5-css3/04-tailwind-css.md) |
| 5 | Bootstrap | [05-bootstrap.md](013-html5-css3/05-bootstrap.md) |
| 6 | Material Design | [06-material-design.md](013-html5-css3/06-material-design.md) |
| 7 | PWA (Progressive Web Apps) | [07-pwa.md](013-html5-css3/07-pwa.md) |
| 8 | CSS Анимации и эффекты | [08-animations-effects.md](013-html5-css3/08-animations-effects.md) |

---

### [🎮 014 — Game Engines For Web](014-game-engines/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Web Game Engines Comparison (Three.js, Babylon, Phaser, PixiJS, Unity/Godot WebGL) | [01-web-game-engines-comparison.md](014-game-engines/01-web-game-engines-comparison.md) |

---

### [🟠 015 — Svelte](015-svelte/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Svelte Basics (Reactivity, Components, Stores) | [01-svelte-basics.md](015-svelte/01-svelte-basics.md) |
| 2 | Svelte Advanced & No Virtual DOM (Compiler, Solid.js, SvelteKit, SSR) | [02-svelte-advanced-no-vdom.md](015-svelte/02-svelte-advanced-no-vdom.md) |
| 3 | Svelte Practical Tips | [03-svelte-practical-tips.md](015-svelte/03-svelte-practical-tips.md) |

---

### [🤖 016 — Modern Code Review, Copilot & Agent Coding](016-modern-code-review/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Code Review Best Practices, AI Pair Programming, Agentic Workflows | [01-modern-code-review-copilot.md](016-modern-code-review/01-modern-code-review-copilot.md) |

---

### [🗄️ 017 — Базы данных](017-databases/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | [Реляционные базы данных и SQL](017-databases/01-relational-sql.md) | ACID, нормализация, индексы, транзакции, JOIN'ы, оконные функции |
| 2 | [NoSQL и NewSQL](017-databases/02-nosql-newsql.md) | Document, Key-Value, Wide-Column, Graph, NewSQL, CAP theorem |
| 3 | [Проектирование и администрирование БД](017-databases/03-design-administration.md) | Схемы, миграции, шардинг, репликация, бэкапы, мониторинг |

---

### [🏗️ 018 — Системный дизайн и архитектура](018-system-design-architecture/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | [Основы системного дизайна](018-system-design-architecture/01-system-design-basics.md) | Load balancing, caching, CDN, rate limiting, async processing |
| 2 | [Микросервисы и распределённые системы](018-system-design-architecture/02-microservices-distributed.md) | Service decomposition, inter-service communication, distributed transactions |
| 3 | [Масштабирование и performance](018-system-design-architecture/03-scaling-performance.md) | Horizontal vs vertical, CAP, consistency, caching strategies, capacity planning |

---

### [🎨 101 — Diffusion Models](101-diffusion-models/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Diffusion Models Overview (SD, DALL-E, Midjourney, Flux, Sora, limitations) | [01-diffusion-models-overview.md](101-diffusion-models/01-diffusion-models-overview.md) |

---

### [🚀 102 — Start-up Theory & Facts](102-startup-theory/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Start-up Theory (PMF, Lean Startup, Metrics, Fundraising, Pivot) | [01-startup-theory-facts.md](102-startup-theory/01-startup-theory-facts.md) |
| 2 | Startup Success & Failure Stories | [02-startup-success-failure.md](102-startup-theory/02-startup-success-failure.md) |
| 3 | Полезные источники и рекомендации | [03-startup-resources-recommendations.md](102-startup-theory/03-startup-resources-recommendations.md) |
| 4 | Платформы для создания и продажи плагинов | [04-plugin-marketplaces.md](102-startup-theory/04-plugin-marketplaces.md) |

---

### [🥗 103 — Здоровое питание и образ жизни](103-healthy-lifestyle/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Основы здорового питания, режим, спорт, сон, продуктивность | [01-healthy-lifestyle-basics.md](103-healthy-lifestyle/01-healthy-lifestyle-basics.md) |
| 2 | Экономическая грамотность и экономия в быту | [05-economic-literacy.md](103-healthy-lifestyle/05-economic-literacy.md) |
| 3 | [Биохакинг и долголетие](103-healthy-lifestyle/03-biohacking-longevity.md) | Hallmarks of Aging, аутофагия, добавки, крио/сауна |
| 4 | [Работа мозга — когнитивная оптимизация](103-healthy-lifestyle/04-brain-optimization.md) | Нейротрансмиттеры, ноотропы, протоколы фокуса |
| 5 | [Экономическая грамотность](103-healthy-lifestyle/05-economic-literacy.md) | Бюджет, сбережения, инвестиции |

---

### [🖥️ 104 — Avalonia & XAML Mobile](104-avalonia-xaml-mobile/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Avalonia UI, .NET MAUI, Uno Platform — сравнение и архитектура | [01-avalonia-xaml-mobile.md](104-avalonia-xaml-mobile/01-avalonia-xaml-mobile.md) |

---

### [💼 105 — Финансовые корпорации](105-financial-corporations/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | История и основатели | [01-history-founders.md](105-financial-corporations/01-history-founders.md) |
| 2 | Финансовая мощь и глобальное влияние | [02-power-influence.md](105-financial-corporations/02-power-influence.md) |
| 3 | Скандалы и кризисы | [03-scandals-crises.md](105-financial-corporations/03-scandals-crises.md) |
| 4 | Инновации и технологии | [04-innovation-technology.md](105-financial-corporations/04-innovation-technology.md) |
| 5 | Социальное влияние и критика | [05-social-impact.md](105-financial-corporations/05-social-impact.md) |
| 6 | Полезные библиотеки и ресурсы | [06-libraries-resources.md](105-financial-corporations/06-libraries-resources.md) |

---

### [🛠️ 106 — DevOps, инфраструктура и CI/CD](106-devops-cicd/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | [Основы DevOps и CI/CD](106-devops-cicd/01-devops-basics.md) | CI/CD, GitOps, IaC, Trunk-Based Development |
| 2 | [Контейнеризация и оркестрация](106-devops-cicd/02-containerization-orchestration.md) | Docker, Kubernetes, Helm, Service Mesh |
| 3 | [Облачные платформы и инфраструктура](106-devops-cicd/03-cloud-platforms.md) | AWS, Azure, GCP, Terraform, FinOps |
| 4 | [Мониторинг, логирование и observability](106-devops-cicd/04-monitoring-observability.md) | Prometheus, OpenTelemetry, SLI/SLO, алертинг |

---

### [🧑‍💼 107 — Софт-скиллы для Senior](107-soft-skills-senior/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | [Коммуникация и фасилитация](107-soft-skills-senior/01-communication-facilitation.md) | SBI-фидбек, фасилитация, аргументация, RFC |
| 2 | [Менторство и развитие команды](107-soft-skills-senior/02-mentoring-team-development.md) | Situational Leadership, code review, 1:1, инженерная культура |
| 3 | [Принятие решений и управление конфликтами](107-soft-skills-senior/03-decision-conflicts.md) | ADR, Harvard Negotiation, DESC, BATNA |

---

### [🔬 201 — Научные открытия XX века](201-science-20th-century/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Физика, биология, химия, медицина, космос XX века (1900–1999) | [01-science-20th-century.md](201-science-20th-century/01-science-20th-century.md) |

---

### [🔬 202 — Научные открытия XXI века](202-science-21st-century/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Физика, биология, AI, космос XXI века (2000–2025) | [01-science-21st-century.md](202-science-21st-century/01-science-21st-century.md) |

---

### [🔬 203 — Теории происхождения](203-origin-theories/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Научные теории | [01-scientific.md](203-origin-theories/01-scientific.md) |
| 2 | Псевдонаучные теории | [02-pseudoscientific.md](203-origin-theories/02-pseudoscientific.md) |
| 3 | Философские концепции | [03-philosophical.md](203-origin-theories/03-philosophical.md) |
| 4 | Полезные библиотеки и ресурсы | [04-libraries-resources.md](203-origin-theories/04-libraries-resources.md) |

---

### [🤖 204 — Искусственный интеллект и ML](204-ai-ml/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | [Основы машинного обучения](204-ai-ml/01-ml-basics.md) | Supervised/Unsupervised, нейросети, метрики |
| 2 | [LLM и NLP](204-ai-ml/02-llm-nlp.md) | Transformer, GPT, BERT, RAG, fine-tuning |
| 3 | [Практическое применение AI](204-ai-ml/03-ai-practical.md) | AI-агенты, Semantic Kernel, MLOps |

---

### [📜 301 — Historical Dates: 1810–1900](301-history-1810-1900/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | События 1810-1900 (войны, технологии, наука, политика) | [01-history-1810-1900.md](301-history-1810-1900/01-history-1810-1900.md) |

---

### [📜 302 — Historical Dates: 1901–1930](302-history-1901-1930/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | События 1901-1930 (Мирные войны, революции, наука) | [01-history-1901-1930.md](302-history-1901-1930/01-history-1901-1930.md) |

---

### [📜 303 — Historical Dates: 1931–1945](303-history-1931-1945/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | События 1931-1945 (Вторая мировая война) | [01-history-1931-1945.md](303-history-1931-1945/01-history-1931-1945.md) |

---

### [📜 304 — Historical Dates: 1946–1980](304-history-1946-1980/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | События 1946-1980 (Холодная война, космос, технологии) | [01-history-1946-1980.md](304-history-1946-1980/01-history-1946-1980.md) |

---

### [📜 305 — Historical Dates: 1981–1993](305-history-1981-1993/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | События 1981-1993 (PC революция, распад СССР) | [01-history-1981-1993.md](305-history-1981-1993/01-history-1981-1993.md) |

---

### [📜 306 — Historical Dates: 1993–2012](306-history-1993-2012/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | События 1993-2012 (Интернет, Web 2.0, кризисы) | [01-history-1993-2012.md](306-history-1993-2012/01-history-1993-2012.md) |

---

### [📜 307 — Historical Dates: 2013–Present](307-history-2013-present/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | События 2013-present (AI, войны, пандемия, космос), с точными датами | [01-history-2013-present.md](307-history-2013-present/01-history-2013-present.md) |

---

### [📖 308 — Мировая литература](308-world-literature/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Общий справочник | [01-world-literature.md](308-world-literature/01-world-literature.md) |
| 2 | Мировая классика | [02-world-classics.md](308-world-literature/02-world-classics.md) |
| 3 | Русская классика | [03-russian-classics.md](308-world-literature/03-russian-classics.md) |
| 4 | Современная литература | [04-contemporary-literature.md](308-world-literature/04-contemporary-literature.md) |
| 5 | Фантастика (Sci-Fi) | [05-science-fiction.md](308-world-literature/05-science-fiction.md) |
| 6 | Фэнтези | [06-fantasy.md](308-world-literature/06-fantasy.md) |
| 7 | Детектив и триллер | [07-detective-thriller.md](308-world-literature/07-detective-thriller.md) |
| 8 | Антиутопия | [08-dystopia.md](308-world-literature/08-dystopia.md) |
| 9 | Исторический роман | [09-historical-novel.md](308-world-literature/09-historical-novel.md) |
| 10 | Нон-фикшн и научпоп | [10-nonfiction.md](308-world-literature/10-nonfiction.md) |
| 11 | Исторические исследования | [11-historical-research.md](308-world-literature/11-historical-research.md) |

---

### [🏛️ 309 — Мифы и легенды](309-myths-legends/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Греческая и римская мифология | [01-greek-roman.md](309-myths-legends/01-greek-roman.md) |
| 2 | Скандинавская и германская мифология | [02-norse-germanic.md](309-myths-legends/02-norse-germanic.md) |
| 3 | Египетская и месопотамская мифология | [03-egypt-mesopotamia.md](309-myths-legends/03-egypt-mesopotamia.md) |
| 4 | Славянская и балтийская мифология | [04-slavic-baltic.md](309-myths-legends/04-slavic-baltic.md) |
| 5 | Индийская, китайская и японская мифология | [05-india-china-japan.md](309-myths-legends/05-india-china-japan.md) |
| 6 | Кельтская, финская и мифология Америки | [06-celtic-finno-americas.md](309-myths-legends/06-celtic-finno-americas.md) |
| 7 | Полезные библиотеки и ресурсы | [07-libraries-resources.md](309-myths-legends/07-libraries-resources.md) |

---

### [🕍 310 — Мировые религии](310-world-religions/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Ранние верования и древние религии | [01-early-ancient.md](310-world-religions/01-early-ancient.md) |
| 2 | Буддизм и восточные религии | [02-buddhism-eastern.md](310-world-religions/02-buddhism-eastern.md) |
| 3 | Иудаизм | [03-judaism.md](310-world-religions/03-judaism.md) |
| 4 | Христианство | [04-christianity.md](310-world-religions/04-christianity.md) |
| 5 | Ислам | [05-islam.md](310-world-religions/05-islam.md) |
| 6 | Новые религиозные движения | [06-modern-movements.md](310-world-religions/06-modern-movements.md) |
| 7 | Полезные библиотеки и ресурсы | [07-libraries-resources.md](310-world-religions/07-libraries-resources.md) |

---

### [🧠 311 — Современная психология](311-modern-psychology/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | Разделы психологии | [01-branches.md](311-modern-psychology/01-branches.md) |
| 2 | Известные учёные и деятели | [02-famous-figures.md](311-modern-psychology/02-famous-figures.md) |
| 3 | Псевдопсихологи и за что они цепляют | [03-pseudopsychology.md](311-modern-psychology/03-pseudopsychology.md) |
| 4 | Борьба с зависимостями | [04-addiction.md](311-modern-psychology/04-addiction.md) |
| 5 | Психотипы и характеры | [05-psychotypes.md](311-modern-psychology/05-psychotypes.md) |
| 6 | Корректировка социальных отношений | [06-social-relations.md](311-modern-psychology/06-social-relations.md) |
| 7 | Инфобизнес и НЛП | [07-infobusiness-nlp.md](311-modern-psychology/07-infobusiness-nlp.md) |
| 8 | Как правильно разрешать конфликты | [08-conflict-resolution.md](311-modern-psychology/08-conflict-resolution.md) |
| 9 | Как правильно преодолевать трудности | [09-overcoming-difficulties.md](311-modern-psychology/09-overcoming-difficulties.md) |
| 10 | Как быть счастливым | [10-happiness.md](311-modern-psychology/10-happiness.md) |
| 11 | Корпоративная культура | [11-corporate-culture.md](311-modern-psychology/11-corporate-culture.md) |

---

### [🧠 312 — Философия и критическое мышление](312-philosophy-critical-thinking/README.md)

| # | Раздел | Файл |
|---|--------|------|
| 1 | [Основы критического мышления](312-philosophy-critical-thinking/01-critical-thinking.md) | Логические ошибки, аргументация, фаллибилизм |
| 2 | [Научный метод и логика](312-philosophy-critical-thinking/02-scientific-method-logic.md) | Индукция/дедукция, фальсификация, бритва Оккамы |
| 3 | [Философские концепции и этика](312-philosophy-critical-thinking/03-philosophy-ethics.md) | Метафизика, эпистемология, этика, экзистенциализм |

---

## 🎯 Итого

**41 раздел, 200+ статей** по всем темам.

| Раздел | Статей | Ключевые темы |
|--------|--------|---------------|
| 🔒 001 — Безопасность | 5 | OWASP, JWT, OAuth, AES, PBKDF2, CORS, mTLS |
| 🖥️ 002 — WPF Advanced | 9 | VisualTree, DP, RoutedEvents, Binding, Templates, MVVM |
| 🧵 003 — Многопоточность | 8 | async/await, Task, lock-free, Channel, Dataflow, Memory Model |
| 🛡️ 004 — Защита данных и DLP | 6 | Классификация, TDE, шифрование, DLP, аудит, защита данных пользователя |
| 🗄️ 005 — Коллекции | 5 | Dictionary internals, Big O, LINQ, Span, Memory |
| 🏗️ 006 — Паттерны | 6 | SOLID, GoF (12), CQRS, Outbox, Saga, Clean Architecture |
| ⚡ 007 — Оптимизация | 6 | BenchmarkDotNet, GC, SIMD, xUnit, TestContainers, Code Review |
| 🧠 008 — GC & Memory | 5 | Mark & Compact, SOS, LOH, POH, SafeHandle, HeapHardLimit, NUMA |
| 🔐 009 — Authorization | 1 | IdentityServer, Entra ID, Auth0, OPA, ABAC |
| 🅰️ 010 — Angular | 5 | Components, NgRx, Signals, SSR, Practical Tips |
| 💚 011 — Vue.js | 5 | Composition API, Pinia, Nuxt.js, Composables |
| ⚛️ 012 — React | 5 | Hooks, memo, Next.js, Suspense, Practical Tips |
| 🌐 013 — HTML5/CSS3 | 8 | Semantic HTML, Flexbox/Grid, Canvas, Tailwind, Bootstrap, Material, PWA, анимации |
| 🎮 014 — Game Engines | 1 | Three.js, Babylon, Phaser, Unity |
| 🟠 015 — Svelte | 3 | Compiler, No VDOM, SvelteKit |
| 🤖 016 — Code Review | 1 | Copilot, Agentic Workflows |
| 🗄️ 017 — Базы данных | 3 | SQL, NoSQL, NewSQL, проектирование, шардинг, репликация |
| 🏗️ 018 — Системный дизайн | 3 | Load balancing, caching, микросервисы, масштабирование, CAP |
| 🎨 101 — Diffusion Models | 1 | SD, DALL-E, Midjourney, Flux, Sora |
| 🚀 102 — Start-up Theory | 4 | PMF, Fundraising, успех/провал, ресурсы, плагины |
| 🥗 103 — Healthy Lifestyle | 5 | Питание, сон, спорт, биохакинг, мозг, экономика в быту |
| 🖥️ 104 — Avalonia/XAML | 1 | Avalonia, MAUI |
| 💼 105 — Финансовые корпорации | 7 | История, власть, скандалы, технологии, критика, ресурсы, корп. культура |
| 🛠️ 106 — DevOps, инфраструктура и CI/CD | 4 | CI/CD, Docker, K8s, облака, мониторинг, observability |
| 🧑‍💼 107 — Софт-скиллы для Senior | 3 | Коммуникация, менторинг, конфликты, решения, фасилитация |
| 🔬 201 — Science XX | 1 | Кванты, ДНК, космос |
| 🔬 202 — Science XXI | 1 | AI, CRISPR, гравитация |
| 🔬 203 — Теории происхождения | 4 | Научные, псевдонаучные, философские + ресурсы |
| 🤖 204 — Искусственный интеллект и ML | 3 | ML, LLM, NLP, RAG, AI-агенты, MLOps |
| 📜 301-307 — History (7) | 7 | 1810–2025 |
| 📖 308 — Literature | 11 | Классика, фантастика, детектив, антиутопия и др. |
| 🏛️ 309 — Мифы и легенды | 7 | Греция, Скандинавия, Египет, Славяне, Индия, Кельты + ресурсы |
| 🕍 310 — Мировые религии | 7 | Религии от древних до современных + ресурсы |
| 🧠 311 — Современная психология | 11 | Разделы, учёные, зависимости, психотипы, отношения, конфликты, счастье, корп. культура |
| 🧠 312 — Философия | 3 | Логика, научный метод, этика, критическое мышление |

## 💡 Рекомендации для Senior

- **Не просто знайте**, а понимайте **почему** работает так, а не иначе
- Будьте готовы объяснить **trade-offs** каждого подхода
- Покажите знание **internals** — как работает State Machine, Dispatcher, Binding Engine
- Подкрепляйте ответы **примерами из реального опыта**
- Обращайте внимание на **факты** — они показывают глубину понимания

## 🚀 Как запустить viewer (1 строка)

```bash
python3 -m http.server 8080                   # Python
npx serve . -p 8080                            # Node.js (npx)
npx http-server -p 8080                        # Node.js (альтернатива)
php -S localhost:8080                          # PHP
ruby -run -e httpd . -p 8080                   # Ruby
deno run -A jsr:@std/http/file-server          # Deno
dotnet tool install --global dotnet-serve ; dotnet serve --port 8080  # .NET
caddy file-server --listen :8080               # Caddy
```
