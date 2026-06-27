# Angular Version History

---

| Версия | Дата | Ключевые изменения |
|---|---|---|
| **Angular 2** | Sep 2016 | **Rewrite** from AngularJS. Components, DI, Router, TypeScript, Zone.js |
| **Angular 4** | Mar 2017 | Smaller bundles, `*ngIf` with `else`, Animation package, improved `HttpClient` |
| **Angular 5** | Nov 2017 | Build optimizer, `@angular/service-worker`, `@angular/pwa`, `Forms` `updateOn` |
| **Angular 6** | May 2018 | **Angular CLI** workspace, `ng add`/`ng update`, RxJS v6, Elements, Ivy preview |
| **Angular 7** | Oct 2018 | CLI prompts, Virtual Scroll (CDK), Drag & Drop, improved accessibility |
| **Angular 8** | May 2019 | **Ivy** renderer opt-in, Differential loading (ES2015+), Lazy loading with dynamic imports |
| **Angular 9** | Feb 2020 | **Ivy** default renderer, smaller bundles (~25-40%), faster testing, improved build time |
| **Angular 10** | Jun 2020 | Strict mode, `DateRangePicker`, `Wizard` component, deprecated APIs removed |
| **Angular 11** | Nov 2020 | **Automatic font inlining**, Hot Module Replacement (HMR), stricter types, `ng serve` improvements |
| **Angular 12** | May 2021 | **Ivy everywhere** (View Engine removed), Tailwind CSS support, `ng test` with Karma, ESM |
| **Angular 13** | Nov 2021 | **View Engine removed**, IE11 support dropped, RxJS 7.5, Angular Material improvements |
| **Angular 14** | Jun 2022 | **Standalone components** (no NgModule needed), typed forms, `inject()` function |
| **Angular 15** | Nov 2022 | **Image directive** (`NgOptimizedImage`), functional guards, better stack traces |
| **Angular 16** | May 2023 | **Signals** (signal-based reactivity, no Zone.js needed), RxJS interop, hydration |
| **Angular 17** | Nov 2023 | **New control flow** (`@if`/`@for`/`@switch`), **deferred loading** (`@defer`), SSR improvements |
| **Angular 18** | May 2024 | **Material 3**, `@angular/ssr`, default `@if`/`@for`, Signal Forms |
| **Angular 19** | Nov 2024 | **LinkedSignal**, `resource()` API, improved hydration |

**Факт:** Angular follows **semantic versioning**, major every 6 months.

**Факт:** **AngularJS** (1.x) ≠ Angular (2+). AngularJS **EOL** — Jan 2022.

### Breaking Changes — ключевые миграции

| From | To | Что изменилось |
|---|---|---|
| AngularJS (1.x) | Angular (2+) | Полный rewrite |
| RxJS 5 | RxJS 6 | pipe() вместо chain, lettable operators |
| View Engine | Ivy (9+) | Новый renderer, меньше bundle |
| `*ngIf`/`*ngFor` | `@if/@for` (17+) | Блочный control flow |
| NgModule | Standalone (14+) | Component без NgModule |
| Zone.js reactivity | Signals (16+) | Опциональный Signal-based |

**Факт:** Обновление Angular: `ng update @angular/core @angular/cli` — автоматическая миграция.
