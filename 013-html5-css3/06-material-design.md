# Material Design

> **Тип**: Design system
> **Автор**: Google
> **Релиз**: 2014 (Material Design 1), 2018 (Material Design 2), 2021 (Material Design 3 / Material You)

---

## Концепция

Material Design — это не CSS-фреймворк, а полноценная **дизайн-система** от Google.

### Ключевые принципы

| Принцип | Описание |
|---|---|
| **Material как метафора** | Тени, слои, поверхность — элементы ведут себя как физические объекты |
| **Смелый, графичный, намеренный** | Яркие цвета, типографика, осмысленные анимации |
| **Motion создаёт смысл** | Анимация не украшение — она объясняет, что происходит |
| **Адаптивный дизайн** | Одна система для всех экранов |
| **Dynamic Color** (M3) | Цвета подбираются из обоев пользователя |

---

## Версии

| Версия | Дата | Ключевые изменения |
|---|---|---|
| **MD1** | 2014 | Оригинальный Material Design (Lollipop 5.0) |
| **MD2** | 2018 | Material Theming, больше кастомизации |
| **Material You (M3)** | 2021 | Dynamic Color, Monet, адаптивные макеты |
| **Material 3 stable** | 2022 | Полный редизайн, rounded corners |

---

## Реализации

### Web

| Библиотека | Описание |
|---|---|
| **Material Web Components (MWC)** | Google — нативные Web Components |
| **Material Design Lite (MDL)** | Устаревшая (2015–2016) |
| **Materialize CSS** | Сторонний фреймворк |
| **MUI (Material UI)** | React-реализация (самая популярная) |
| **Vuetify** | Vue.js реализация |
| **Angular Material** | Angular-реализация (официальная) |

### Mobile

| Платформа | Реализация |
|---|---|
| **Android** | Jetpack Compose Material 3 |
| **iOS** | Нет официальной (SF Symbols — аналог Apple) |
| **Flutter** | Material library (встроенная) |

---

## Material Design vs Bootstrap vs Tailwind

| Критерий | Material | Bootstrap | Tailwind |
|---|---|---|---|
| **Тип** | Design system | CSS framework | Utility-first CSS |
| **Философия** | Guided (как должно быть) | Components (быстро) | Utility (гибко) |
| **Кастомизация** | Через темы | Через Sass | Через конфиг |
| **Уникальность** | Узнаваемый Google-стиль | Узнаваемый Bootstrap | Любой стиль |
| **Компоненты** | Полный набор | Полный набор | Нет (строить из классов) |
| **Анимация** | Встроенная (motion) | Базовая | Через CSS |
| **Android/iOS** | Нативная поддержка | Нет | Нет |

---

## Material You (M3) — Dynamic Color

### Как работает

1. Android считывает доминирующий цвет с обоев
2. Генерирует 5 ключевых цветов (primary, secondary, tertiary, neutral, neutral variant)
3. Система (и приложения) перекрашиваются в эти цвета

```css
/* Material 3 цвета через CSS custom properties */
:root {
  --md-sys-color-primary: #6750A4;
  --md-sys-color-on-primary: #FFFFFF;
  --md-sys-color-surface: #FEF7FF;
  --md-sys-color-on-surface: #1D1B20;
  --md-ref-typeface-plain: 'Roboto', system-ui, sans-serif;
}
```

### Ключевые компоненты M3

| Компонент | Особенность |
|---|---|
| **Navigation Bar** | Bottom navigation с анимацией |
| **FAB (Floating Action Button)** | Крупный, с иконкой |
| **Cards** | Rounded corners (16px), тени |
| **Switches** | Анимированные |
| **Dialogs** | Полноэкранные на мобильных |
| **Tabs** | Материальные вкладки |

---

## MUI (Material UI) — React

### Установка

```bash
npm install @mui/material @emotion/react @emotion/styled
```

### Использование

```jsx
import { Button, Card, Typography } from '@mui/material';

<Card sx={{ maxWidth: 345, p: 2 }}>
  <Typography variant="h5" gutterBottom>Заголовок</Typography>
  <Button variant="contained" color="primary">Действие</Button>
</Card>
```

---

## Плюсы и минусы

### Плюсы ✅

- **Цельная дизайн-система** — всё выглядит единообразно
- **Анимации** — лучшие среди CSS-фреймворков
- **Material You** — адаптивные цвета (Android 12+)
- **Библиотеки** — Angular Material, MUI, Vuetify — качественные реализации
- **Доступность** — встроенная поддержка a11y

### Минусы ❌

- **Узнаваемость** — все приложения выглядят как Google
- **Тяжёлый** — MUI v5: ~30KB gzipped (tree-shaken)
- **Android-centric** — iOS разработчики жалуются на «не iOS-опыт»
- **Версии** — быстрое устаревание (MDL, Paper)
- **Dynamic Color** — только Android 12+, iOS/web — эмуляция
