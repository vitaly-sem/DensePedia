# CSS-анимации и визуальные эффекты

> CSS transitions, animations, keyframes, JavaScript-driven анимации, библиотеки, производительность.

---

## 1. CSS Transitions

Плавный переход между состояниями элемента.

```css
.button {
  background: blue;
  transition: background 0.3s ease, transform 0.2s ease-out;
}
.button:hover {
  background: darkblue;
  transform: scale(1.05);
}
```

### Параметры transition

| Свойство | Значения |
|---|---|
| `transition-property` | `all`, `background`, `opacity`, `transform` |
| `transition-duration` | `0.3s`, `300ms` |
| `transition-timing-function` | `ease`, `linear`, `ease-in`, `ease-out`, `ease-in-out`, `cubic-bezier()` |
| `transition-delay` | `0s`, `0.1s` |

### Анимируемые vs неанимируемые

| ✅ Анимируется | ❌ Не анимируется |
|---|---|
| `opacity`, `transform`, `color`, `background` | `display`, `height: auto` |
| `width`, `height` (числовые) | `font-family` |
| `margin`, `padding` | `background-image` |
| `box-shadow`, `filter` | — |

---

## 2. CSS @keyframes Animations

Более сложные анимации с ключевыми кадрами.

```css
@keyframes slideIn {
  from {
    transform: translateX(-100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

.element {
  animation: slideIn 0.5s ease-out forwards;
}
```

### Свойства animation

| Свойство | Пример |
|---|---|
| `animation-name` | `slideIn` |
| `animation-duration` | `0.5s` |
| `animation-timing-function` | `ease`, `steps(4)` |
| `animation-delay` | `0.2s` |
| `animation-iteration-count` | `1`, `infinite`, `3` |
| `animation-direction` | `normal`, `reverse`, `alternate` |
| `animation-fill-mode` | `none`, `forwards`, `backwards`, `both` |
| `animation-play-state` | `running`, `paused` |

### Пример: пульсирующая кнопка

```css
@keyframes pulse {
  0%   { transform: scale(1); }
  50%  { transform: scale(1.1); }
  100% { transform: scale(1); }
}
.pulse-btn {
  animation: pulse 2s ease-in-out infinite;
}
```

---

## 3. CSS-эффекты

### Filters

```css
.blur   { filter: blur(5px); }
.bright { filter: brightness(1.5); }
.shadow { filter: drop-shadow(2px 4px 6px black); }
.grayscale { filter: grayscale(100%); }
.sepia  { filter: sepia(60%); }
.hue    { filter: hue-rotate(90deg); }
.invert { filter: invert(100%); }
.blend  { filter: contrast(200%) saturate(120%); }
```

### Backdrop-filter

Эффекты на фоне элемента (стекло/матовость):

```css
.glass {
  background: rgba(255,255,255,0.1);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
}
```

### Clip-path

Обрезка элемента в произвольную форму:

```css
.circle { clip-path: circle(50%); }
.triangle { clip-path: polygon(50% 0%, 0% 100%, 100% 100%); }
.hexagon { clip-path: polygon(25% 0%, 75% 0%, 100% 50%, 75% 100%, 25% 100%, 0% 50%); }
```

### Blend Modes

```css
.blend-multiply    { mix-blend-mode: multiply; }
.blend-screen      { mix-blend-mode: screen; }
.blend-overlay     { mix-blend-mode: overlay; }
.blend-difference  { mix-blend-mode: difference; }
```

---

## 4. Scroll-driven анимации (CSS Scroll Timeline)

Новый стандарт (Chrome 115+):

```css
@keyframes fadeIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}

.scroll-anim {
  animation: fadeIn 1s linear;
  animation-timeline: scroll();
}
```

С поддержкой view():

```css
.reveal {
  animation: slideUp 0.6s ease-out;
  animation-timeline: view();
  animation-range: entry 0% entry 100%;
}
```

---

## 5. JavaScript-driven анимации

### Web Animations API (WAAPI)

```js
element.animate([
  { transform: 'translateX(0)', opacity: 1 },
  { transform: 'translateX(200px)', opacity: 0 }
], {
  duration: 1000,
  easing: 'ease-in-out',
  fill: 'forwards'
});
```

