# Guía de construcción desde cero — Escáner de documentos web

> **Para quién es este documento:** una IA (o desarrollador) que va a construir el proyecto desde un directorio vacío, paso a paso, usando el repositorio de referencia solo cuando tenga dudas concretas.
>
> **Repositorio de referencia:** https://github.com/Jg-Stevan/web-scanner
> **Rama de referencia:** `main` (v6.2, commit `08a8d97`)
>
> **Filosofía de trabajo:** este documento te dice **qué hacer en cada paso**. El repo te dice **cómo lo hizo el original** cuando tengas una duda puntual. No copies a ciegas; entiende y luego escribe.

---

## 0. Cómo usar esta guía

1. Sigue las fases **en orden**. Cada fase termina con un **checkpoint** verificable.
2. Antes de empezar cada fase, lee la sección "Consultar en el repo" — te dice exactamente qué archivos mirar si te atascas.
3. Si una decisión no está en esta guía ni en el repo, **pregunta antes de inventar**. No improvises constantes, colores ni algoritmos.
4. La **regla de oro** manda sobre todo: *"podemos modificar otras cosas pero NUNCA la capacidad de poder ver bien la imagen y reconocer el texto"*.
5. Al final de cada fase: `lint` limpio + `tsc --noEmit` limpio + commit.

---

## 1. Preparación del entorno

### Qué hacer

```bash
mkdir web-scanner && cd web-scanner
git init
bun init -y          # o: npm init -y (pero el original usa bun)
```

Instala el stack base (aún sin nada específico del escáner):

```bash
bun add next@16 react react-dom zustand framer-motion sonner vaul lucide-react
bun add -d typescript @types/react @types/node tailwindcss postcss autoprefixer eslint
```

Configura:
- `next.config.ts` con `output: "export"`, `basePath: process.env.NEXT_PUBLIC_BASE_PATH ?? ""`, `images.unoptimized: true`.
- `tsconfig.json` en modo estricto.
- Tailwind CSS 4.
- `app/layout.tsx` con `viewport-fit=cover`, `100dvh`, fuente sistema iOS.

### Consultar en el repo si dudas

- `next.config.*` — cómo está configurado el export estático y el basePath.
- `tailwind.config.*` y `app/globals.css` — la paleta iOS y las variables.
- `app/layout.tsx` — metadata, viewport, registro del service worker.
- `package.json` — versiones exactas de cada dependencia.

### Checkpoint

- [ ] `bun run dev` levanta un "Hola mundo".
- [ ] `bunx tsc --noEmit` sin errores.
- [ ] `bun run lint` sin errores.

---

## 2. Fase 1 — Esqueleto y modelo de datos

### Qué construir

1. **`src/lib/scanner/types.ts`** con el modelo exacto:
   - `ScannerView`, `PageFilter` (SOLO 3: `original|text|bw`), `Point`, `Quad`, `PageQuality`, `PagePrecision`, `ScanPage`, `ScanDocument`, `CapturePage`, `ScannerSettings`.
   - `defaultQuad()`, `capturePageKey()`, `normalizePageFilter()`, `nextId()`.
   - `DEFAULT_SETTINGS = { enhance: true, ocrEnabled: false, exportQuality: "máxima" }`.

2. **Store Zustand** (`src/lib/scanner/store.ts`) con vistas `library|camera|editor|settings`, páginas de captura, documentos y ajustes. Persistencia a localStorage de ajustes (con migraciones one-time usando claves separadas).

3. **Phone frame + navegación** de 4 vistas con bottom nav estilo iOS. Todo en español.

4. **Toasts** con sonner y **sheets** con vaul, ya integrados (los usarás mucho).

### Reglas duras que empiezan aquí

- Solo 3 filtros. Añadir presets = bug histórico #13.
- `capturePageKey` es la ÚNICA fuente de verdad del estado procesado. Todo preview/merge/rotación la usa.
- Todo en español. Paleta iOS obligatoria: `#007AFF`, `#34C759`, `#FF3B30`, `#8E8E93`, `#F2F2F7`, `#1C1C1E/#2C2C2E/#3A3A3C`.

### Consultar en el repo si dudas

