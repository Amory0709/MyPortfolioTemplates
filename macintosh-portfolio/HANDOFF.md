# Portfolio · mac-portfolio — Handoff

> 给后续接手的人（agent / 自己 / 别人）。覆盖从"大白方块 bug"到"6 canvas 全部跑起来"的全过程。

---

## TL;DR

`/Users/hihihan/projects/mac-portfolio/` 是一个 portfolio 单页，原版 hero + 5 张项目卡都用 `<img>` 显示预渲染图（PNG mockup），结果缩放或加载时出现大白方块 / 图片丢失。**改成 6 个 `<canvas>` + Three.js 实时渲染 GLB 模型**：1 个 hero（Macintosh 128K 3D 模型）+ 5 个 floppy 软盘（每个项目一种颜色）。页面背景 `#f5f5f5` 直接透过透明 canvas 显示，**没有白方块了**。

| 验证项 | 结果 |
|---|---|
| 加载时间 | 1110ms（首次） |
| JS errors | 0 |
| Failed requests | 0（three.js + 6 GLB 全 200） |
| Console warnings | 4（都是 `GPU stall due to ReadPixels`，SwiftShader 软渲才有，真 GPU 无） |
| hero3d 尺寸 | 720×450 ✅ |
| floppy-3d ×5 尺寸 | 226×301 (3:4 portrait) ✅ |
| WebGL | OK |

---

## 项目结构

```
/Users/hihihan/projects/mac-portfolio/
├── index.html            ← 600 行，当前 live 版本（canvas 版）
├── index.html.bak        ← 390 行，原版（<img> 版，保留作对照）
├── models/
│   ├── macintosh_classic.glb     50KB  (Daz "Macintosh 128K Computer 1984" CC BY-NC 4.0)
│   ├── Computer_Retro.glb        294KB
│   ├── ComputerScreen_Retro.glb  1.3MB
│   ├── Keyboard_Retro.glb        1.4MB
│   ├── FloppyDisk_Retro.glb      370KB
│   ├── floppy_box.glb            233KB
│   ├── floppy_disk.glb           853KB
│   └── floppies/
│       ├── floppy_red.glb         3.5MB  ← Sound of Humanity
│       ├── floppy_orange.glb      3.5MB  ← Electoral Map
│       ├── floppy_yellow.glb      3.5MB  ← BLUEbikes
│       ├── floppy_green.glb       3.5MB  ← 3D Asset Editor
│       ├── floppy_blue.glb        3.5MB  ← MeshBVH X-Ray
│       ├── floppy_cream.glb       3.5MB
│       └── floppy_beige.glb       3.5MB
├── img/                  ← 原 PNG 资源（hero-mockup 1.3MB + 5 个 floppy 60KB）
├── textures/
├── screenshots/
├── daz_ref.jpg           ← Mac 模型参考图
├── preview.html / preview_v2.html     早期 preview 页面
├── snap_*.html / floppy_*.html        各阶段 snapshot / palette 实验
└── HANDOFF.md            ← 本文件
```

> 不是 git repo。改动靠 `index.html.bak` 做版本对照，没有真正的历史。

---

## 怎么跑

```bash
cd /Users/hihihan/projects/mac-portfolio
python3 -m http.server 8765 --bind 127.0.0.1
# 打开 http://127.0.0.1:8765/index.html
```

> **必须 HTTP，不能 `file://`** — importmap + ES module 需要 origin。localhost 是干净的，跨域 ok。

当前 server PID 3225（Python http.server），后台 nohup 跑着；如需重启：

```bash
lsof -ti :8765 | xargs kill
cd /Users/hihihan/projects/mac-portfolio && nohup python3 -m http.server 8765 --bind 127.0.0.1 > /tmp/mac-portfolio-8765.log 2>&1 &
```

---

## 改动总览（index.html.bak → index.html）

### 删的

- `<img class="hero-img" src="./img/hero-mockup.png" />`
- 5 个 `<img src="./img/floppies/floppy_*.png" />`

### 加的

| 块 | 说明 |
|---|---|
| 6 个 `<canvas>` | `hero3d` + 5 个 `.floppy-3d`（每个 `data-floppy="0..4"`） |
| `<script type="importmap">` | 把 `three` + `three/addons/` 指向 `unpkg.com/three@0.160.0` |
| `<script type="module">` | 主逻辑 ES module |
| `ModelView` class | 一个 GLB 一个透明 canvas：parallax + idle 微动 + hover 加 yaw |
| `makeRenderer` / `makeScene` / `getEnv` | 共用 WebGL 基础设施（共享 env map via WeakMap） |
| Side panel wiring | hover floppy → 右侧滑出详情 + 该 canvas 进入 hover 状态 |

