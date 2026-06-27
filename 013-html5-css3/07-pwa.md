# Progressive Web Apps (PWA)

> **Технология**: Веб-приложения с возможностями нативных приложений
> **Стандарт**: W3C, Google (инициатор)
> **Поддержка**: Chrome, Edge, Firefox, Safari (частично), Samsung Internet

---

## Что такое PWA?

**Progressive Web App** — веб-сайт, который ведёт себя как нативное приложение:

- 🔔 Push-уведомления
- 📱 Иконка на домашнем экране
- 📡 Офлайн-режим
- 🚀 Быстрая загрузка (даже на slow network)
- 🔄 Фоновые обновления

---

## Три кита PWA

### 1. Service Worker

Скрипт, работающий в фоновом потоке (отдельно от страницы).

```js
// service-worker.js
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('v1').then(cache => {
      return cache.addAll(['/', '/index.html', '/app.js', '/style.css']);
    })
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(response => {
      return response || fetch(event.request);
    })
  );
});
```

**Жизненный цикл Service Worker**:
```
Register → Install → Activate → Fetch (ожидание)
                  ↓             ↑
                Ошибка      Обновление (waiting)
```

### 2. Web App Manifest

JSON-файл, описывающий приложение:

```json
{
  "name": "My PWA",
  "short_name": "PWA",
  "description": "Progressive Web App",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

### 3. HTTPS

Service Worker работает только через HTTPS (или localhost).

---

## Ключевые API

| API | Назначение | Поддержка |
|---|---|---|
| **Cache API** | Кэширование ресурсов | ✅ Все |
| **IndexedDB** | Клиентская база данных | ✅ Все |
| **Push API** | Push-уведомления | Chrome, Edge, Firefox |
| **Notification API** | Отображение уведомлений | ✅ Все |
| **Background Sync** | Отложенная отправка данных | Chrome, Edge |
| **Periodic Background Sync** | Периодическая синхронизация | Chrome (Android) |
| **Web Share API** | Шеринг (нативный) | Chrome, Safari |
| **Badging API** | Значок уведомления на иконке | Chrome, Edge |
| **File System Access** | Доступ к файловой системе | Chrome |
| **Idle Detection** | Определение бездействия | Chrome |

---

## PWA vs Нативные приложения

| Критерий | PWA | Нативное приложение |
|---|---|---|
| **Установка** | Через браузер | App Store / Google Play |
| **Размер** | КБ–МБ | 10–500 МБ |
| **Офлайн** | ✅ Service Worker | ✅ |
| **Push-уведомления** | ✅ | ✅ |
| **Доступ к NFC/Bluetooth** | ❌ | ✅ |
| **Доступ к файловой системе** | 🔜 (частично) | ✅ |
| **Стоимость публикации** | Бесплатно | $99/год (Apple), $25 (Google) |
| **Обновления** | Мгновенно | Через магазин |
| **SEO** | Индексируются | Нет |
| **Производительность** | Почти native | Native |
| **Доля рынка (пример)** | Twitter Lite: +250% твитов | Twitter native |

---

## Известные PWA

| PWA | Результат после внедрения |
|---|---|
| **Twitter/X Lite** | +250% твитов, -80% размера |
| **Pinterest** | +40% времени на сайте |
| **Uber** | Быстрый заказ даже в 2G |
| **Spotify** | Мгновенная загрузка, офлайн |
| **Telegram** | Десктоп-версия как PWA |
| **Starbucks** | Офлайн-заказы |
| **Tinder** | Ускорение загрузки в 2x |
| **OLX** | Увеличение конверсии на 30% |

---

## Lighthouse PWA Audit

**Минимальные требования** (Lighthouse score > 80):

- [ ] **Service Worker** зарегистрирован
- [ ] **HTTPS** включён
- [ ] **Manifest** c start_url, display, icons
- [ ] **Отзывчивый дизайн** (mobile-friendly)
- [ ] **Быстрая загрузка** (FCP < 3s, TTI < 5s)
- [ ] **Работа в офлайн** (хотя бы страница с сообщением)
- [ ] **Скроллинг** без задержки (60fps)
- [ ] **Переходы** SPA-стиля (переход без перезагрузки)

---

## Safari PWA — статус

| Возможность | Статус (iOS 17) |
|---|---|
| Установка на Home Screen | ✅ |
| Push-уведомления | ✅ (с 2023) |
| Badging | ✅ |
| Service Worker | ✅ |
| Background Sync | ❌ |
| Web Share API | ✅ |
| File System Access | ❌ |
| Periodic Background Sync | ❌ |

> **Важно**: На iOS PWA нельзя отправить push без открытия Safari. Установка PWA не так видна, как экран «Добавить на экран домой».

---

## Инструменты

| Инструмент | Назначение |
|---|---|
| **Workbox** (Google) | Библиотека для Service Worker (кэширование, синхронизация) |
| **Lighthouse** | Аудит PWA (в DevTools) |
| **PWA Builder** (Microsoft) | Генерация manifest, service worker |
| **Manifest Generator** | Генератор manifest.json |
| **PWABuilder.com** | Упаковка PWA в App Store/Google Play |
| **Custom Tabs (Chrome)** | Эмуляция браузерного UI |
| **Trusted Web Activity (TWA)** | PWA в Google Play как нативное приложение |