- `src/lib/scanner/types.ts` — el modelo literal.
- `src/lib/scanner/store.ts` — cómo se estructura la store y las migraciones.
- `src/components/scanner/BottomNav.tsx` — el nav.
- `app/layout.tsx` — el phone frame y viewport.

### Checkpoint

- [ ] Puedo cambiar entre las 4 vistas y ver placeholders.
- [ ] La store persiste ajustes en localStorage y los recarga.
- [ ] `capturePageKey` devuelve el mismo string para el mismo estado y cambia al rotar/filtrar/recortar.
- [ ] Migración de filtros legacy funciona: `auto→text`, `blackwhite→bw`, `natural→original`.

---

## 3. Fase 2 — Cámara básica (sin worker)

### Qué construir

1. **`CameraView.tsx`** con:
   - Apertura con cascada simple: `[{facingMode:{exact:"environment"}, width:{ideal:3840}} → {facingMode:"environment", width:{ideal:3840}} → {facingMode:"environment"} → {video:true}]`.
   - Visor con `<video>` a pantalla completa.
   - Overlay SVG del quad (trazo `#007AFF` 2px, puntos blancos en vértices, exterior oscurecido con `evenodd`).
   - Botón shutter 72×72.
   - Botón de importar galería.
   - Fallback a cámara sintética (escena simulada) si `getUserMedia` no está disponible.

2. **`image-processor.ts`** con lo mínimo:
   - `loadImage(src)`, `downscaleImage(img, max)`, `snapshotVideo(video)`.
   - `evaluateQuality(source)` (canvas 120 px, LapVar, contraste, brillo → nivel).
   - `fileToCaptureDataUrl(file)` con la cascada nativa (sin HEIC aún).

3. **`store.ts`**: `addCapturePage`, `updateCapturePage`, `capturePages`.

### Reglas duras que aplican ya

- **Nunca 2 cámaras abiertas a la vez.** Cierra cada stream antes de abrir el siguiente.
- **Over-constraining prohibido:** SOLO `width: {ideal: 3840}`. Nada de `height`, `frameRate`, `aspectRatio`.
- **`toBlob` primero**, `toDataURL` solo como respaldo.
- Sin permiso → aviso con CTA "Activar cámara" + fallback explícito a simulado. Nunca fallback silencioso.

### Consultar en el repo si dudas

- `CameraView.tsx` — el bloque de apertura (`openWithCascade`, `probeCamera`, `chooseMainProbe`).
- `image-processor.ts` — `loadImage`, `downscaleImage`, `snapshotVideo`, `evaluateQuality`, `fileToCaptureDataUrl`.
- `quality.ts` — la fórmula del score del obturador (aún no la usas, pero léela).
- `frame-loop.ts` — aún no lo usas, pero mira cómo estructura los frames.

### Checkpoint

- [ ] La cámara abre en Chrome Android con la trasera (no la frontal).
- [ ] El shutter captura un frame y lo añade a `capturePages`.
- [ ] El editor se abre (placeholder) con la página capturada.
- [ ] Permiso denegado → toast claro + botón "Activar cámara".
- [ ] Importar un JPG funciona; importar un PNG también.

---

## 4. Fase 3 — Motor de visión (worker OpenCV)

### Qué construir

1. **`public/vendor/opencv-4.5.5.js`** (descarga el build self-hosted, no lo sirvas desde CDN por defecto).

2. **`public/scanner/detection-worker.js`** — worker clásico con:
   - Carga de OpenCV vía `importScripts`.
   - Protocolo: `{detect}`, `{warp}`, `{enhance}`, `{config}` de entrada; `{ready}`, `{result}`, `{warped}`, `{enhanced}`, `{error}` de salida.
   - `processFrame` con **cascada de 6 pasadas** (los 6 pares low/high/dilate/eps exactos).
   - `selectQuad` con scoring (área + aspecto + blancura).
   - `refineQuad` con RANSAC (inlier ≤5 px, máx 24 modelos) y least-squares recortando 12 % por extremo.
   - `warpPerspective` con `INTER_CUBIC`, `SHRINK_QUAD_PX = 3.5` (solo no manual), cap ajustable por config.

3. **`detector-client.ts`** — singleton self-healing, promise-chain de 1 mensaje en vuelo, timeout de arranque 25 s, cache-buster `?v=7`.