### 关键设计决策

1. **透明 canvas** — `WebGLRenderer({ alpha: true })` + **不设 `scene.background`**，让页面 `#f5f5f5` 直接透出。这是修白方块的核心。
2. **每个 canvas 一个 renderer** — 共用 env map（WeakMap cache），不共 renderer（每个 canvas 独立 setSize，避免 resize 互相打架）。
3. **No `<canvas>` fallback 走纯 CSS** — `.no-webgl .no-webgl-fallback { display: flex }`，WebGL init 失败时父元素加 `no-webgl` 类，显示 "WebGL unavailable"。
4. **`outputColorSpace = SRGBColorSpace` + `ACESFilmicToneMapping`** — GLB 在 light bg 上不死黑。
5. **roughness 兜底** — 遍历 mesh 时如果 `material.roughness < 0.5` 强行拉到 0.55，让光面模型在 light bg 上有可读高光。

### Project data（驱动 side panel + floppy 顺序）

```js
const projects = [
  { title: 'Sound of Humanity',      color: 'red',    ..., link: 'https://amory0709.github.io/datavis/soundofhumanity/' },
  { title: 'Electoral Map',          color: 'orange', ..., link: 'https://amory0709.github.io/datavis/Electoral/' },
  { title: 'BLUEbikes Availability', color: 'yellow', ..., link: 'https://amory0709.github.io/datavis/BLUEbikes/' },
  { title: '3D Asset Editor',        color: 'green',  ..., link: 'https://amory0709.github.io/3d-editor/' },
  { title: 'MeshBVH X-Ray',          color: 'blue',   ..., link: 'https://amory0709.github.io/three-mesh-bvh-xray/' }
];
```

`projects[i].color` 决定 `floppy_${color}.glb`。**顺序 = 项目卡顺序 = side panel 顺序。**

### ModelView 参数表

| 实例 | glb | fov | cam | fit | parallax | idleSpin | modelY |
|---|---|---|---|---|---|---|---|
| hero3d | `macintosh_classic.glb` | 32 | [0.95, 0.55, 1.6] | 0.95 | 0.20 | 0.25 | 0.05 |
| 5× floppy | `floppy_${color}.glb` | 28 | [0.5, 0.35, 1.1] | 1.55 | 0.35 | 0.40 | 0 |

> **`fit` 是关键** — `0.95` for Mac（占满 hero），`1.55` for floppies（软盘小，要放大）。`fit` 表示 max(bbox.x, bbox.y, bbox.z) 缩放后的大小（场景单位）。

---

## 已知问题 / 未解决

### 🔴 性能：5 个 floppy GLB 每个 3.5MB（共 ~17.5MB）

每个 GLB 里塞了 4K 贴图。首次打开下载耗时。需要优化，三选一：

| 选项 | 预期 | 工作量 | 用户偏好 |
|---|---|---|---|
| 1. 重新导出低贴图版（diffuse → 1024²） | 每 ~400KB，总 2MB | 30min（Blender 重新 bake） | ? |
| 2. DRACO 压缩 | 再降 60-70% | 1h（`gltf-transform` 或 Blender） | ? |
| 3. lazy-load（IntersectionObserver） | 字节不变，加载体感改善 | 15min（加 IO + 占位 placeholder） | ? |

**用户没选，先保留方案 1+3 组合（重新 bake + IO lazy-load）作为后续默认。**

### 🟡 Hero 当前用 `macintosh_classic.glb`（50KB，简化版）

完整模型 `Computer_Retro.glb` + `Keyboard_Retro.glb` + `ComputerScreen_Retro.glb` 都在目录里没接进来。如果想做完整 Mac 拼装动画（屏幕能亮、键盘能插），需要把它们都 load 进同一个 modelRoot 然后分组 transform。

### 🟡 Mobile responsive

```css
@media (max-width: 600px) { .floppy-row { grid-template-columns: repeat(5, 1fr); gap: 6px; } }
```
手机上是 5 列 6px gap，软盘会非常小且可能 overflow。**还没在真机测过。**

### 🟡 `models/` 目录里多出来的 glb（ComputerScreen / Keyboard / FloppyDisk_Retro）没用上

是早期想过要拼 Mac 但没做的残留。要么删，要么用上。

### 🟢 平台兼容性

- macOS Safari / Chrome / Firefox：验证 OK（hero 720×450，floppy 226×301）
- Linux/Win 没测
- SwiftShader 软渲有 4 个 GPU stall warning（`ReadPixels`），真 GPU 无 — 不影响真机

---

## 调试 tips

