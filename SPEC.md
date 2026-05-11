# Face Puzzle (Derretido) — Especificación

Documento de referencia técnica del juego. Estado al 2026-05-08.

---

## 1. Concepto

Juego web que usa la cámara frontal del dispositivo para detectar el rostro del jugador, "borrar" sus rasgos faciales (cejas, ojos, nariz, boca) y hacer que cada parte caiga desde arriba. El jugador debe **pestañear en el momento exacto** para fijar cada pieza lo más cerca posible de su posición original.

Score basado en la distancia entre la posición donde se fijó cada parte y su ubicación original detectada.

---

## 2. Mecánica de juego

### Flujo

1. **Pantalla intro** → instrucciones + botón "Entendido".
2. **Iniciar cámara** → pide permisos, carga modelos de IA, abre stream.
3. **Modo live** → detecta cara en tiempo real, dibuja cajitas guía sobre cada parte, muestra barra EAR (Eye Aspect Ratio).
4. **Capturar partes** → al pulsar el botón, recorta las 6 piezas faciales y guarda sus posiciones objetivo.
5. **Comenzar** → la cara queda con las zonas de cejas/ojos/nariz/boca rellenadas con color de piel (efecto "sin rasgos"). Cada pieza aparece arriba del canvas y baja a velocidad constante.
6. **Pestañeo = stop** → al detectar pestañeo (EAR < 0.21 con histéresis y cooldown), la pieza actual se fija en la Y donde iba. Pasa a la siguiente.
7. **Resultado** → cuando todas las piezas están fijadas, calcula score (100 = perfecto, penaliza por píxeles de diferencia promedio) y muestra ranking ("Ojo de cirujano", "Frankenstein moderno", etc.).

### Piezas detectadas (en orden de caída)

| # | Parte | Landmarks face-api (índices) |
|---|---|---|
| 1 | Ceja izquierda | 17–21 |
| 2 | Ceja derecha | 22–26 |
| 3 | Ojo izquierdo | 36–41 |
| 4 | Ojo derecho | 42–47 |
| 5 | Nariz | 27–35 |
| 6 | Boca | 48–59 (outer lips) |

### Detección de pestañeo (EAR)

Eye Aspect Ratio clásico: `EAR = (||p2-p6|| + ||p3-p5||) / (2·||p1-p4||)`. Se promedian ambos ojos.

- **Umbral cierre**: EAR < 0.21 → ojo considerado cerrado.
- **Umbral apertura**: EAR > 0.27 → vuelve a estado abierto (histéresis para evitar dobles disparos).
- **Cooldown**: 600 ms entre pestañeos consecutivos.

---

## 3. Stack técnico