4. **`frame-loop.ts`** — `CameraFrameLoop` con `requestVideoFrameCallback`, downscale a 400 px, histograma de exposición, backpressure por descarte, telemetría a 10 Hz, historiales de 2 s con cap 24.

5. **Fallback canvas** (sin worker) con la misma matemática: homografía DLT 8×8 con pivoteo, muestreo bilineal, `cropQuad` con scale uniforme.

### Reglas duras

- El worker **cierra SIEMPRE el bitmap** en `finally` y libera los `Mat` en LIFO (`withMats`).
- Los datos salen **copiados** del buffer WASM.
- 1 mensaje en vuelo: si el worker está ocupado, el frame loop **descarta** (no encola).
- El cap de warp se ajusta en caliente vía `config {maxWarpLongSide}`: high → 4032, medium/low → 3200.
- **Nunca upscalar** en el warp.

### Consultar en el repo si dudas

- `detection-worker.js` — los 6 pases, `selectQuad`, `refineQuad`, `warpPerspective`, el manejo de Mats.
- `detector-client.ts` — promise-chain, self-healing, timeout.
- `frame-loop.ts` — `requestVideoFrameCallback`, backpressure, histograma.
- `image-processor.ts` — la ruta fallback con DLT.

### Checkpoint

- [ ] El worker arranca y responde `{ready}`.
- [ ] Detectar una hoja blanca sobre mesa oscura devuelve un quad convexo.
- [ ] Un documento a distancia normal (no llenando el encuadre) se detecta (área mínima 10 %, no 25 %).
- [ ] El warp produce una imagen sin fondo y sin upscaling.
- [ ] Matar el worker → la app sigue funcionando con la ruta canvas.

---

## 5. Fase 4 — Filtros y pipeline completo

### Qué construir

1. **`image-processor.ts`** con `processImage(source, quad, filter, rotation, opts)` que:
   - Llama al worker para warp (o fallback canvas).
   - Aplica el filtro correspondiente (`original→raw`, `text→text`, `bw→bw`) con la **matemática exacta**:
     - Modelo de sombras común: downscale a 800 px, close morfológico separable (kernel 25), ganancias bilineales.
     - Unsharp antes del filtro (excepto raw): `GaussianBlur(k=7, σ=1.5)` + `src·1.5 − blur·0.5`.
     - `text`: white-point p85, S-curve pivote 0.72 ×1.8, black point 0.2, ganancia `min(4, ink/gray)`, factor total `min(8)` conservando croma.
     - `bw`: Bradley-Roth (ventana w/12 impar, umbral 0.15) + despeckle flood-fill 4-conectividad <3 px.
   - Encode: raw/text/bw → **PNG**; resto JPEG 0.9.
   - Thumbnails: **160 px JPEG 0.8**.

2. **Cache LRU de previews** (12 entradas) en el editor.

3. **Editor en modo revisión**: preview procesado, pills, carrusel de thumbs (usando `p.thumbnail`, no `p.original`), toolbar de 7 botones.

### Reglas duras

- **Procesado siempre PNG** (raw/text/bw).
- **El recorte manual manda al píxel:** `quadManual: true` ⇒ warp sin refine y sin shrink.
- `setPreview(null)` ANTES de procesar (evita que el OCR lea la página anterior).
- Filtros conmutativos con rotaciones de 90° → cache-hit por clave.

### Consultar en el repo si dudas

- `image-processor.ts` — las funciones de filtro y el modelo de sombras.
- `detection-worker.js` — el bloque de `enhanceImage`.
- `EditorView.tsx` — `cachePreviewEntry`, el LRU, `processImage` en el preview.
- `types.ts` — las constantes de filtro (`TEXT_CLARO_*`, `BW_*`).

### Checkpoint

- [ ] Los 3 filtros producen: text = papel blanco + tinta marcada sin "píxeles feos"; bw = B/N inmune a sombras sin motas <3 px; original = foto pura.
- [ ] El thumbnail se usa en el carrusel (no se decodifica la original de 12 MP).
- [ ] El cambio de filtro reprocesa solo si cambia la clave.
- [ ] La rotación de 90° es píxel-idéntica al reproceso completo.

