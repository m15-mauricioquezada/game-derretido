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

## 7. Bug pendiente

### Bug 3: Cámara no muestra nada después de dar permisos

**Síntoma reportado**: Al pulsar "Iniciar cámara", el usuario otorga permisos pero "no pasa nada" — pantalla queda en negro.

**Hipótesis principal** (sin verificar aún):
- El `<video>` tiene `visibility: hidden` en CSS. Solo se ve el `<canvas>` overlay dibujado en el render loop.
- El render loop usa `dom.video.videoWidth/videoHeight`, que pueden ser `0` si `loadedmetadata` no disparó (race condition: el handler se asigna *después* de que el evento ya se disparó).
- Resultado: canvas dimensionado a 0×0 → pantalla negra silenciosa, sin error.

**Fix propuesto** (no aplicado todavía):
1. Cambiar el patrón de espera a algo robusto:
   ```js
   if (video.readyState >= 1) { /* ya tiene metadata */ }
   else { await new Promise(r => video.addEventListener('loadedmetadata', r, {once:true})); }
   ```
2. Quitar `visibility: hidden` del video y manejar la oclusión con `z-index` + `opacity:0` o dejar el video debajo del canvas.
3. Agregar logs de status visibles en cada paso (`Solicitando cámara` → `Stream OK` → `Metadata OK` → `Detectando…`) para debug.

---

## 8. Roadmap de publicación

Pendiente de aprobación. Dos opciones evaluadas:

### Opción B1 — Agregar al hub `games-kitchhero` (recomendada)

Replicar el patrón de `pingpongale`:
- Copiar `face-puzzle.html` → `game-pingpongale/web/public/derretido/index.html`.
- Agregar tarjeta `🫠 Derretido` al hub `web/public/index.html`.
- `firebase deploy --only hosting:prod` desde `game-pingpongale`.
- URL final: `https://games-kitchhero-prod.web.app/derretido/`.

**Ventaja**: hub ya configurado, deploy en 1 comando, consistente con pingpongale.

### Opción B2 — Proyecto Firebase independiente

- Crear `firebase.json` + `.firebaserc` en `game-derretido`.
- Crear nuevo Firebase Hosting site desde Firebase Console.
- Deploy.
- URL final: `https://game-derretido.web.app` (o similar).

**Ventaja**: repo y dominio independientes.
**Desventaja**: más pasos manuales (crear site).

---

## 9. Estructura de archivos del repo

```
game-derretido/
├── face-puzzle.html       # juego completo, single-file, 1.37 MB
└── SPEC.md                # este documento
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
- Patrón hub multi-juego: ver `/Users/mauricioquezada/M15_Dev/game-pingpongale/`
