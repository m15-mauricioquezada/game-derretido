# Plan de Refactor v2 — Face Tracking Dinámico

**Archivo objetivo:** `face-puzzle.html` (bloque `<script>` del juego, no la librería face-api)  
**Estado:** en desarrollo  
**Referencia técnica:** SPEC.md sección 11

---

## Resumen

Tres cambios secuenciales que convierten el juego en una experiencia equivalente a los filtros de Instagram/Snapchat:

| ID | Cambio | Por qué | Esfuerzo |
|----|--------|---------|----------|
| C | Blink calibrado | EAR fijo no funciona para todos los usuarios/distancias | 2h |
| B | Borrado dinámico | La cara borrada sigue el movimiento de la cabeza | 1h |
| A | Piezas siguen la cara | Las piezas caen ancladas a la cara, no en coords fijas | 3-4h |

---

## Cambio C — Blink Calibrado

> El problema actual: `BLINK_THRESHOLD = 0.25` es fijo. Cada persona tiene diferente apertura ocular y la distancia a la cámara también afecta el EAR. La solución es medir el EAR base del usuario al inicio y calcular el umbral dinámicamente.

### Checklist

- [ ] **C1** — Reducir `inputSize` de TinyFaceDetector de `320` → `160`
  - En `loadModels()` o en `detectOnce()`: `new faceapi.TinyFaceDetectorOptions({ inputSize: 160, scoreThreshold: 0.5 })`
  - Efecto: detección ~2× más rápida (~15-20fps vs ~8-10fps), acepta menor precisión de detección a cambio de mayor frecuencia de muestras EAR

- [ ] **C2** — Agregar variables de calibración (scope global del juego)
  ```js
  let earBaseline = 0.32;        // EAR promedio con ojos abiertos (se calibra)
  let earCalibrated = false;
  const EAR_BLINK_RATIO = 0.65;  // el blink se dispara cuando EAR cae al 65% del baseline
  const EAR_REOPEN_RATIO = 0.80; // histeresis: reapertura al 80% del baseline
  ```

- [ ] **C3** — Agregar función `calibrateEAR()`
  ```js
  async function calibrateEAR() {
    // Muestrea EAR durante 2 segundos con ojos abiertos
    // Retorna el promedio como baseline
    setStatus('Mantén los ojos abiertos… calibrando', '');
    const samples = [];
    const start = performance.now();
    while (performance.now() - start < 2000) {
      const d = await detectOnce();
      if (d) {
        const pts = d.landmarks.positions;
        const ear = (eyeEAR(pts.slice(36, 42)) + eyeEAR(pts.slice(42, 48))) / 2;
        if (ear > 0.1) samples.push(ear);  // descartar outliers bajos
      }
      await new Promise(r => setTimeout(r, 50));
    }
    if (samples.length < 5) return 0.32;  // fallback si hay pocos datos
    // Usar percentil 75 (no el promedio) para ser robusto ante parpadeos durante calibración
    samples.sort((a, b) => a - b);
    return samples[Math.floor(samples.length * 0.75)];
  }
  ```

- [ ] **C4** — Agregar rolling average de EAR sobre los últimos 3 frames
  ```js
  const earHistory = [];
  const EAR_HISTORY_SIZE = 3;

  function smoothedEAR(rawEar) {
    earHistory.push(rawEar);
    if (earHistory.length > EAR_HISTORY_SIZE) earHistory.shift();
    return earHistory.reduce((s, v) => s + v, 0) / earHistory.length;
  }
  ```

- [ ] **C5** — Modificar `updateEar()` para usar threshold dinámico y EAR suavizado
  ```js
  function updateEar(lm) {
    const pts = lm.positions;
    const left = pts.slice(36, 42);
    const right = pts.slice(42, 48);
    const rawEar = (eyeEAR(left) + eyeEAR(right)) / 2;
    const ear = smoothedEAR(rawEar);

    // Barra visual: mapear usando baseline real
    const norm = Math.max(0, Math.min(1, (ear - 0.1) / (earBaseline - 0.1)));
    dom.earFill.style.width = (norm * 100) + '%';

    // Threshold dinámico basado en baseline calibrado
    const blinkThreshold = earBaseline * EAR_BLINK_RATIO;
    const reopenThreshold = earBaseline * EAR_REOPEN_RATIO;

    const now = performance.now();
    if (earState.open && ear < blinkThreshold) {
      earState.open = false;
      if (now - earState.lastBlinkAt > BLINK_COOLDOWN_MS) {
        earState.lastBlinkAt = now;
        dom.blinkInd.classList.add('blinking');
        setTimeout(() => dom.blinkInd.classList.remove('blinking'), 200);
        onBlink();
      }
    } else if (!earState.open && ear > reopenThreshold) {
      earState.open = true;
    }
  }
  ```