---

## 6. Fase 5 — Editor completo

### Qué construir

1. **Gestos** (F-ZOOM):
   - Pinza 2 dedos 1×–6× anclada al punto medio.
   - Pan con 1 dedo >1×, clampeado a la imagen; a 1× → swipe de páginas (eje decidido una vez con `|dx| > |dy|·1.2`), goma 72 px, spring (500/42).
   - Doble toque (1× ↔ 2.5× centrado, <320 ms, ≤8 px).
   - Rueda con **listener nativo** `{passive:false}` (el `onWheel` de React es pasivo).
   - Compare manteniendo pulsado a 1× (350 ms) → vibrate(18) + overlay original 0.14 s.
   - **Guard correcto:** activar salvo que `e.target.closest("button")` exista.

2. **Crop mode**:
   - 4 handles de esquina con pad táctil 44×44.
   - 4 puntos medios que trasladan aristas.
   - **Lupa 3×** (radio 84 px) en el lado vertical OPUESTO al dedo, crosshair amarillo.
   - Al soltar un handle → persistir `{quad, quadManual: true}` en el store.
   - **Snapshot al entrar** en crop → Cancelar restaura el snapshot (no el estado actual).
   - "Detección automática" → `detectDocumentEdges` + `quadManual: false`.

3. **Rotación instantánea** (~0.2 s):
   - Rotar el data URL procesado en canvas.
   - Re-encode PNG async.
   - Sembrar la cache con la nueva clave.
   - Una sola `updateCapturePage({rotation, processed, processedKey, thumbnail})`.

4. **Multi-página** con "Seguir escaneando" / "Añadir página" (merge en modo documento con snapshot de ids ANTES de los awaits).

5. **Overlay de éxito** al guardar (auto-cierra 3400 ms con acciones, 1750 ms sin).

### Reglas duras

- Cancelar crop ≠ Aplicar (snapshot al entrar).
- No usar `role="button"` con botones anidados (accesibilidad).
- El zoom rueda usa listener nativo, no el de React.

### Consultar en el repo si dudas

- `EditorView.tsx` — TODO. Es el archivo más importante de esta fase.
- `PresentationView.tsx` — cómo se pasa el thumbnail real a la presentación.

### Checkpoint

- [ ] Pinza, pan, doble toque y rueda funcionan sin re-renderizar el árbol entero.
- [ ] Compare muestra la original al mantener pulsado (y no se queda pegado en botones).
- [ ] Crop con lupa: la cruz coincide con el punto de corte real.
- [ ] Cancelar crop restaura el quad del snapshot.
- [ ] Rotar tarda ~0.2 s y no reprocesa el worker.
- [ ] Merge en modo documento no vacía páginas nuevas (snapshot de ids).

---

## 7. Fase 6 — Persistencia y biblioteca

### Qué construir

1. **`page-store.ts`** con IndexedDB `escaner-ios` v2:
   - Stores `documents` (keyPath `id`) y `meta` (`initialized`, `manualOrder`).
   - Imágenes como **Blob**.
   - Abrir/cerrar la DB por operación.
   - Todo best-effort con catch silencioso.

2. **`dataUrlToBlob` SIN `fetch()`**:
   - Base64: `atob → Uint8Array → Blob`.
   - Percent-encoded (SVG de mocks): parse manual.

3. **`LibraryView.tsx`**:
   - Grid/lista con filtros (Recientes/Favoritos/A-Z/Manual + etiquetas).
   - Búsqueda con **debounce 200 ms** + caché `Map` del texto normalizado.
   - Papelera de 30 días con purga al hidratar.
   - Long-press con guard `e.target.closest("button")`.

4. **Migraciones localStorage**:
   - `escaner-settings-v2-maxq`: `exportQuality "alta" → "máxima"`.
   - `escaner-settings-v3-ocr-manual`: `ocrEnabled = false` en TODAS las instalaciones.
   - Claves separadas (una por migración).

### Reglas duras

- `loadAllDocuments()` devuelve `null` (instalación nueva, conserva mocks) o `[]` (vacía real).
- Página corrupta → omitir, no lanzar.
- Papelera: purga a los 30 días al hidratar.
- Migración de filtros legacy al hidratar.

