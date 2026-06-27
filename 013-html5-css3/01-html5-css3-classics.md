# HTML5 CSS3 Classics

---

## Semantic HTML5

```html
<!-- Структурные элементы -->
<header>
  <nav>
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/about">About</a></li>
    </ul>
  </nav>
</header>

<main>
  <article>
    <h1>Article Title</h1>
    <section>
      <h2>Section</h2>
      <p>Content...</p>
    </section>
  </article>
  
  <aside>Sidebar content</aside>
</main>

<footer>
  <address>Contact: email@example.com</address>
</footer>

<figure>
  <img src="photo.jpg" alt="Description" />
  <figcaption>Caption text</figcaption>
</figure>

<time datetime="2024-01-15">January 15</time>
<mark>Highlighted</mark>
<progress value="70" max="100">70%</progress>
```

**Факт:** Семантическая разметка улучшает SEO и accessibility (screen readers).

---

## CSS3 Selectors

```css
/* Attribute selectors */
[data-type="primary"] { }
[href^="https"] { }        /* starts with */
[src$=".jpg"] { }          /* ends with */
[class*="icon"] { }        /* contains */

/* Structural pseudo-classes */
li:first-child { }
li:last-child { }
li:nth-child(2n) { }       /* even */
li:nth-child(3n+1) { }     /* every 3rd starting from 1 */
li:nth-last-child(2) { }   /* 2nd from end */
p:only-child { }

/* Type selectors */
div:first-of-type { }
div:last-of-type { }
div:nth-of-type(2) { }

/* UI states */
input:disabled { }
input:checked + label { }
input:required { }
input:valid { border-color: green; }
input:invalid { border-color: red; }
input:focus-visible { outline: 2px solid blue; }

/* Negation */
:not(.disabled) { }
:not(:first-child) { }

/* Combinators */
div > p { }          /* child */
div + p { }          /* adjacent sibling */
div ~ p { }          /* general sibling */
```

---

## Flexbox

```css
.container {
  display: flex;
  flex-direction: row;         /* row | column | row-reverse | column-reverse */
  flex-wrap: wrap;             /* nowrap | wrap | wrap-reverse */
  justify-content: center;     /* flex-start | center | space-between | space-around | space-evenly */
  align-items: stretch;        /* stretch | center | flex-start | flex-end | baseline */
  align-content: center;       /* When wrapping */
  gap: 16px;                   /* Grid-gap for Flexbox */
}

.item {
  flex: 1 1 200px;             /* grow shrink basis */
  align-self: center;          /* Override align-items */
  order: -1;                   /* Reorder */
}
```

**Факт:** `gap` для Flexbox поддерживается с 2021 года (Firefox 63+, Chrome 84+).

---

## Grid

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);      /* 3 equal columns */
  grid-template-rows: auto 1fr auto;          /* header, content, footer */
  gap: 16px;
  
  /* Named areas */
  grid-template-areas:
    "header  header  header"
    "sidebar content content"
    "footer  footer  footer";
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.content { grid-area: content; }
.footer  { grid-area: footer; }

/* Auto-fit / Auto-fill */
.responsive-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  /* Automatically adjusts columns to viewport */
}

/* Centering (holy grail) */
.centered {
  display: grid;
  place-items: center;         /* align-items + justify-items */
}

/* Subgrid (Chrome 117+, Firefox 71+) */
.nested {
  display: grid;
  grid-template-columns: subgrid;
}
```

---

## Animations & Transitions

```css
/* Transitions — smooth property change */
.button {
  background: blue;
  transition: background 0.3s ease, transform 0.2s ease-out;
}
.button:hover {
  background: darkblue;
  transform: translateY(-2px);
}

/* Keyframe animations */
@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateX(-100%);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.animated {
  animation: slideIn 0.5s ease-out forwards;  /* forwards: keep end state */
  animation-delay: 0.1s;
  animation-iteration-count: 1;
  animation-direction: normal;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}

/* Transforms */
.transform-2d {
  transform: rotate(45deg) scale(1.2) translate(10px, 20px) skewX(10deg);
}
.transform-3d {
  transform: perspective(500px) rotateY(45deg) translateZ(100px);
  backface-visibility: hidden;
}
```

---

## Responsive Design

```css
/* Mobile-first */
.container { 
  width: 100%; 
  padding: 16px; 
}

/* Tablet */
@media (min-width: 768px) {
  .container { max-width: 720px; padding: 24px; }
}

/* Desktop */
@media (min-width: 1024px) {
  .container { max-width: 960px; }
}

/* Wide */
@media (min-width: 1280px) {
  .container { max-width: 1200px; }
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  body { background: #111; color: #eee; }
}

/* Reduced motion */
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}

/* Print */
@media print {
  nav, footer { display: none; }
  body { font-size: 12pt; }
}
```

---

## Чек-лист

- [ ] Semantic HTML5: header, nav, main, article, section, aside, footer
- [ ] CSS3 Selectors: nth-child, attribute, pseudo-classes, combinators
- [ ] Flexbox: direction, wrap, justify/align, gap, flex shorthand
- [ ] Grid: template, auto-fill/auto-fit, named areas, place-items
- [ ] Transitions: property, duration, timing, delay
- [ ] Animations: keyframes, animation shorthand
- [ ] Responsive: mobile-first, media queries, prefers-color-scheme