- [ ] **C6** — Integrar calibración en el flujo de `btnStart` (después de `startCamera`)
  ```js
  dom.btnStart.addEventListener('click', async () => {
    dom.btnStart.disabled = true;
    try {
      await loadModels();
      await startCamera();
      // Calibrar antes de iniciar el loop
      earBaseline = await calibrateEAR();
      earCalibrated = true;
      setStatus('Calibración lista. Mira al frente y pulsa "Capturar partes"');
      startLiveLoop();
      dom.btnStart.disabled = true;
    } catch (e) {
      setStatus('Error: ' + e.message, 'error');
      dom.btnStart.disabled = false;
    }
  });
  ```

- [ ] **C7** — Eliminar constantes estáticas que ya no aplican
  - Eliminar: `const BLINK_THRESHOLD = 0.25;`
  - Eliminar: `const BLINK_REOPEN = 0.30;`
  - Mantener: `const BLINK_COOLDOWN_MS = 400;`

- [ ] **C8** — Limpiar `earHistory` en reset
  ```js
  // En dom.btnReset handler:
  earHistory.length = 0;
  earCalibrated = false;
  ```

---

## Cambio B — Borrado de Cara Dinámico

> Actualmente `drawErasedFace()` usa `capturedLandmarks` (estático, guardado al momento de capturar las partes). Si la persona mueve la cabeza, el borrado queda en la posición original. El fix es usar `lastDetection.landmarks` (actual) en cada frame del render loop.

### Checklist

- [ ] **B1** — Modificar firma de `drawErasedFace()` para recibir `lm` como parámetro
  ```js
  // ANTES:
  function drawErasedFace() {
    if (!capturedLandmarks) return;
    const poly = buildFacePolygon(capturedLandmarks);

  // DESPUÉS:
  function drawErasedFace(lm) {
    if (!lm) return;
    const poly = buildFacePolygon(lm);
  ```

- [ ] **B2** — Actualizar `renderCaptured()` para pasar `capturedLandmarks`
  ```js
  function renderCaptured() {
    ctx.clearRect(0, 0, videoW, videoH);
    ctx.drawImage(dom.video, 0, 0, videoW, videoH);
    drawErasedFace(capturedLandmarks);  // preview usa captura estática
  }
  ```

- [ ] **B3** — Actualizar `renderGameFrame()` para pasar landmarks actuales
  ```js
  function renderGameFrame() {
    ctx.clearRect(0, 0, videoW, videoH);
    ctx.drawImage(dom.video, 0, 0, videoW, videoH);
    // Usar detección actual si existe, fallback a captura
    const lm = (lastDetection ? lastDetection.landmarks : null) ?? capturedLandmarks;
    drawErasedFace(lm);
    // ... resto sin cambios ...
  }
  ```

- [ ] **B4** — Agregar `ctx.filter` para bordes suaves en `drawErasedFace()`
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
  > Safari < 18 fallback: usar `ctx.shadowBlur = 10; ctx.shadowColor = sk;` antes del fill.

---

## Cambio A — Coordenadas Relativas (Piezas Siguen la Cara)

> Actualmente cada pieza guarda su posición en píxeles absolutos (`bbox.x`, `bbox.y`). Si la persona se mueve, las piezas quedan en la posición original. La solución: guardar la posición de cada pieza como offset relativo al pivot de la cara (nariz tip, landmark 30), normalizado por la escala interpupilar. En cada frame se recalcula la posición en pantalla desde la cara actual.

### Checklist

- [ ] **A1** — Agregar función `getFaceRef(lm)`
  ```js
  function getFaceRef(lm) {
    const pts = lm.positions;
    const leftEyeCenter  = { x: (pts[36].x + pts[39].x) / 2, y: (pts[36].y + pts[39].y) / 2 };
    const rightEyeCenter = { x: (pts[42].x + pts[45].x) / 2, y: (pts[42].y + pts[45].y) / 2 };
    const scale = Math.hypot(
      rightEyeCenter.x - leftEyeCenter.x,
      rightEyeCenter.y - leftEyeCenter.y
    );
    return { cx: pts[30].x, cy: pts[30].y, scale: scale || 1 };
  }
  ```

- [ ] **A2** — Agregar función `getPartScreenPos(part, currentRef)`
  ```js
  function getPartScreenPos(part, currentRef) {
    const s = currentRef.scale / part.captureRef.scale;
    return {
      x: currentRef.cx + part.relativeCenter.dx * s,
      y: currentRef.cy + part.relativeCenter.dy * s,
      w: part.relativeSize.w * currentRef.scale,
      h: part.relativeSize.h * currentRef.scale,
    };
  }
  ```