### Consultar en el repo si dudas

- `page-store.ts` — todo.
- `LibraryView.tsx` — filtros, búsqueda, papelera, long-press.
- `store.ts` — migraciones de localStorage con claves separadas.

### Checkpoint

- [ ] Documentos persisten entre sesiones (o se pierden limpiamente sin IndexedDB).
- [ ] Los mocks (página demo) persisten cambios: renombrar, favoritos, tags.
- [ ] Papelera purga a los 30 días.
- [ ] Búsqueda responde fluida (debounce + caché).
- [ ] Una instalación vieja con OCR ON se migra a OFF **una sola vez**.

---

## 8. Fase 7 — OCR, HEIC y export

### Qué construir

1. **`ocr.ts`** dual:
   - Si data URL > 10 MB → directo a local.
   - Si `NEXT_PUBLIC_STATIC !== "1"` y existe servidor → `POST {basePath}/api/ocr`.
   - Fallback local: Tesseract.js UMD 5.1.1, `createWorker("spa+eng", 1)`, imagen reducida a 1600 px JPEG 0.92, `terminate()` SIEMPRE en `finally`.
   - `OCR_NO_TEXT = "(sin texto legible)"`; `ocrTextIsValid` rechaza vacío, sentinela, y respuestas que empiecen con "lo siento"/"i'm sorry"/"i apologize".

2. **OCR bajo demanda** (default OFF) + auto-OCR de fondo con gates (`!previewLoading`, `!autoOcrBusy`, 1 intento por página).

3. **Sheet "Texto"** con búsqueda resaltada, chips, "Reconocer de nuevo", "Copiar texto", y con N>1: OCR en todas + copiar todas (`── Página i ──`, UTF-8 con BOM).

4. **HEIC** con `heic2any@0.0.4` (import dinámico, chunk ~1.4 MB solo cuando hace falta):
   - Cascada nativa → rescate HEIC → re-decode del JPEG.
   - Toast "Convirtiendo HEIC…" solo si `looksHeic`.

5. **PDF (`pdf-export.ts`)**:
   - jsPDF `{compress: true}`.
   - Tamaño adaptativo: lado mayor 297 mm.
   - Presupuesto: `min(max(N,3), 8) MB`.
   - Escalera de re-encode (primer intento que cabe gana).
   - PNG de filtros embebido SIN re-encode.
   - JPEG sobre canvas con fondo blanco y `toBlob` async.
   - XMP (título, autor "Escáner", keywords=tags).
   - "Exportar todo": presupuesto 30 MB, portada A4 + índice + miniaturas + outline.

6. **TXT (`text-export.ts`)**: UTF-8 con BOM, separadores `── Página i ──`, share nativo con truncado a 4000 chars.

### Reglas duras

- OCR **apagado por defecto**.
- Gate `NEXT_PUBLIC_STATIC` evita el 404 antes de cada OCR en Pages.
- `toBlob` async siempre.

### Consultar en el repo si dudas

- `ocr.ts`, `pdf-export.ts`, `text-export.ts`.
- `image-processor.ts` — `fileToCaptureDataUrl` con HEIC.
- `CameraView.tsx` — `onFilePicked` con toast de conversión.

### Checkpoint

- [ ] Un HEIC de iPhone abre solo (convertido). Un `.jpg` con contenido HEIC también.
- [ ] OCR está OFF por defecto; el botón "Texto" extrae con Tesseract spa+eng.
- [ ] El PDF sale ≤8 MB con calidad "máxima" y las páginas conservan nitidez.
- [ ] El TXT abre bien en Excel/Notepad (BOM).

---

## 9. Fase 8 — Benchmark de dispositivo

### Qué construir

**`device-capability.ts`** con dos benchmarks:
- **CPU**: 12 rondas de `Float32Array` 256×256 con `sqrt + sin` (escala logarítmica: 20 ms → 1.0, 500 ms → 0.0).
- **Canvas**: 1024×1024, 4 gradientes + `toBlob` JPEG 0.9.

