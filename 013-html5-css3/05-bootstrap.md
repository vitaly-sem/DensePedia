# Bootstrap

> **Тип**: CSS framework (component-based)
> **Автор**: Mark Otto, Jacob Thornton (Twitter)
> **Релиз**: 2011 (v1), 2023 (v5.3)

---

## Концепция

**Component-first** подход: готовые компоненты с предустановленными стилями.

```html
<div class="card" style="width: 18rem;">
  <img src="..." class="card-img-top" alt="...">
  <div class="card-body">
    <h5 class="card-title">Заголовок</h5>
    <p class="card-text">Описание</p>
    <a href="#" class="btn btn-primary">Подробнее</a>
  </div>
</div>
```

### Версии

| Версия | Дата | Ключевые изменения |
|---|---|---|
| **v1** | 2011 | Первый релиз от Twitter (только CSS) |
| **v2** | 2012 | Responsive design, 12-column grid |
| **v3** | 2013 | **Mobile-first**, flat design, Glyphicons |
| **v4** | 2018 | Sass, Flexbox grid, cards, rem единицы |
| **v5** | 2021 | CSS Custom Properties, utilities API, updated forms |
| **v5.3** | 2023 | Dark mode, color modes |

---

## Ключевые компоненты

| Компонент | Назначение |
|---|---|
| **Grid** | 12-колоночная сетка (flexbox) |
| **Navbar** | Готовая навигация |
| **Cards** | Универсальные контейнеры |
| **Buttons** | Кнопки с разными стилями |
| **Forms** | Формы с валидацией |
| **Modals** | Модальные окна |
| **Carousel** | Слайдер |
| **Accordion** | Раскрывающиеся панели |
| **Toasts** | Уведомления |
| **Tooltips** | Всплывающие подсказки |

---

## Grid System

```html
<div class="container">
  <div class="row">
    <div class="col-sm-8">col-8</div>
    <div class="col-sm-4">col-4</div>
  </div>
</div>
```

### Breakpoints

| Breakpoint | Prefix | Width |
|---|---|---|
| X-Small | — | <576px |
| Small | `sm` | ≥576px |
| Medium | `md` | ≥768px |
| Large | `lg` | ≥992px |
| Extra Large | `xl` | ≥1200px |
| XX-Large | `xxl` | ≥1400px |

---

## Bootstrap vs Tailwind

| Критерий | Bootstrap | Tailwind CSS |
|---|---|---|
| **Подход** | Component-based | Utility-first |
| **Размер** | ~25KB gzipped | ~10KB purged |
| **Старт** | Быстрый (готовые компоненты) | Медленный (нужен дизайн) |
| **Гибкость** | Ограничен компонентами | Любой дизайн |
| **Компоненты** | ✅ Встроенные (JS + CSS) | ❌ Нужно собрать самому |
| **jQuery?** | v5 — vanilla JS (ранее был jQuery) | Нет |
| **Кастомизация** | Sass variables | tailwind.config.js |
| **Уникальность дизайна** | Все сайты похожи | Каждый сайт может быть уникальным |

---

## Плюсы и минусы

### Плюсы ✅

- **Быстрый старт** — готовые компоненты «из коробки»
- **Документация** — лучшая среди CSS-фреймворков
- **Сообщество** — огромное количество тем, шаблонов, тем
- **jQuery-free** — с v5 полностью vanilla JS
- **RTL** — поддержка справа-налево (арабский, иврит)

### Минусы ❌

- **Все сайты похожи** — «Bootstrap look» (если не кастомизировать)
- **Избыточность** — много лишнего кода, если нужен только базовый функционал
- **Тяжёлый** — даже после минификации ~30KB
- **Зависимости** — Popper.js для tooltips, popovers

---

## Интеграция

| Пакет | Команда |
|---|---|
| **npm** | `npm i bootstrap @popperjs/core` |
| **CDN** | `<link href="cdn.jsdelivr.net/npm/bootstrap@5.3.x/dist/css/bootstrap.min.css">` |
| **React** | `npm i react-bootstrap bootstrap` |
| **Vue** | `npm i bootstrap-vue-3 bootstrap` |
| **Angular** | `npm i ngx-bootstrap bootstrap` |
| **Rails** | `gem 'bootstrap', '~> 5.3'` |
