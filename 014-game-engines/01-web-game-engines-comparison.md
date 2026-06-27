# Web Game Engines — Comparison

---

## Сравнительная таблица

| Engine | Type | Graphics | 3D | 2D | Physics | Language | File Size | Best for |
|---|---|---|---|---|---|---|---|---|
| **Three.js** | Library | WebGL/WebGPU | ✅✅✅ | ✅ | ❌ (addon) | JS/TS | ~500KB | 3D visualizations, configurators |
| **Babylon.js** | Framework | WebGL/WebGPU | ✅✅✅ | ✅ | ✅ (cannon) | JS/TS | ~2MB | 3D games, XR, complex scenes |
| **Phaser** | Framework | WebGL/Canvas | ❌ | ✅✅✅ | ✅ (matter) | JS/TS | ~1MB | 2D games, mobile web |
| **PixiJS** | Renderer | WebGL/WebGPU | ❌ | ✅✅✅ | ❌ (addon) | JS/TS | ~400KB | High-perf 2D rendering |
| **PlayCanvas** | Platform | WebGL | ✅✅ | ✅ | ✅ (ammo) | JS | Cloud | Collaborative 3D |
| **Unity (WebGL)** | Full Engine | WebGL (via Emscripten) | ✅✅✅ | ✅✅ | ✅ (physX) | C# | ~5-20MB | AAA port, complex 3D |
| **Godot (Wasm)** | Full Engine | WebGL | ✅✅ | ✅✅ | ✅ (built-in) | GDScript/C# | ~3-10MB | Open-source, 2D/3D |
| **Cocos Creator** | Framework | WebGL/Canvas | ✅ | ✅✅✅ | ✅ (built-in) | TS | ~1-3MB | 2D mobile games |

---

## Three.js — 3D визуализации

```javascript
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
const renderer = new THREE.WebGLRenderer({ antialias: true })
renderer.setSize(window.innerWidth, window.innerHeight)
document.body.appendChild(renderer.domElement)

// Geometry
const geometry = new THREE.BoxGeometry(1, 1, 1)
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 })
const cube = new THREE.Mesh(geometry, material)
scene.add(cube)

// Lighting
const light = new THREE.DirectionalLight(0xffffff, 1)
light.position.set(5, 10, 7)
scene.add(light)
scene.add(new THREE.AmbientLight(0x404040))

const controls = new OrbitControls(camera, renderer.domElement)
camera.position.z = 5

// Animation loop
function animate() {
  requestAnimationFrame(animate)
  cube.rotation.x += 0.01
  cube.rotation.y += 0.01
  controls.update()
  renderer.render(scene, camera)
}
animate()
```

**Факт:** Three.js — самая популярная 3D библиотека (200K+ GitHub stars). Не full engine, а low-level 3D library.

---

## Babylon.js — 3D framework

```javascript
import { Engine, Scene, ArcRotateCamera, HemisphericLight, MeshBuilder, Vector3 } from '@babylonjs/core'

const canvas = document.getElementById('renderCanvas')
const engine = new Engine(canvas, true)
const scene = new Scene(engine)

const camera = new ArcRotateCamera('camera', -Math.PI / 2, Math.PI / 2.5, 10)
camera.setTarget(Vector3.Zero())
camera.attachControl(canvas, true)

const light = new HemisphericLight('light', new Vector3(0, 1, 0), scene)
const sphere = MeshBuilder.CreateSphere('sphere', { diameter: 2 }, scene)

// Physics
import '@babylonjs/core/Physics/physicsEngine'
scene.enablePhysics(new Vector3(0, -9.81, 0), new CannonJSPlugin())

engine.runRenderLoop(() => scene.render())
```

**Факт:** Babylon.js имеет встроенную поддержку **WebXR** (VR/AR).

---

## Phaser — 2D игры

