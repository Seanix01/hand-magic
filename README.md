# ✦ Hand Magic

Realtime hand-gesture drawing in the browser. No backend, no build, no npm — open `index.html` over `localhost`, allow camera, and start sketching with your fingertip in mid-air.

> Single HTML file • MediaPipe HandLandmarker • Three.js • Canvas 2D

[English](#english) · [Русский](#русский)

---

## English

### Run

```bash
git clone https://github.com/Seanix01/hand-magic.git
cd hand-magic
python3 -m http.server 8765
open http://localhost:8765/
```

`getUserMedia` requires `localhost` or HTTPS — opening via `file://` won't work.

### Gestures

| Gesture | Action | Landmark color |
|---|---|---|
| 👆 index up | draw with fingertip | green |
| 👌 pinch (thumb+index) | grab the nearest stroke + drag (and zoom by hand-to-camera distance) | yellow |
| ✋ open palm | erase (Paint-style — slices strokes into pieces) | red |
| ✊ fist / no hand | idle | pink |

Bottom palette has 10 pen colors. Top-right buttons: `Clear all` / `Undo`.

### Stack

Pure browser, no bundler, all from CDN via importmap:

- **Hand tracking** — [@mediapipe/tasks-vision](https://www.npmjs.com/package/@mediapipe/tasks-vision) `HandLandmarker`, 21 keypoints/hand, GPU-accelerated WASM
- **3D / GPU** — [Three.js](https://threejs.org) r160 — `BufferGeometry` + `Points` + custom GLSL shader for the spark trail
- **Drawing** — Canvas 2D API
- **Camera** — WebRTC `getUserMedia`

Everything in a single `index.html`.

### Architecture

Layers, bottom to top:

```
UI (palette / toolbar / hud)        DOM
landmarks overlay                   Canvas 2D
draw layer (strokes)                Canvas 2D
spark trail                         WebGL via Three.js
video                               MediaStream + scaleX(-1)
```

All layers `position: fixed; inset: 0` + `transform: scaleX(-1)` — single mirror axis.

#### Stroke

```js
{
  points: [{x, y}, ...],   // normalized [0..1] in raw frame
  offsetX, offsetY,        // pinch-drag delta
  scale, pivotX, pivotY,   // pinch-zoom around pivot
  color, width,
}
```

Render: `final = (p − pivot) × scale + pivot + offset`, then mapped to screen px via `videoLayout` (compensates `object-fit: cover` cropping).

`bakeStroke()` collapses the current transform back into raw `points` on pinch release — every new grab starts fresh at `scale=1, offset=0`.

#### Eraser

Doesn't drop the whole stroke — walks point-by-point, removes ones inside the palm-radius circle, splits surviving runs (≥ 2 points) into independent strokes that each inherit a transform copy. Drag your hand through the middle of a line and it snaps into two pieces, each individually grabbable.

#### Spark trail

Pool of 800 GPU particles (Three.js `Points`). When drawing, ~35% of frames spawn one spark at the fingertip with low outward velocity, fading in ~0.7s. Color = current `PEN_COLOR` with slight lightness variance. **Critical: `NormalBlending`, not `Additive`** — additive accumulates to white where sparks overlap; normal blending keeps them as discrete colored dots.

### Tuning

Knobs near the top of `<script>`:

- `MIN_STEP = 0.004` — minimum normalized distance between consecutive stroke points
- `pinch < 0.05` — pinch-detection threshold
- `0.025` in `detectGesture` — finger-extended threshold
- `palmWidth(lm) * 1.1` — eraser radius
- `width: 8` in `startStroke` — pen thickness
- `uSize: 2.8` — spark size in shader
- `Math.random() < 0.35` — spark spawn rate

### Roadmap

- [ ] Pen thickness reacting to fingertip speed
- [ ] PNG snapshot (composite of the video frame + draw canvas)
- [ ] Two hands at once
- [ ] 3D drawing using landmark Z-coordinate
- [ ] Bloom postprocess on strokes
- [ ] Catmull-Rom smoothing between stroke points

---

## Русский

Realtime-рисование жестами в браузере. Никакого бэкенда, билдов и npm — открываешь `index.html` через `localhost`, разрешаешь камеру и рисуешь пальцем поверх своего изображения с вебки.

> Один HTML-файл • MediaPipe HandLandmarker • Three.js • Canvas 2D

### Запуск

```bash
git clone https://github.com/Seanix01/hand-magic.git
cd hand-magic
python3 -m http.server 8765
open http://localhost:8765/
```

`getUserMedia` требует `localhost` или HTTPS — через `file://` не сработает.

### Жесты

| Жест | Действие | Цвет landmarks |
|---|---|---|
| 👆 указательный | рисует штрих кончиком пальца | зелёный |
| 👌 пинч (большой+указательный) | хватает ближайший штрих + таскает + зум при движении руки к/от камеры | жёлтый |
| ✋ открытая ладонь | стирает (как ластик в Paint — режет линии на куски) | красный |
| ✊ кулак / нет руки | пасс | розовый |

Палитра внизу — 10 цветов. Кнопки сверху-справа: `Clear all` / `Undo`.

### Стек

Чистый браузер. Ни бандлера, ни npm, всё с CDN через importmap.

- **Трекинг руки** — MediaPipe Tasks Vision (`HandLandmarker`), 21 точка/руку, GPU-ускоренный WASM
- **3D / GPU** — Three.js r160, `BufferGeometry` + `Points` + кастомный GLSL для шлейфа искр
- **Рисование** — Canvas 2D API
- **Камера** — WebRTC `getUserMedia`

Всё в одном `index.html`.

### Архитектура

Слои в браузере (снизу вверх):

```
UI (палитра / toolbar / hud)        DOM
landmarks overlay                   Canvas 2D
draw layer (штрихи)                 Canvas 2D
spark trail                         WebGL через Three.js
video                               MediaStream + scaleX(-1)
```

Все слои `position: fixed; inset: 0` + `transform: scaleX(-1)` — единое зеркало для всех.

#### Штрих

```js
{
  points: [{x, y}, ...],   // нормализованные [0..1] в raw frame
  offsetX, offsetY,        // дельта при перетаскивании
  scale, pivotX, pivotY,   // зум при пинче, вокруг pivot
  color, width,
}
```

Render: `final = (p − pivot) × scale + pivot + offset`, потом маппинг на screen px через `videoLayout` (компенсация `object-fit: cover`).

`bakeStroke()` коллапсит трансформацию обратно в `points` при отпускании пинча — каждый следующий захват начинается с чистого `scale=1, offset=0`.

#### Ластик

Не удаляет штрих целиком — идёт по точкам, выбрасывает попавшие в радиус ладони, оставшиеся run'ы (≥ 2 точки) становятся новыми штрихами со своими копиями transform. Проводишь рукой через середину линии — получаешь два независимых куска, каждый можно отдельно зумить пинчем.

#### Spark trail

Пул из 800 GPU-частиц (Three.js `Points`). При рисовании ~35% кадров спавнят одну искру у кончика пальца, разлетаются с малой скоростью, гаснут за ~0.7 сек. Цвет = текущий `PEN_COLOR` с лёгкой вариацией яркости. **Важно: `NormalBlending`, не Additive** — Additive накапливается до белого где искры перекрываются, normal blending оставляет дискретные цветные точки.

### Настройка

Knob'ы в верху `<script>`:

- `MIN_STEP = 0.004` — минимальная дистанция между точками штриха
- `pinch < 0.05` — порог распознавания пинча
- `0.025` в `detectGesture` — порог "палец поднят"
- `palmWidth(lm) * 1.1` — радиус ластика
- `width: 8` в `startStroke` — толщина пера
- `uSize: 2.8` — размер искры в шейдере
- `Math.random() < 0.35` — частота спавна искр

### Roadmap

- [ ] Толщина пера от скорости пальца
- [ ] Snapshot в PNG (composite кадра видео + draw canvas)
- [ ] Поддержка двух рук одновременно
- [ ] 3D-рисование через Z-координату landmarks
- [ ] Bloom-постпроцесс на штрихах
- [ ] Catmull-Rom сглаживание между точками штриха

---

Made with [Claude Code](https://claude.ai/code) — single-session vibe-coding experiment.