```
score = round(100 · (0.55·cpuScore + 0.20·coreScore + 0.25·memScore))
coreScore: navigator.hardwareConcurrency / 8
memScore: (deviceMemory − 1.5) / 6.5   (iOS no expone deviceMemory → 0.6)
tiers: ≥66 → high (cap 4032) · ≥33 → medium (3200) · si no low (3200)
```

Cache **7 días** en localStorage `escaner-device-cap-v1`. Una vez por sesión con `requestIdleCallback`. Mientras no hay medida → 4032.

El cap se aplica en caliente al worker (`config {maxWarpLongSide}`) y lo usa el editor en cada preview.

### Reglas duras

- **low NUNCA baja de 3200** (v6.1 lo prohibió: texto ilegible en Tecno de entrada).
- iOS sin `deviceMemory` → 0.6 (no 0).

### Consultar en el repo si dudas

- `device-capability.ts` — los dos benchmarks y la fórmula.

### Checkpoint

- [ ] En un dispositivo de gama alta → tier high, cap 4032.
- [ ] En gama baja → tier low, cap 3200 (no 2560).
- [ ] El worker recibe el cap en caliente sin reiniciar.

---

## 10. Fase 9 — v6.1 + v6.2 (captura fluida)

### Qué construir

1. **F-RES-PRIORITY (v6.1)** — `captureSmart`:
   - Guard `processingRef` + `cooldownRef` (1500 ms) ANTES del cooldown (si no, se descartan toques en silencio).
   - Frame A: `snapshotVideo` + medidas (downscale ≤400 px).
   - Foto hi-res: `Promise.race([takePhoto(), timeout 8000 ms])`.
   - Si NO hay foto → toast warning y usar el mejor frame por lapVar.
   - Con foto → **la foto SIEMPRE gana** (no hay gate).
   - Liberar frames perdedores con **R-14** (`canvas.width = canvas.height = 0`).
   - iOS/Safari sin `ImageCapture` → `<input type="file" capture="environment">`.
   - Toast único por sesión: *"En iPhone: para máxima calidad dispara manualmente (foto nativa)"*.

2. **F-DEFER-CROP (v6.2)** — `handleCaptureDataUrl`:
   - Decode ÚNICO, flash blanco + vibrate(30).
   - `downscaleImage` a 4032 si excede, `toBlob` 0.95.
   - `evaluateQuality` (canvas 120 px).
   - Crear página con `quad: defaultQuad()` + `autoQuadPending: true`.
   - `setView("editor")` INMEDIATO.
   - `void applyAutoQuad(page.id, source)` fire-and-forget.

3. **`applyAutoQuad`** en background:
   - Detectar bordes; si la página ya no existe → nada.
   - Si `quadManual` → solo `autoQuadPending: false`.
   - Si no → aplicar quad detectado (cambia la clave → reproceso automático).

4. **Pill "Ajustando recorte…"** en el editor mientras `autoQuadPending && mode==="review"` (`bottom-[176px]`; el de OCR va en `bottom-[128px]`).

### Reglas duras

- La foto del sensor SIEMPRE gana al frame de preview.
- Timeout 8 s (no 5 s: en gama baja nunca llegaba).
- Detección FUERA del camino crítico de captura.

### Consultar en el repo si dudas

- `CameraView.tsx` — `captureSmart`, `handleCaptureDataUrl`, `applyAutoQuad`, `notifyCaptured`.
- `frame-loop.ts` — `notifyCaptured`, rearm.
- `store.ts` — `addCapturePage`, `updateCapturePage`.

### Checkpoint

- [ ] Capturar en gama baja: el editor abre en <1 s.
- [ ] La página guardada es la del sensor, no un frame 720p.
- [ ] El auto-recorte aterriza en background (pill visible → desaparece → preview se actualiza solo).
- [ ] Recorte manual posterior se respeta al píxel tras re-filtrar/rotar.
- [ ] Con documento quieto: 4/6 muestras >0.8 en 1200 ms → auto-captura UNA vez.

---

## 11. Fase 10 — PWA y pulido

### Qué construir

1. **`public/sw.js`** network-first, tope de entradas en RUNTIME_CACHE (~80, purgar FIFO), excluir `/downloads/`, `cache.put` solo si `fresh.ok`.

2. **`public/manifest.webmanifest`** con iconos, nombre, theme color, display standalone.

