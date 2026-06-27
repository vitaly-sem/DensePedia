# HTML5 New Features Part 1 — до 2020

---

## Canvas

```html
<canvas id="myCanvas" width="800" height="600"></canvas>
```

```javascript
const canvas = document.getElementById('myCanvas')
const ctx = canvas.getContext('2d')

// Drawing
ctx.fillStyle = 'red'
ctx.fillRect(10, 10, 100, 50)

ctx.strokeStyle = 'blue'
ctx.lineWidth = 2
ctx.strokeRect(120, 10, 100, 50)

ctx.beginPath()
ctx.arc(300, 35, 30, 0, Math.PI * 2)
ctx.fillStyle = 'green'
ctx.fill()

// Text
ctx.font = '24px Arial'
ctx.fillText('Hello Canvas', 400, 40)

// Transform
ctx.save()
ctx.translate(100, 200)
ctx.rotate(Math.PI / 4)
ctx.fillRect(0, 0, 50, 50)
ctx.restore()

// Image
const img = new Image()
img.onload = () => ctx.drawImage(img, 0, 300)
img.src = 'image.jpg'

// Pixel manipulation
const imageData = ctx.getImageData(0, 0, 100, 100)
for (let i = 0; i < imageData.data.length; i += 4) {
  imageData.data[i] = 255 - imageData.data[i]      // Red invert
  imageData.data[i + 1] = 255 - imageData.data[i + 1]  // Green invert
  imageData.data[i + 2] = 255 - imageData.data[i + 2]  // Blue invert
}
ctx.putImageData(imageData, 0, 0)
```

**Факт:** Canvas — immediate mode (не DOM). Быстрее SVG для большого числа объектов (> 1000).

---

## SVG

```html
<svg viewBox="0 0 200 100" xmlns="http://www.w3.org/2000/svg">
  <circle cx="50" cy="50" r="40" fill="red" stroke="black" stroke-width="2" />
  <rect x="100" y="10" width="80" height="80" rx="8" fill="blue" />
  <path d="M 10 80 Q 52.5 10, 95 80 T 180 80" fill="none" stroke="green" stroke-width="3" />
  
  <defs>
    <linearGradient id="grad1" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" stop-color="yellow" />
      <stop offset="100%" stop-color="red" />
    </linearGradient>
  </defs>
  <ellipse cx="100" cy="50" rx="90" ry="30" fill="url(#grad1)" />
</svg>
```

**Факт:** SVG — retained mode (DOM-элементы). Лучше для иконок, логотипов, интерактивных элементов.

---

## WebSockets

```javascript
const ws = new WebSocket('wss://api.example.com/ws')

ws.onopen = () => {
  console.log('Connected')
  ws.send(JSON.stringify({ type: 'subscribe', channel: 'prices' }))
}

ws.onmessage = (event) => {
  const data = JSON.parse(event.data)
  updatePrice(data.symbol, data.price)
}

ws.onclose = (event) => {
  if (!event.wasClean) {
    // Reconnect
    setTimeout(connect, 1000)
  }
}

ws.onerror = (error) => console.error('WebSocket error', error)
```

**Факт:** WebSocket поддерживает **binary frames** (Blob/ArrayBuffer), не только текст.

---

## Web Workers

```javascript
// main.js
const worker = new Worker('worker.js')

worker.postMessage({ type: 'process', data: largeArray })

worker.onmessage = (event) => {
  console.log('Result:', event.data)
  worker.terminate()
}

worker.onerror = (error) => console.error('Worker error', error)

// worker.js
self.onmessage = (event) => {
  const { data } = event.data
  const result = heavyComputation(data)
  self.postMessage(result)
}

// SharedWorker (shared between tabs)
const sharedWorker = new SharedWorker('shared-worker.js')
sharedWorker.port.postMessage('ping')
sharedWorker.port.onmessage = (e) => console.log(e.data)
```

**Факт:** Workers имеют свой V8 isolate — не блокируют UI thread.
**Факт:** Shared Workers разделяются между вкладками одного origin.

---

## LocalStorage & IndexedDB

```javascript
// LocalStorage (синхронный, 5-10 MB)
localStorage.setItem('theme', 'dark')
const theme = localStorage.getItem('theme')
localStorage.removeItem('theme')
localStorage.clear()

// IndexedDB (асинхронный, практически безлимитный)
const request = indexedDB.open('AppCache', 1)

request.onupgradeneeded = (event) => {
  const db = event.target.result
  const store = db.createObjectStore('users', { keyPath: 'id' })
  store.createIndex('email', 'email', { unique: true })
}

request.onsuccess = (event) => {
  const db = event.target.result
  
  const transaction = db.transaction(['users'], 'readwrite')
  const store = transaction.objectStore('users')
  
  store.add({ id: 1, name: 'John', email: 'john@example.com' })
  
  const getRequest = store.get(1)
  getRequest.onsuccess = () => console.log(getRequest.result)
}
```

---

## Other APIs

```javascript
// Drag & Drop
element.draggable = true
element.ondragstart = (e) => e.dataTransfer.setData('text/plain', item.id)
dropZone.ondragover = (e) => e.preventDefault()
dropZone.ondrop = (e) => {
  e.preventDefault()
  const id = e.dataTransfer.getData('text/plain')
  handleDrop(id)
}

// Geolocation
navigator.geolocation.getCurrentPosition(
  (pos) => console.log(pos.coords.latitude, pos.coords.longitude),
  (err) => console.error(err),
  { enableHighAccuracy: true, timeout: 5000 }
)

// History API (SPA routing)
history.pushState({ page: 'home' }, '', '/home')
window.onpopstate = (event) => navigate(event.state.page)

// Web Audio
const audioCtx = new AudioContext()
const oscillator = audioCtx.createOscillator()
oscillator.type = 'sine'
oscillator.frequency.setValueAtTime(440, audioCtx.currentTime)
oscillator.connect(audioCtx.destination)
oscillator.start()
```

---

## Чек-лист

- [ ] Canvas: 2D context, immediate mode, pixel manipulation
- [ ] SVG: retained mode, viewBox, gradients, paths
- [ ] WebSockets: binary frames, reconnection
- [ ] Web Workers: separate thread, SharedWorker, limitations (no DOM)
- [ ] Storage: LocalStorage (sync, 5MB), IndexedDB (async, unlimited)
- [ ] History API: pushState, popstate (SPA routing)
- [ ] Drag & Drop: dataTransfer, drag events
- [ ] Geolocation: permissions, accuracy
