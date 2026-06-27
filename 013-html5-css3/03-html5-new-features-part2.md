# HTML5 New Features Part 2 — с 2020

---

## Web Components

**Факты:**
- Набор стандартов: **Custom Elements**, **Shadow DOM**, **HTML Templates**
- Framework-agnostic (работает в любом HTML)
- Поддержка: Chrome, Firefox, Safari, Edge

```javascript
// 1. Custom Element
class UserCard extends HTMLElement {
  static observedAttributes = ['user-id', 'name']
  
  constructor() {
    super()
    this.attachShadow({ mode: 'open' })  // Shadow DOM
  }
  
  connectedCallback() {
    this.render()
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue !== newValue) this.render()
  }
  
  render() {
    this.shadowRoot.innerHTML = `
      <style>
        .card { border: 1px solid #ddd; padding: 16px; border-radius: 8px; }
        .name { font-weight: bold; }
      </style>
      <div class="card">
        <div class="name">${this.getAttribute('name')}</div>
        <slot></slot>
      </div>
    `
  }
}

customElements.define('user-card', UserCard)

// HTML: <user-card user-id="1" name="John"><p>Additional content</p></user-card>

// 2. HTML Templates
const template = document.createElement('template')
template.innerHTML = `<button class="btn"><slot></slot></button>`

class MyButton extends HTMLElement {
  constructor() {
    super()
    this.attachShadow({ mode: 'open' })
    this.shadowRoot.appendChild(template.content.cloneNode(true))
  }
}
```

**Факт:** Shadow DOM обеспечивает **изоляцию стилей** — CSS не проникает внутрь и наружу.

---

## CSS Container Queries

**Факт:** Container queries позволяют реагировать на **размер контейнера**, а не viewport.

```css
/* Define containment context */
.card-container {
  container-type: inline-size;       /* Respond to width */
  container-name: card;              /* Named container */
}

/* Query by container width */
@container card (min-width: 400px) {
  .card { 
    display: grid;
    grid-template-columns: 200px 1fr;
  }
  .card-image { order: -1; }
}

@container card (max-width: 399px) {
  .card { display: flex; flex-direction: column; }
  .card-image { width: 100%; }
}

/* Style queries (query by property) */
@container card style(--variant: featured) {
  .card { border-color: gold; }
}

/* Container query units */
.card { 
  font-size: clamp(1rem, 5cqi, 2rem);  /* 5% of container inline-size */
  padding: 2cqw;                        /* 2% of container width */
}
```

**Факт:** Container queries — **ключевой прорыв** в responsive design. Поддержка: Chrome 105+, Firefox 110+, Safari 16+.

---

## WebGPU

**Факт:** WebGPU — современный API для GPU (замена WebGL). Низкоуровневый доступ к GPU.

```javascript
async function initGPU() {
  if (!navigator.gpu) throw new Error('WebGPU not supported')
  
  const adapter = await navigator.gpu.requestAdapter()
  const device = await adapter.requestDevice()
  
  const shaderCode = `
    @vertex fn vs(@builtin(vertex_index) idx: u32) -> @builtin(position) vec4<f32> {
      let pos = array(vec2(0.0, 0.5), vec2(-0.5, -0.5), vec2(0.5, -0.5));
      return vec4<f32>(pos[idx], 0.0, 1.0);
    }
    @fragment fn fs() -> @location(0) vec4<f32> {
      return vec4(1.0, 0.0, 0.0, 1.0);
    }
  `
  
  const shader = device.createShaderModule({ code: shaderCode })
  // ... pipeline, render pass, draw
}
```

**Факт:** WebGPU даёт доступ к compute shaders (не только rendering). Используется для ML inference, физики, обработки изображений.

---

## WebAssembly (Wasm)

```javascript
// Rust → Wasm компиляция
// rustup target add wasm32-unknown-unknown
// cargo build --target wasm32-unknown-unknown --release

// JavaScript
async function loadWasm() {
  const response = await fetch('module.wasm')
  const bytes = await response.arrayBuffer()
  const { instance } = await WebAssembly.instantiate(bytes, {
    env: {
      print: (ptr, len) => {
        const str = new TextDecoder().decode(
          new Uint8Array(memory.buffer, ptr, len)
        )
        console.log(str)
      }
    }
  })
  
  const result = instance.exports.add(2, 3)
  console.log('Wasm result:', result)
}

// WASM GC (garbage collection for languages like Kotlin/Dart)
// .NET WASM (Blazor WebAssembly)
```

**Факт:** Wasm работает **близко к native скорости**. Подходит для: games, image/video processing, cryptography, emulators.

---

## Service Workers & PWA

```javascript
// sw.js — Service Worker
const CACHE_NAME = 'v1'
const ASSETS = ['/', '/index.html', '/app.js', '/style.css']

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(ASSETS))
  )
})

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      const fetchPromise = fetch(event.request).then(response => {
        caches.open(CACHE_NAME).then(cache => cache.put(event.request, response.clone()))
        return response
      })
      return cached || fetchPromise  // Network-first, fallback to cache
    })
  )
})

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys => 
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
    )
  )
})

// manifest.json — PWA manifest
{
  "name": "My PWA",
  "short_name": "PWA",
  "start_url": "/",
  "display": "standalone",        // Fullscreen PWA
  "background_color": "#ffffff",
  "theme_color": "#2196f3",
  "icons": [{ "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" }]
}
```

---

## WebRTC

```javascript
// Peer-to-peer communication
const pc = new RTCPeerConnection({
  iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
})

// Add local stream
const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true })
stream.getTracks().forEach(track => pc.addTrack(track, stream))

// Create offer
const offer = await pc.createOffer()
await pc.setLocalDescription(offer)
// Send offer to peer via signaling server

// Receive answer
await pc.setRemoteDescription(new RTCSessionDescription(answer))

// Data channel (peer-to-peer file/data)
const dataChannel = pc.createDataChannel('chat')
dataChannel.onmessage = (e) => console.log('Received:', e.data)
dataChannel.send('Hello peer!')
```

---

## CSS Layers (@layer)

```css
/* Control cascade order */
@layer reset, base, components, utilities;

@layer reset {
  * { margin: 0; padding: 0; }
}

@layer base {
  body { font-family: system-ui; line-height: 1.5; }
}

@layer components {
  .card { padding: 16px; border-radius: 8px; }
}

@layer utilities {
  .hidden { display: none; }
}

/* Layer order: reset < base < components < utilities */
/* Components override reset, utilities override everything */
```

---

## Чек-лист

- [ ] Web Components: Custom Elements, Shadow DOM, Templates, Slots
- [ ] Container Queries: container-type, @container, style queries
- [ ] WebGPU: GPU adapter/device, compute shaders
- [ ] WebAssembly: wasm modules, near-native performance
- [ ] Service Workers: cache, fetch intercept, PWA manifest
- [ ] WebRTC: RTCPeerConnection, STUN/TURN, DataChannel
- [ ] CSS Layers: @layer cascade control