3. **Registro del SW** en `pwa.ts`:
   - Si `document.readyState === "complete"` al montar → registrar YA (no depender del evento `load` que ya pasó).

4. **Linterna (F-FLASH v3)**:
   - Botón habilitado siempre en cámara real (sin `disabled`, solo `opacity-40`).
   - `setTorchState(on)`: aplicar → **verificar `getSettings().torch`** → si falló al encender, deshacer con `{torch:false}`.
   - Reintentos a `0 / 250 / 700 / 1500 ms` + en evento `playing` del `<video>`.
   - Toast exacto de fallo (8000 ms, icono 🔦) con las causas listas.

5. **`track.onended` / `track.onmute`**: detener frame loop, liberar `streamRef`, volver a `idle`, toast accionable + re-arranque completo (nonce).

6. **iOS específico**:
   - `typeof ImageCapture === "undefined"` → `canTakePhoto = false`, shutter abre cámara nativa.

7. **`frame-loop.ts`**: re-armar `firstAttemptMs` tras la PRIMERA detección exitosa; máximo 2 avisos "No detecto el documento" por sesión.

### Consultar en el repo si dudas

- `sw.js`, `manifest.webmanifest`.
- `pwa.ts` — registro del SW.
- `CameraView.tsx` — linterna, `track.onended`, iOS.

### Checkpoint

- [ ] PWA instala desde el navegador y funciona offline.
- [ ] Linterna funciona en Android aunque `capabilities.torch` no se anuncie.
- [ ] Toast de fallo de linterna lista las causas exactas.
- [ ] Pérdida de cámara a mitad → re-arranque completo sin quedar congelada.

---

## 12. Fase 11 — Verificación final (la más importante)

### Qué hacer

Recorre la **tabla de gotchas** (24 filas) una por una. Para cada una:

1. Lee el síntoma.
2. Comprueba si tu implementación lo reproduce.
3. Si lo reproduce, aplica la solución obligatoria.
4. Anota en un `QA.md` cómo lo verificaste (o por qué no aplica).

### Gotchas que más se olvidan

| # | Síntoma | Verificación |
|---|---|---|
| 3 | No cambia lente ni flash | Sondas SECUENCIALES cerrando cada stream. |
| 6 | Crash por memoria en Safari | `toBlob` async + liberar canvas con `width=height=0`. |
| 9 | Marco parpadea ~5 Hz | Conservar último quad; descartar frames (no emitir null). |
| 10 | Auto-captura infinita | `rearmNeeded` (score ≤0.8 o pérdida de detección). |
| 13 | OCR pega texto en página equivocada | `setPreview(null)` antes de procesar + OCR exige `!previewLoading`. |
| 15 | Imagen espejada al rotar (fallback) | Scale uniforme en `cropQuad`. |
| 16 | "Píxeles feos" en el papel | Mapa de ganancias bilineal + `TEXT_GAIN_MAX=4` + `APPLY_FACTOR_MAX=8`. |
| 20 | Pérdida de cámara a mitad | `onended` → toast + re-arranque. |
| 23 | Doble tap en Guardar cae en "No hay páginas" | Botón deshabilitado DURANTE la descarga + "seguro de vida" en `finally`. |
| 24 | Cancelar recorte ≡ Aplicar | Snapshot al entrar en crop. |

### Criterios de aceptación finales

- [ ] Capturar en gama baja: editor <1 s, página a resolución del sensor, texto legible.
- [ ] Recorte automático en background + recorte manual respetado al píxel.
- [ ] Auto-captura dispara UNA vez (cooldown 1.5 s, re-arm exige re-encuadre).
- [ ] Linterna funciona en Android sin `capabilities.torch`.
- [ ] HEIC de iPhone abre solo; `.jpg` con contenido HEIC también.
- [ ] Los 3 filtros producen los resultados descritos.
- [ ] Rotar procesada ~0.2 s.
- [ ] Matar worker no rompe la app.
- [ ] Sin IndexedDB todo funciona en memoria.
- [ ] OCR OFF por defecto; instalación vieja migra a OFF una vez.
- [ ] PDF ≤8 MB con calidad máxima.
- [ ] **Ninguna fila de la tabla §20 reproduce su síntoma.**