**Преимущества**: нативная производительность (та же, что и у CSS)

### requestAnimationFrame

Для кастомных анимаций:

```js
function animate() {
  element.style.transform = `translateX(${position}px)`;
  position += 1;
  if (position < 200) requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

---

## 6. Библиотеки анимаций

| Библиотека | Тип | Размер | Особенность |
|---|---|---|---|
| **Animate.css** | CSS классы | ~30KB | Готовые классы `animate__fadeIn` |
| **GSAP (GreenSock)** | JS | ~30KB | Профессиональная, timeline, scroll |
| **Framer Motion** | React | ~30KB | React-only, spring physics |
| **Motion (One)** | JS/React | ~15KB | Новая версия Framer (free) |
| **Lottie (dotLottie)** | JSON/JS | ~5KB | After Effects → веб |
| **Rive** | JS | ~5KB | State machine для анимаций |
| **Three.js** | WebGL | ~500KB | 3D анимации |
| **AOS (Animate On Scroll)** | CSS + JS | ~15KB | Scroll-driven |
| **ScrollReveal** | JS | ~5KB | Scroll анимации |
| **Mo.js** | JS | ~20KB | SVG анимации |
| **Hover.css** | CSS | ~20KB | Hover-эффекты |
| **Typed.js** | JS | ~10KB | Анимация печати текста |
| **Particles.js** | JS | ~30KB | Частицы, фон |
| **Vivus** | JS | ~10KB | Рисование SVG |

---

## 7. Практические техники

### Stagger (задержка для списка)

```css
.item:nth-child(1) { animation-delay: 0.0s; }
.item:nth-child(2) { animation-delay: 0.1s; }
.item:nth-child(3) { animation-delay: 0.2s; }
/* или через JS: style="--i: 2" и animation-delay: calc(var(--i) * 0.1s) */
```

### Spring animation (через WAAPI или библиотеки)

```js
element.animate([
  { transform: 'scale(0)' },
  { transform: 'scale(1.2)', easing: 'cubic-bezier(0.5, 1.6, 0.5, 0.7)' },
  { transform: 'scale(1)' }
], { duration: 400 });
```

### Skeleton loading

```css
@keyframes shimmer {
  0% { background-position: -200px 0; }
  100% { background-position: calc(200px + 100%) 0; }
}
.skeleton {
  background: linear-gradient(90deg, #eee 25%, #f5f5f5 50%, #eee 75%);
  background-size: 200px 100%;
  animation: shimmer 1.5s infinite;
}
```

---

## 8. Производительность анимаций

### Что нагружает GPU (быстро) vs CPU (медленно)

| ✅ GPU — быстро | ❌ CPU — медленно |
|---|---|
| `transform: translate/scale/rotate` | `width`, `height` |
| `opacity` | `top`, `left`, `margin` |
| `filter` (частично GPU) | `box-shadow` (перерисовка) |
| `clip-path` | `background` перерисовка |

### Практические советы

1. **Используйте `transform` и `opacity`** — они не вызывают reflow
2. **`will-change`** — подсказка браузеру: `will-change: transform, opacity`
3. **Не анимируйте `top/left`** — используйте `transform: translate()`
4. **Не анимируйте `width/height`** — используйте `transform: scale()`
5. **`contain: layout style paint`** — изоляция перерисовки
6. **Проверяйте через DevTools → Performance** — где перерисовка?

---

## 9. Доступность анимаций

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

Или через JS:

```js
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)');
if (prefersReducedMotion.matches) {
  // отключаем анимации
}
```

---

## Чек-лист

- [ ] Простые переходы → CSS transitions
- [ ] Сложные сценарии → CSS @keyframes
- [ ] Scroll-driven → CSS Scroll Timeline или AOS
- [ ] Тяжёлые анимации → GSAP / Framer Motion
- [ ] SVG/логотипы → Lottie / Rive
- [ ] 3D → Three.js
- [ ] GPU-friendly → transform + opacity
- [ ] Доступность → prefers-reduced-motion