- [ ] **A3** — Modificar `captureParts()` para calcular `captureRef`
  ```js
  // Después de: const lm = stable.landmarks;
  const captureRef = getFaceRef(lm);
  // Pasar captureRef a cada capturePart():
  parts.push(capturePart('Ceja izquierda', pts.slice(17, 22), 1.5, 4.0, 14, captureRef));
  // ... mismo para todas las partes ...
  ```

- [ ] **A4** — Modificar `capturePart()` para recibir `captureRef` y guardar offset relativo
  ```js
  function capturePart(name, basePoints, expandX, expandY, pad, captureRef) {
    // ...código actual de recorte...
    return {
      // ...campos actuales...
      relativeCenter: {
        dx: (minX + w/2) - captureRef.cx,
        dy: (minY + h/2) - captureRef.cy,
      },
      relativeSize: { w: w / captureRef.scale, h: h / captureRef.scale },
      captureRef,
    };
  }
  ```

- [ ] **A5** — Modificar `drawPartImage()` para aceptar posición dinámica
  ```js
  // ANTES:
  function drawPartImage(part, y) {
    ctx.save();
    ctx.drawImage(part.image, part.bbox.x, y);
    ctx.restore();
  }
  // DESPUÉS:
  function drawPartImage(part, x, y, w, h) {
    ctx.save();
    ctx.drawImage(part.image, x, y, w ?? part.w, h ?? part.h);
    ctx.restore();
  }
  ```

- [ ] **A6** — Actualizar `renderGameFrame()` para usar `getPartScreenPos`
  ```js
  function renderGameFrame() {
    ctx.clearRect(0, 0, videoW, videoH);
    ctx.drawImage(dom.video, 0, 0, videoW, videoH);
    const lm = (lastDetection?.landmarks) ?? capturedLandmarks;
    drawErasedFace(lm);

    const currentRef = lastDetection ? getFaceRef(lastDetection.landmarks) : null;

    // Piezas fijadas
    for (let i = 0; i < currentPartIdx; i++) {
      const p = parts[i];
      if (currentRef) {
        const pos = getPartScreenPos(p, currentRef);
        const fixedScreenY = pos.y - pos.h / 2 + (p.fixedY - p.originalY);
        drawPartImage(p, pos.x - pos.w / 2, fixedScreenY, pos.w, pos.h);
      } else {
        drawPartImage(p, p.bbox.x, p.fixedY - p.h / 2, p.w, p.h);
      }
    }

    // Pieza cayendo
    if (currentPartIdx < parts.length) {
      const p = parts[currentPartIdx];
      if (currentRef) {
        const pos = getPartScreenPos(p, currentRef);
        drawPartImage(p, pos.x - pos.w / 2, p.currentY, pos.w, pos.h);
      } else {
        drawPartImage(p, p.bbox.x, p.currentY, p.w, p.h);
      }
    }

    advancePart();  // nota: mover fuera de renderGameFrame si ya está separado
  }
  ```

- [ ] **A7** — Actualizar `showResult()` para normalizar score por escala
  ```js
  const avgDiffPx = parts.reduce((sum, p) => {
    const diff = Math.abs(p.fixedY - p.originalY);
    // Normalizar: 1 unidad de distancia interpupilar = 100px base
    return sum + (diff / p.captureRef.scale) * 100;
  }, 0) / parts.length;
  ```

- [ ] **A8** — Limpiar `captureRef` en `advancePart()` no aplica, pero verificar que `fixedY` y `originalY` siguen siendo coordenadas absolutas de canvas (no relativas) — el score se normaliza en `showResult()` con el factor de escala, los demás usos internos quedan igual

---

## Notas de implementación

### Orden estricto de commits

```
C → B → A
```
Cada cambio debe funcionar de forma aislada antes de pasar al siguiente.

### Testing manual por cambio

**Tras C:** pestañear debe disparar el trigger de forma consistente para distintas personas y distancias. La barra EAR debe moverse claramente al pestañear.

**Tras B:** mover la cabeza de lado a lado mientras la cara está "borrada" — el parche de piel debe seguir la cara perfectamente.

**Tras A:** mover la cabeza mientras las piezas caen — las piezas deben moverse con la cara, manteniendo su posición relativa.

### Fallbacks importantes

- `getPartScreenPos`: si `currentRef` es null (no hay detección en ese frame), usar bbox original como fallback
- `drawErasedFace`: si `lm` es null, no dibujar nada (no crash)
- `calibrateEAR`: si hay menos de 5 muestras, usar baseline de 0.32

---

## Checklist de deploy

- [ ] Commit en `game-derretido` con mensaje descriptivo por cada cambio (C, B, A)
- [ ] Verificar que el Action `deploy-to-hub.yml` corre sin errores
- [ ] Verificar que el Action `firebase-deploy.yml` en `kitchhero-games` corre sin errores
- [ ] Hard refresh en `https://games-kitchhero-prod.web.app/derretido/` y probar los tres cambios