### Prueba manual obligatoria en:

- Chrome Android (cámara real, linterna, HEIC).
- iPhone Safari (cámara nativa, PWA, sin IndexedDB en modo privado).
- Un gama baja (Tecno o similar) con documento quieto y a distancia.

---

## 13. Reglas de trabajo para la IA

Estas reglas aplican en TODAS las fases:

1. **Nunca violes una regla dura** sin pedir permiso explícito y justificar con evidencia de dispositivo real.
2. **Cuando el spec diga `NOMBRE = valor`**, ese valor es sagrado. No lo "mejores".
3. **Cuando una decisión no esté en la guía ni en el repo**, pregunta antes de inventar.
4. **Al terminar cada fase:** `bun run lint` + `bunx tsc --noEmit` + `BUILD_STATIC=1 bunx next build` deben pasar limpios.
5. **La regla de oro manda:** si un cambio amenaza la resolución o la legibilidad, NO lo hagas.
6. **No mezcles documentos:** esta guía es para construir desde cero. El `INSTRUCCIONES-AGENTE` es para arreglar el repo existente. El `ESPECIFICACION-REPLICA` es para reconstruir con fidelidad.
7. **Todo en español:** código, comentarios, UI, toasts, errores.
8. **Estética iOS pixel-perfect:** paleta, tipografía, transiciones del spec.
9. **Sin backend obligatorio:** build 100 % estático para GitHub Pages.
10. **Antes de declarar terminado:** recorre la tabla §20 como checklist de QA.

---

## 14. Cómo consultar el repo cuando tengas dudas

El repo es tu referencia, no tu plantilla. Úsalo así:

1. **Antes de escribir un módulo nuevo**, mira cómo está estructurado en el repo:
   - ¿Qué exporta?
   - ¿Qué recibe?
   - ¿Qué constantes usa?
2. **Si el comportamiento no está claro en la guía**, busca el archivo correspondiente en el repo y léelo. No copies literalmente: entiende y reescribe.
3. **Si el repo contradice la guía**, gana la guía (está calibrada con bugs reales posteriores).
4. **Nunca pegues código del repo sin entenderlo.** Los comentarios del original explican *por qué* de cada decisión rara; esos comentarios son más valiosos que el código.

### Mapa rápido repo → fase

| Fase | Archivos clave del repo |
|---|---|
| 1 | `types.ts`, `store.ts`, `BottomNav.tsx`, `app/layout.tsx` |
| 2 | `CameraView.tsx` (apertura), `image-processor.ts` (básico), `quality.ts` |
| 3 | `detection-worker.js`, `detector-client.ts`, `frame-loop.ts` |
| 4 | `image-processor.ts` (filtros), `detection-worker.js` (enhance) |
| 5 | `EditorView.tsx`, `PresentationView.tsx` |
| 6 | `page-store.ts`, `LibraryView.tsx` |
| 7 | `ocr.ts`, `pdf-export.ts`, `text-export.ts`, `image-processor.ts` (HEIC) |
| 8 | `device-capability.ts` |
| 9 | `CameraView.tsx` (captureSmart, applyAutoQuad) |
| 10 | `sw.js`, `manifest.webmanifest`, `pwa.ts`, `CameraView.tsx` (torch) |
| 11 | Tabla §20 + `QA.md` que tú creas |

---

## 15. Veredicto final

| Objetivo | Cómo lo logra esta guía |
|---|---|
| Construir desde cero | Fases 1–10 secuenciales con checkpoints. |
| No inventar | Reglas duras + consulta al repo cuando hay duda. |
| No repetir bugs | Fase 11 recorre la tabla §20. |
| Fidelidad de comportamiento | Constantes sagradas + algoritmos exactos. |
| Fidelidad visual | Paleta iOS + consulta a `design-specs.md` del repo. |
| Independencia del repo | La guía se sostiene sola; el repo es apoyo. |

**En una frase:** esta guía te lleva de cero a un producto funcionalmente fiel en 11 fases, con checkpoints verificables y reglas duras que evitan los bugs históricos. El repo está ahí para cuando una decisión concreta no esté clara, pero no lo necesitas para entender el producto.

---

*Fin de la guía de construcción desde cero.*