```javascript
const config = {
  type: Phaser.AUTO,
  width: 800,
  height: 600,
  physics: { default: 'arcade', arcade: { gravity: { y: 300 } } },
  scene: { preload, create, update }
}

const game = new Phaser.Game(config)

function preload() {
  this.load.image('sky', 'assets/sky.png')
  this.load.image('platform', 'assets/platform.png')
  this.load.spritesheet('player', 'assets/player.png', { frameWidth: 32, frameHeight: 48 })
}

function create() {
  this.add.image(400, 300, 'sky')
  
  platforms = this.physics.add.staticGroup()
  platforms.create(400, 568, 'platform').setScale(2).refreshBody()
  
  player = this.physics.add.sprite(100, 450, 'player')
  player.setBounce(0.2)
  player.setCollideWorldBounds(true)
  
  this.physics.add.collider(player, platforms)
  cursors = this.input.keyboard.createCursorKeys()
}

function update() {
  if (cursors.left.isDown) player.setVelocityX(-160)
  else if (cursors.right.isDown) player.setVelocityX(160)
  else player.setVelocityX(0)
  
  if (cursors.up.isDown && player.body.touching.down)
    player.setVelocityY(-330)
}
```

---

## PixiJS — высокопроизводительный 2D рендеринг

```javascript
import { Application, Graphics, Text, TextStyle } from 'pixi.js'

const app = new Application({
  width: 800,
  height: 600,
  backgroundColor: 0x1099bb,
  antialias: true,
  resolution: window.devicePixelRatio || 1
})
document.body.appendChild(app.view)

// Graphics
const graphics = new Graphics()
graphics.beginFill(0xff0000)
graphics.drawRoundedRect(100, 100, 100, 100, 16)
graphics.endFill()
app.stage.addChild(graphics)

// Text
const style = new TextStyle({ fontFamily: 'Arial', fontSize: 36, fill: 'white' })
const text = new Text('Hello PixiJS!', style)
text.x = 50
text.y = 50
app.stage.addChild(text)

// Sprite
const sprite = Sprite.from('image.png')
sprite.anchor.set(0.5)
sprite.x = app.screen.width / 2
sprite.y = app.screen.height / 2
app.stage.addChild(sprite)

app.ticker.add(() => {
  sprite.rotation += 0.01
})
```

---

## Unity WebGL — AAA портирование

**Факты:**
- Компилирует C# → IL2CPP → WebAssembly
- Полный engine в браузере (физика, audio, particles, shaders)
- **Размер:** 5-20 MB (initial download)
- **Требования:** WebGL 2.0, WASM
- **Ограничения:** threading (SharedArrayBuffer требует COOP/COEP headers), файловая система (IDBFS)

```javascript
// Unity WebGL — JavaScript integration
var unityInstance = UnityLoader.instantiate('unity-container', 'Build/Game.json', {
  onProgress: (progress) => console.log(`${progress * 100}%`),
  Module: {
    onRuntimeInitialized: () => {
      unityInstance.SendMessage('GameManager', 'StartGame', 'level1')
    }
  }
})

// Send message to Unity
unityInstance.SendMessage('GameManager', 'SetScore', 100)

// Receive from Unity
function onGameOver(score) {
  console.log(`Game over! Score: ${score}`)
}
```

---

## Godot — open-source альтернатива

**Факт:** Godot 4+ экспортирует в WebAssembly через WebGL/WebGPU. GDScript компилируется в bytecode, C# через WASM.

| Aspect | Unity WebGL | Godot Wasm | PlayCanvas |
|---|---|---|---|
| **Build size** | 5-20 MB | 3-10 MB | Cloud |
| **Performance** | High | Medium-High | Medium |
| **Editor** | Desktop | Desktop | Browser |
| **Open Source** | ❌ | ✅ | ❌ |
| **C# support** | ✅ | ✅ (4+) | ❌ |
| **2D** | ✅ | ✅✅ (best-in-class) | ✅ |
| **3D** | ✅✅✅ | ✅✅ | ✅✅ |
| **Price** | Free (indie) | Free | Free (limited) |

---

## Чек-лист

- [ ] Three.js: 3D library, WebGL/WebGPU, 200K+ stars
- [ ] Babylon.js: 3D framework, WebXR, physics
- [ ] Phaser: 2D game framework, Arcade/Matter physics
- [ ] PixiJS: 2D renderer, GPU-accelerated, lightweight
- [ ] WebGL build: Unity (AAA), Godot (open-source), PlayCanvas (cloud)
- [ ] Choose by: 2D vs 3D, performance needs, team skills