| Capa | Tecnología |
|---|---|
| Lenguaje | HTML5 + CSS3 + JS vanilla (sin frameworks, sin build step) |
| Detección facial | [face-api.js](https://github.com/justadudewhohacks/face-api.js) v0.22.2 |
| Modelos | TinyFaceDetector (~190 KB) + faceLandmark68Net (~350 KB) |
| Render | Canvas 2D + `<video>` con `getUserMedia` |
| Empaquetado | Single-file HTML auto-contenido (~1.37 MB) |

### Por qué face-api y no MediaPipe

Se evaluaron tres opciones:
- **MediaPipe Face Landmarker**: mejor calidad (478 landmarks 3D + blendshapes nativos para blink), pero modelo de ~10 MB → HTML resultante ~14 MB en base64.
- **face-api.js** (elegido): 68 landmarks suficientes para cejas/ojos/nariz/boca + EAR funciona bien. Modelos pequeños → HTML de 1.37 MB.
- **Detección manual con FaceDetector API**: descartado por baja calidad y soporte limitado.

---

## 4. Arquitectura del archivo

`face-puzzle.html` es un **único archivo auto-contenido** que funciona 100% offline. Estructura:

```
<!DOCTYPE html>
<html>
<head>
  <style>… (CSS del juego, ~280 líneas)</style>
</head>
<body>
  <div id="app">
    <header>… score panel …</header>
    <div id="stage">
      <video id="video">
      <canvas id="overlay">
      <div id="status">
      <div id="blink-indicator"> ← barra EAR en vivo
      <div id="controls"> ← Iniciar / Capturar / Comenzar / Reset
      <div id="loading">
      <div id="intro"> ← modal de bienvenida
      <div id="result"> ← modal de score final
    </div>
  </div>
  <script>face-api.min.js inline (~648 KB)</script>
  <script>
    const TINY_MANIFEST = {…};                 // JSON manifest del detector
    const TINY_B64 = "AAAA…";                  // shard binario en base64 (~258 KB)
    const LM_MANIFEST = {…};                   // JSON manifest del landmarker
    const LM_B64 = "AAAA…";                    // shard binario en base64 (~476 KB)
    // ... lógica del juego (~600 líneas)
  </script>
</body>
</html>
```

### Cómo se cargan los modelos sin red

face-api.js fue diseñado para cargar modelos por HTTP desde un directorio. Para servirlos desde el propio HTML:

1. Los binarios `.shard1` se convirtieron a base64 y se embebieron en variables.
2. Los `.json` manifests se embebieron como objetos JS.
3. Se hace **monkey-patch del fetch interno de face-api**:

```js
faceapi.env.monkeyPatch({ fetch: customFetch });
```

donde `customFetch` intercepta URLs que terminan en los nombres de manifest/shard y devuelve un `Response` construido desde el base64 decodificado.

> **Nota crítica**: face-api captura su referencia a `fetch` al cargar el módulo. Sobreescribir solo `window.fetch` **no funciona** — el patch debe hacerse vía `faceapi.env.monkeyPatch` después de que face-api esté cargado.

### Build

`build.py` ensambla el HTML final desde:
- `game.html` (template con placeholders)
- `face-api.min.js` (descargado de jsdelivr)
- 4 archivos del modelo (descargados de GitHub `justadudewhohacks/face-api.js/weights`)

```bash
cd /tmp/face-puzzle-build
python3 build.py
# → escribe /Users/mauricioquezada/M15_Dev/game-derretido/face-puzzle.html
```

---

## 5. Detalles de implementación clave

### 5.1 Recorte de partes con polígono expandido

Cada parte se recorta como un polígono cerrado (no un rectángulo) usando los landmarks. El polígono se **expande respecto a su centroide** con factores diferentes en X/Y para incluir un margen visual:

```js
function expandPolygonAroundCentroid(points, factorX, factorY, padPx) {
  // calcula centroide, mueve cada punto hacia afuera por factor + offset normal
}
```

Factores usados:
- Cejas: `(1.0, 2.4)` — expande mucho verticalmente para tomar la zona arriba/abajo de la ceja.
- Ojos: `(1.15, 1.8)` — moderado.
- Nariz: `(1.05, 1.05)` — casi sin expansión.
- Boca: `(1.1, 1.5)` — expansión vertical para cubrir labios completos.

### 5.2 "Borrado" de la cara

Sobre el canvas se pinta cada polígono expandido con un **radial gradient** del color promedio de las mejillas (samples de landmarks 1, 2, 14, 15) hacia un tono ligeramente más oscuro en los bordes. Esto da un efecto suave que no se nota como un "parche".

### 5.3 Espejado del video

El video y canvas tienen `transform: scaleX(-1)` en CSS para que el jugador se vea como en un espejo. **Las coordenadas internas usan los píxeles reales** del video (no espejadas) — el espejo es solo visual.

Cuando se dibuja texto sobre el canvas, se aplica un `ctx.scale(-1, 1)` temporal para contrarrestar el flip y que el texto no salga al revés.

### 5.4 Score

```
score = max(0, round(100 - avgDiffPx · 0.6))
```

Donde `avgDiffPx` es el promedio de `|fixedY - originalY|` para todas las piezas fijadas.

Ranking por score:
- ≥ 95 → "¡Perfecto! 🏆"
- ≥ 80 → "¡Bien hecho! 👍"
- ≥ 60 → "Cara digna de un Picasso 🎨"
- ≥ 40 → "Frankenstein moderno 🧟"
- < 40 → "Reconstrucción imposible 😅"

---

## 6. Bugs resueltos durante el desarrollo

### Bug 1: Modelos no cargaban (404)

**Síntoma**: Errores `failed to fetch: (404) File not found, from url: …/models/tiny_face_detector_model-weights_manifest.json` al pulsar "Iniciar cámara".

**Causa**: El interceptor inicial sobreescribía `window.fetch`, pero face-api ya había capturado una referencia al `fetch` original al cargar el módulo. El interceptor nunca se invocaba.

**Fix**: Usar `faceapi.env.monkeyPatch({ fetch: customFetch })` que sí afecta el entorno interno de face-api.

### Bug 2: Botón "Entendido" no respondía

**Síntoma**: Al hacer click en el botón del modal de intro, no pasaba nada.

**Causa**: En viewports estrechos (móvil o paneles de IDE), el contenido del `intro-card` excedía la altura del contenedor `#intro` (que era `position: absolute` con `align-items: center`). El botón quedaba **fuera del viewport** y el click no llegaba al elemento.

**Fix**: 
- `#intro` y `#result` ahora son `position: fixed` con `overflow-y: auto` y `align-items: flex-start`.
- El card usa `margin: auto` para centrarse cuando hay espacio, y permite scroll cuando no.

---

## 7. Bugs resueltos (continuación)

### Bug 3: Cámara no muestra nada después de dar permisos ✓ resuelto

**Síntoma**: Al pulsar "Iniciar cámara", el usuario otorga permisos pero la pantalla queda en negro.

**Causa**: Race condition en `startCamera()`. El código original asignaba `onloadedmetadata` *después* de setear `srcObject`. Si el evento ya había disparado (común en Chrome mobile y Safari), la Promise nunca resolvía, `videoWidth` quedaba 0, y el canvas se dimensionaba a 0×0.

**Fix aplicado** (en `startCamera()`):
```js
dom.video.srcObject = stream;

// Robusto contra race condition
if (dom.video.readyState < 1) {
  await new Promise(r => dom.video.addEventListener('loadedmetadata', r, { once: true }));
}
await dom.video.play();

// Algunos browsers reportan videoWidth=0 brevemente tras metadata
if (!dom.video.videoWidth) {
  await new Promise(r => dom.video.addEventListener('canplay', r, { once: true }));
}

videoW = dom.video.videoWidth || 640;
videoH = dom.video.videoHeight || 480;
```

---

## 8. Publicación — Arquitectura multi-repo

El juego se publica mediante un flujo automático de GitHub Actions:

1. Push a `main` en este repo (`m15-mauricioquezada/game-derretido`)
2. El Action `.github/workflows/deploy-to-hub.yml` copia `face-puzzle.html` → `web/public/derretido/index.html` en el hub (`m15-mauricioquezada/kitchhero-games`)
3. El hub detecta el push en `web/public/**` y ejecuta `firebase deploy --only hosting:prod`
4. URL final: `https://games-kitchhero-prod.web.app/derretido/`

**Secrets requeridos en este repo:**
- `HUB_DEPLOY_TOKEN` — GitHub PAT con permisos `repo` + `workflow` sobre `kitchhero-games`

---

## 9. Estructura de archivos del repo

```
game-derretido/
├── face-puzzle.html                        # juego completo, single-file, 1.37 MB
├── SPEC.md                                 # este documento
└── .github/workflows/deploy-to-hub.yml    # CI/CD → kitchhero-games hub
```

Build artifacts (fuera del repo):
```
/tmp/face-puzzle-build/
├── game.html              # template con placeholders
├── build.py               # ensamblador
├── face-api.min.js        # ~648 KB
├── tiny_face_detector_model-weights_manifest.json
├── tiny_face_detector_model-shard1
├── tiny.b64               # shard en base64
├── face_landmark_68_model-weights_manifest.json
├── face_landmark_68_model-shard1
└── landmark.b64           # shard en base64
```

---

## 10. Referencias

- face-api.js: https://github.com/justadudewhohacks/face-api.js
- EAR (Eye Aspect Ratio): Soukupová & Čech, "Real-Time Eye Blink Detection using Facial Landmarks" (2016)
- Hub multi-juego: `m15-mauricioquezada/kitchhero-games`

---

## 11. Actividad de cambio — Refactor v2: Face Tracking Dinámico

**Estado:** pendiente de implementación  
**Motivación:** El juego actual captura landmarks una sola vez y usa coordenadas de píxeles estáticas. Si la persona se mueve, el borrado de la cara y las piezas flotantes quedan fijas en pantalla en lugar de seguir la cabeza. Además, la detección de pestañeo con EAR + face-api.js es poco confiable (demasiado sensible a distancia y ángulo de cabeza).

**Fuente de inspiración:** Investigación de la arquitectura real de Lens Studio (Snapchat). Todo el efecto vive en coordenadas locales relativas al FaceTracker, no en píxeles absolutos. Cuando la cabeza se mueve, el transform padre actualiza y todos los hijos (máscara + piezas) se mueven solos.

---

### 11.1 Diagnóstico del problema actual

#### Sistema de coordenadas actual (incorrecto)

```
captureParts()
  └─ guarda bbox en píxeles absolutos: { x: 240, y: 180, w: 80, h: 40 }

renderGameFrame()
  └─ pinta cara borrada en pts capturados (estáticos)
  └─ pinta piezas en bbox.x, bbox.y (estáticos)
  └─ si persona se mueve → todo queda en posición original
```

#### Sistema correcto (inspirado en Lens Studio)

```
FaceTracker (actualiza cada frame con lastDetection)
  └─ faceCenter: punto de referencia (landmark 30 = nariz tip)
  └─ faceScale: distancia interpupilar normalizada

Cada pieza guarda:
  └─ relativeCenter: { dx, dy } desde faceCenter en momento de captura
  └─ relativeSize: { w, h } normalizado por faceScale

renderGameFrame()
  └─ recalcula faceCenter y faceScale desde lastDetection
  └─ screenPos = faceCenter_actual + relativeCenter * (faceScale_actual / faceScale_captura)
  └─ cara borrada usa lastDetection.landmarks (no capturedLandmarks)
```

---

### 11.2 Cambio A — Sistema de coordenadas relativas para piezas

**Qué resuelve:** las piezas siguen la cara cuando el usuario se mueve.

#### Paso 1: agregar función auxiliar `getFaceRef`

Agregar en el bloque de funciones auxiliares (después de `expandPolygonAroundCentroid`):

```js
function getFaceRef(lm) {
  // Punto de referencia: nariz tip (landmark 30)
  // Escala: distancia interpupilar (centro ojo izq → centro ojo der)
  const pts = lm.positions;
  const leftEyeCenter = {
    x: (pts[36].x + pts[39].x) / 2,
    y: (pts[36].y + pts[39].y) / 2,
  };
  const rightEyeCenter = {
    x: (pts[42].x + pts[45].x) / 2,
    y: (pts[42].y + pts[45].y) / 2,
  };
  const scale = Math.hypot(
    rightEyeCenter.x - leftEyeCenter.x,
    rightEyeCenter.y - leftEyeCenter.y
  );
  return {
    cx: pts[30].x,   // nariz tip como pivot
    cy: pts[30].y,
    scale: scale || 1,
  };
}
```

#### Paso 2: pasar `captureRef` a `capturePart` y guardar offset relativo

En `captureParts()`, calcular la referencia antes de capturar:

```js
const captureRef = getFaceRef(lm);
```

En `capturePart(name, basePoints, expandX, expandY, pad, captureRef)`, agregar al objeto retornado:

```js
return {
  // ...campos actuales (name, image, originalCenter, bbox, w, h, etc.)...
  relativeCenter: {
    dx: (minX + w/2) - captureRef.cx,
    dy: (minY + h/2) - captureRef.cy,
  },
  relativeSize: {
    w: w / captureRef.scale,
    h: h / captureRef.scale,
  },
  captureRef,
};
```

#### Paso 3: agregar función `getPartScreenPos`

```js
function getPartScreenPos(part, currentRef) {
  const scaleFactor = currentRef.scale / part.captureRef.scale;
  return {
    x: currentRef.cx + part.relativeCenter.dx * scaleFactor,
    y: currentRef.cy + part.relativeCenter.dy * scaleFactor,
    w: part.relativeSize.w * currentRef.scale,
    h: part.relativeSize.h * currentRef.scale,
  };
}
```

#### Paso 4: actualizar `renderGameFrame()`

```js
function renderGameFrame() {
  ctx.clearRect(0, 0, videoW, videoH);
  ctx.drawImage(dom.video, 0, 0, videoW, videoH);
  const lm = lastDetection ? lastDetection.landmarks : capturedLandmarks;
  drawErasedFace(lm);  // ver Cambio B

  const currentRef = lastDetection ? getFaceRef(lastDetection.landmarks) : null;

  // Piezas ya fijadas
  for (let i = 0; i < currentPartIdx; i++) {
    const p = parts[i];
    const pos = currentRef ? getPartScreenPos(p, currentRef) : { x: p.bbox.x, y: p.bbox.y, w: p.w, h: p.h };
    const fixedScreenY = pos.y - pos.h / 2 + (p.fixedY - p.originalY);
    drawPartImage(p, pos.x - pos.w / 2, fixedScreenY, pos.w, pos.h);
  }

  // Pieza cayendo actualmente
  if (currentPartIdx < parts.length) {
    const p = parts[currentPartIdx];
    const pos = currentRef ? getPartScreenPos(p, currentRef) : { x: p.bbox.x, y: 0, w: p.w, h: p.h };
    drawPartImage(p, pos.x - pos.w / 2, p.currentY, pos.w, pos.h);
  }
}
```

#### Paso 5: actualizar `drawPartImage()` para dimensiones dinámicas

```js
// ANTES:
function drawPartImage(part, y) {
  ctx.drawImage(part.image, part.bbox.x, y);
}

// DESPUÉS:
function drawPartImage(part, x, y, w, h) {
  ctx.drawImage(part.image, x, y, w ?? part.w, h ?? part.h);
}
```

#### Paso 6: normalizar el score por escala

```js
// En showResult(), para que el score no varíe con la distancia a la cámara:
const avgDiffPx = parts.reduce((sum, p) => {
  const diff = Math.abs(p.fixedY - p.originalY);
  const normalizedDiff = (diff / p.captureRef.scale) * 100;
  return sum + normalizedDiff;
}, 0) / parts.length;
```

---

### 11.3 Cambio B — Borrado de cara dinámico

**Qué resuelve:** el "blank face" sigue a la persona cuando mueve la cabeza.

#### Paso 1: modificar `drawErasedFace()` para recibir `lm` como parámetro

```js
// ANTES:
function drawErasedFace() {
  if (!capturedLandmarks) return;
  const poly = buildFacePolygon(capturedLandmarks);
  // ...

// DESPUÉS:
function drawErasedFace(lm) {
  if (!lm) return;
  const poly = buildFacePolygon(lm);
  // ...mismo código de cálculo de cx, cy, maxR, gradiente y fill...
}
```

#### Paso 2: actualizar llamadores de `drawErasedFace`

```js
// renderCaptured() — preview estática, usa landmarks capturados:
function renderCaptured() {
  ctx.clearRect(0, 0, videoW, videoH);
  ctx.drawImage(dom.video, 0, 0, videoW, videoH);
  drawErasedFace(capturedLandmarks);
}

// renderGameFrame() — dinámico, usa detección actual (ver Cambio A paso 4):
// drawErasedFace(lm) ya está integrado ahí
```

#### Paso 3: aplicar suavizado de bordes con `ctx.filter`

Dentro de `drawErasedFace(lm)`, envolver el fill con blur para bordes suaves:

```js
ctx.save();
ctx.filter = 'blur(4px)';
ctx.beginPath();
ctx.moveTo(poly[0].x, poly[0].y);
for (let i = 1; i < poly.length; i++) ctx.lineTo(poly[i].x, poly[i].y);
ctx.closePath();
ctx.fillStyle = grad;
ctx.fill();
ctx.filter = 'none';
ctx.restore();
```

> `ctx.filter` está soportado en Chrome, Firefox y Safari 18+. En Safari < 18 se puede usar `ctx.shadowBlur = 8` como fallback (mismo efecto de suavizado de borde).

---

### 11.4 Cambio C — Trigger de boca abierta (reemplaza pestañeo)

**Qué resuelve:** el EAR con face-api.js es poco confiable (sensible a distancia, ángulo, iluminación). La apertura de boca tiene 5× más rango de movimiento y el ratio apertura/ancho es robusto a cambios de escala.

**Mecánica nueva:** el jugador **abre la boca** para fijar la pieza en su posición actual.

#### Paso 1: agregar función `getMouthOpenRatio`

```js
function getMouthOpenRatio(lm) {
  const pts = lm.positions;
  // Apertura vertical: labio superior interior (62) → labio inferior interior (66)
  const mouthOpen = Math.hypot(
    pts[62].x - pts[66].x,
    pts[62].y - pts[66].y
  );
  // Ancho de boca para normalizar por escala/distancia
  const mouthWidth = Math.hypot(
    pts[48].x - pts[54].x,
    pts[48].y - pts[54].y
  );
  return mouthWidth > 0 ? mouthOpen / mouthWidth : 0;
}
```

Índices de landmarks relevantes (face-api 68 pts):
- `pts[48]` — comisura izquierda exterior
- `pts[54]` — comisura derecha exterior
- `pts[62]` — centro labio superior interior
- `pts[66]` — centro labio inferior interior

#### Paso 2: reemplazar variables de estado

```js
// ELIMINAR:
let earState = { open: true, lastBlinkAt: 0 };
const BLINK_THRESHOLD = 0.25;
const BLINK_REOPEN = 0.30;
const BLINK_COOLDOWN_MS = 400;

// AGREGAR:
let mouthState = { open: false, lastTriggerAt: 0 };
const MOUTH_OPEN_THRESHOLD = 0.35;   // ratio apertura/ancho para disparar
const MOUTH_CLOSE_THRESHOLD = 0.20;  // histeresis de cierre
const MOUTH_COOLDOWN_MS = 500;
```

#### Paso 3: reemplazar `updateEar` por `updateMouth`

```js
function updateMouth(lm) {
  const ratio = getMouthOpenRatio(lm);
  // Actualizar barra visual (reutiliza el mismo #ear-fill)
  const norm = Math.max(0, Math.min(1, ratio / 0.5));
  dom.earFill.style.width = (norm * 100) + '%';

  const now = performance.now();
  if (!mouthState.open && ratio > MOUTH_OPEN_THRESHOLD) {
    mouthState.open = true;
    if (now - mouthState.lastTriggerAt > MOUTH_COOLDOWN_MS) {
      mouthState.lastTriggerAt = now;
      dom.blinkInd.classList.add('blinking');
      setTimeout(() => dom.blinkInd.classList.remove('blinking'), 300);
      onBlink();  // reutilizar la función de trigger sin cambios
    }
  } else if (mouthState.open && ratio < MOUTH_CLOSE_THRESHOLD) {
    mouthState.open = false;
  }
}
```

#### Paso 4: reemplazar todas las llamadas a `updateEar`

Buscar en el código: `updateEar(lm)` y reemplazar por `updateMouth(lm)`. Aparece en:
- `renderLive()` — dentro del bloque `if (lastDetection)`
- `startGame()` → `loop()` → `.then(d => { ... updateEar(d.landmarks) ... })`

#### Paso 5: actualizar textos de UI

En el HTML del modal intro, cambiar instrucción 4:
```html
<!-- ANTES -->
<li>Pestañea para soltar cada pieza en su sitio</li>
<!-- DESPUÉS -->
<li>Abre la boca para soltar cada pieza en su sitio</li>
```

En `setStatus` dentro del código JS:
```js
// ANTES:
setStatus('¡Pestañea para fijar las piezas!');
// DESPUÉS:
setStatus('¡Abre la boca para fijar las piezas!');
```

En el label del indicador visual (`#ear-label`):
```html
<!-- ANTES: -->
<span id="ear-label">EAR</span>
<!-- DESPUÉS: -->
<span id="ear-label">BOCA</span>
```

---

### 11.5 Cambio D — Mejoras visuales del borrado (nice to have)

#### D.1 Más puntos de muestreo de color de piel

```js
// ANTES: pts[1], pts[15], pts[2], pts[14]
// DESPUÉS: agregar pts[3], pts[13] (más laterales en la mejilla)
const samples = [pts[1], pts[2], pts[3], pts[13], pts[14], pts[15]];
```

#### D.2 Textura de piel real del usuario (complejidad alta)

Capturar un parche de mejilla al momento de `captureParts()` y usarlo como patrón de relleno:

```js
// En captureParts(), después de sampleSkinColor:
const patch = document.createElement('canvas');
patch.width = 40; patch.height = 40;
patch.getContext('2d').drawImage(dom.video,
  Math.floor(pts[3].x) - 20, Math.floor(pts[3].y) - 20, 40, 40,
  0, 0, 40, 40
);
skinPattern = ctx.createPattern(patch, 'repeat');

// En drawErasedFace(), como fillStyle alternativo:
ctx.fillStyle = skinPattern ?? `rgb(${skinColor[0]},${skinColor[1]},${skinColor[2]})`;
```

Requiere declarar `let skinPattern = null;` junto a las variables globales y limpiarlo en el reset.

---

### 11.6 Orden de implementación recomendado

| Prioridad | Cambio | Impacto percibido | Complejidad |
|-----------|--------|-------------------|-------------|
| 1 | **C — Boca abierta** | Alto: el juego se vuelve jugable | Baja (1-2h) |
| 2 | **B — Borrado dinámico** | Alto: cara sigue el movimiento | Baja (1h) |
| 3 | **A — Coords relativas** | Alto: piezas siguen la cara | Media (3-4h) |
| 4 | **D.1 — Skin sampling** | Medio: mejor color de piel | Baja (30min) |
| 5 | **D.2 — Skin texture** | Medio: visual premium | Alta (2-3h) |

> Implementar C + B primero da un 80% del impacto percibido con el 20% del esfuerzo total.

---

### 11.7 Resumen de funciones afectadas

Todo el juego vive en `face-puzzle.html`. Los cambios son exclusivamente en el bloque `<script>` del juego (no en la librería face-api embebida en línea 353).

| Función | Acción | Cambio |
|---------|--------|--------|
| `getFaceRef(lm)` | Nueva | Retorna `{ cx, cy, scale }` de la cara actual |
| `getPartScreenPos(part, ref)` | Nueva | Retorna posición en pantalla de una pieza según cara actual |
| `getMouthOpenRatio(lm)` | Nueva | Retorna ratio de apertura bucal normalizado |
| `updateMouth(lm)` | Nueva | Reemplaza `updateEar` |
| `capturePart()` | Modificar | Agregar `relativeCenter`, `relativeSize`, `captureRef` al objeto retornado |
| `captureParts()` | Modificar | Calcular `captureRef`, pasarlo a cada `capturePart()` |
| `drawErasedFace()` | Modificar | Recibir `lm` como parámetro, agregar `ctx.filter` blur |
| `renderCaptured()` | Modificar | Pasar `capturedLandmarks` a `drawErasedFace` |
| `renderGameFrame()` | Modificar | Usar `lastDetection.landmarks`, `getFaceRef`, `getPartScreenPos` |
| `drawPartImage()` | Modificar | Aceptar `x, y, w, h` dinámicos |
| `updateEar()` | Eliminar | Reemplazada por `updateMouth` |
| `showResult()` | Modificar | Normalizar score por `captureRef.scale` |
| Variables EAR | Eliminar | `earState`, `BLINK_THRESHOLD`, `BLINK_REOPEN`, `BLINK_COOLDOWN_MS` |
| Variables boca | Agregar | `mouthState`, `MOUTH_OPEN_THRESHOLD`, `MOUTH_CLOSE_THRESHOLD`, `MOUTH_COOLDOWN_MS` |

---

### 11.8 Referencias de investigación

- Lens Studio — Face Inset: https://developers.snap.com/lens-studio/features/ar-tracking/face/face-inset
- Lens Studio — Face Mesh: https://developers.snap.com/lens-studio/features/ar-tracking/face/face-mesh
- Lens Studio — Face Expressions API: https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.Expressions
- Lens Studio — ML Eraser: https://developers.snap.com/lens-studio/4.55.1/references/templates/ml/ml_powered/ml-eraser-disappearing-effects
- Lens Studio — Head Attached 3D Objects: https://developers.snap.com/lens-studio/features/ar-tracking/face/head-attached-3d-objects
- MediaPipe Face Landmarker Web: https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker/web_js
- Filtro 1€ para suavizado de landmarks: https://cristal.univ-lille.fr/~casiez/1euro/
