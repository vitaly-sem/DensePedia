# Tailwind CSS

> **Тип**: Utility-first CSS framework
> **Автор**: Adam Wathan (Tailwind Labs)
> **Релиз**: 2017 (v0.1), 2021 (v3.0), 2024 (v4.0)

---

## Концепция

### Utility-first подход

Вместо готовых компонентов (как Bootstrap) Tailwind предлагает низкоуровневые CSS-классы:

```html
<div class="flex items-center gap-4 p-6 bg-white rounded-xl shadow-lg hover:shadow-xl transition-shadow">
  <img class="w-12 h-12 rounded-full" src="avatar.jpg" alt="">
  <div>
    <h3 class="text-lg font-semibold text-gray-900">Имя</h3>
    <p class="text-sm text-gray-500">Описание</p>
  </div>
</div>
```

### Преимущества

| Преимущество | Описание |
|---|---|
| **Скорость** | Не нужно писать кастомный CSS, имена классов интуитивны |
| **Small bundle** | PurgeCSS удаляет неиспользуемый CSS → итоговый файл ~10KB |
| **Консистентность** | Design tokens (spacing, colours, typography) — единая система |
| **Responsive** | Встроенные breakpoints: `sm:`, `md:`, `lg:`, `xl:`, `2xl:` |
| **Dark mode** | `dark:` prefix для тёмной темы |
| **Customization** | `tailwind.config.js` — полный контроль над дизайн-системой |

---

## Ключевые возможности

### Responsive Design

```html
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
```

### Dark Mode

```html
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
```

### State Variants

```html
<button class="bg-blue-500 hover:bg-blue-700 focus:ring-2 active:bg-blue-800 disabled:opacity-50">
```

### Custom Theme

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: { brand: { 500: '#3b82f6' } },
      spacing: { 128: '32rem' },
    }
  }
}
```

---

## Сравнение с Bootstrap

| Критерий | Tailwind | Bootstrap |
|---|---|---|
| **Подход** | Utility-first | Component-based |
| **Размер (production)** | ~10KB (purged) | ~25-30KB (min) |
| **Кастомизация** | Через конфиг — полная | Через Sass — частичная |
| **Нейминг классов** | Описательный (flex, p-4) | Семантический (.card, .btn) |
| **Design system** | С нуля (требует дизайнера) | Готовая (стартует быстро) |
| **Кривая обучения** | Нужно выучить много классов | Проще для новичков |
| **Гибкость** | Любой дизайн | Ограничен компонентами |

---

## Плюсы и минусы

### Плюсы ✅

- **Производительность**: минимальный CSS (purged)
- **Контроль**: каждая единица дизайна в конфиге
- **Рефакторинг**: HTML vs CSS — не нужно искать стили
- **Сообщество**: огромное количество плагинов, UI-каталогов
- **Tailwind UI**: готовые компоненты (платные, $299)

### Минусы ❌

- **Длинные class-атрибуты** — HTML раздувается
- **Кривая обучения** — запомнить ~300+ классов
- **Зависимость от конфига** — без него не взлетит
- **«Мусор» в шаблонах** — трудно читать

---

## Версии

| Версия | Дата | Ключевые изменения |
|---|---|---|
| v0.1 | 2017 | Первый публичный релиз |
| v1.0 | 2019 | Стабильный API, PostCSS |
| v2.0 | 2020 | Dark mode, JIT engine, forms plugin |
| v3.0 | 2021 | JIT по умолчанию, colors v2, native form reset |
| v4.0 | 2024 | CSS-first config, Lightning CSS, Oxide |

---

## Интеграция с фреймворками

| Фреймворк | Интеграция |
|---|---|
| **React / Next.js** | `npx create-next-app -e with-tailwindcss` |
| **Vue / Nuxt** | `@nuxtjs/tailwindcss` module |
| **Angular** | `ng add @ngrx/tailwindcss` |
| **Svelte / SvelteKit** | `@sveltejs/adapter` + PostCSS |
| **Laravel** | `laravel/ui tailwindcss --auth` |
| **Rails** | `gem 'tailwindcss-rails'` |