| 现象 | 排查 |
|---|---|
| canvas 全黑 / 不显示 | 检查 `scene.background` 没设；或父元素被加了 `.no-webgl`（WebGL init 失败） |
| 模型巨大 / 超出画布 | 调 `opts.fit`（小→放大，大→缩小） |
| 模型位置偏下 / 偏上 | 调 `opts.modelY`（向上 = 正） |
| light bg 上模型太暗 | 调 `roughness` 兜底（当前 0.55），或 `environmentIntensity`（当前 0.55） |
| 模型加载失败 | `console.warn('GLB load failed', url, err)`，看 Network 面板 |
| importmap 404 | 检查 `unpkg.com/three@0.160.0/build/three.module.js` 通不通 |

---

## 跟其他项目的联系

5 个 floppy 链接到：

| Floppy | 链接 | 在 IO homepage cards 里 |
|---|---|---|
| Sound of Humanity | `amory0709.github.io/datavis/soundofhumanity/` | ✓ 有 card |
| Electoral Map | `amory0709.github.io/datavis/Electoral/` | ✓ 有 card |
| BLUEbikes | `amory0709.github.io/datavis/BLUEbikes/` | ✓ 有 card |
| 3D Asset Editor | `amory0709.github.io/3d-editor/` | ✓ 有 card（IO 主页 7 张之一） |
| MeshBVH X-Ray | `amory0709.github.io/three-mesh-bvh-xray/` | ✓ 有 card（IO 主页 7 张之一） |

> `amory0709.github.io` 主页的 7 张 IO cards 跟这个 portfolio 的 5 张 floppy 是**两套独立的视觉**，**内容重叠但设计不同**。没有联动代码。

---

## 如果从零重建

最小可运行骨架：

```html
<!doctype html>
<html><head><style>body{background:#f5f5f5;margin:0}</style></head>
<body>
<canvas id="c" style="width:400px;height:300px"></canvas>

<script type="importmap">
{"imports":{"three":"https://unpkg.com/three@0.160.0/build/three.module.js",
            "three/addons/":"https://unpkg.com/three@0.160.0/examples/jsm/"}}
</script>

<script type="module">
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js';

const canvas = document.getElementById('c');
const renderer = new THREE.WebGLRenderer({canvas, antialias: true, alpha: true});
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(canvas.clientWidth, canvas.clientHeight, false);
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;

const scene = new THREE.Scene();
// 不要 scene.background — 透明 canvas
scene.environment = new THREE.PMREMGenerator(renderer)
  .fromScene(new RoomEnvironment(), 0.04).texture;

const camera = new THREE.PerspectiveCamera(35, canvas.clientWidth/canvas.clientHeight, 0.05, 50);
camera.position.set(1, 0.6, 1.2);

new GLTFLoader().load('./models/foo.glb', (gltf) => {
  const obj = gltf.scene;
  const bb = new THREE.Box3().setFromObject(obj);
  const sz = new THREE.Vector3(); bb.getSize(sz);
  obj.scale.setScalar(1.0 / Math.max(sz.x, sz.y, sz.z));
  bb.setFromObject(obj); obj.position.y -= bb.min.y;  // 落底
  scene.add(obj);
  (function tick(){ renderer.render(scene, camera); requestAnimationFrame(tick); })();
});
</script>
</body></html>
```

---

## 当前 server 状态

```
PID 3225 — python3 -m http.server 8765 --bind 127.0.0.1
URL    http://127.0.0.1:8765/index.html
Log    /tmp/mac-portfolio-8765.log
```

端到端验证（2026-07-12 12:17）：

```
index.html             200  22369B
macintosh_classic.glb  200   49952B
floppy_red.glb         200  3.5MB
floppy_orange.glb      200  3.5MB
floppy_yellow.glb      200  3.5MB
floppy_green.glb       200  3.5MB
floppy_blue.glb        200  3.5MB
```

---

## 下一步（建议优先级）

1. **性能优化 — 方案 1+3**：重新 bake floppy 贴图（1024²）+ IntersectionObserver lazy-load。预期总传输降到 2MB + 体感即时。
2. **真机 mobile 测**：iPhone Safari + Android Chrome。`.floppy-row` 在 <600px 是 5 列 6px gap，可能挤。
3. **Hero 完整 Mac 拼装**：把 `Computer_Retro.glb` + `Keyboard_Retro.glb` + `ComputerScreen_Retro.glb` 都 load 进来，加个 "assemble" 动画（屏幕亮、键盘推入）。
4. **init git repo**（`.gitignore` 把 `__pycache__` / `.DS_Store` 排除）—— 没历史就别从零重建了。
5. 删 `img/` 里的 PNG（已无用，被 canvas 替代）。

---

_Last updated: 2026-07-12 12:20 GMT+8 · session 9a91046c-195f-4f3e-ba15-12f7f94d6fa9_
