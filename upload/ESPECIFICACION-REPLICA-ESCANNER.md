# ESPECIFICACIÓN MAESTRA DE RÉPLICA — Escáner de documentos web
### Documento de transferencia para IA: re-construye este producto EXACTAMENTE igual

> **Cómo usar este documento (leer primero):** esto NO es una idea vaga, es la extracción fiel del código de un producto en producción (v6.2). Si una IA lo sigue, debe reproducir comportamientos, constantes y órdenes EXACTOS — no aproximaciones. Cuando algo diga `NOMBRE = valor`, ese valor es sagrado: fue calibrado con bugs reales en dispositivos reales (iPhone, Tecno de gama de entrada, Chrome Android). Si tienes que elegir entre "mejorarlo" y "respetarlo", RESPÉTALO.
>
> **Regla de oro del producto (dada por su dueño):** *"podemos modificar otras cosas pero NUNCA la capacidad de poder ver bien la imagen y reconocer el texto"*. Toda optimización de rendimiento debe sacrificarse antes que la resolución de captura o la legibilidad del texto.

---

## ÍNDICE
1. El producto en una página
2. Stack y arquitectura
3. Modelo de datos exacto
4. REGLAS DURAS (intocables)
5. Cámara v2 — apertura, sondas, linterna
6. Frame loop y auto-captura k-de-n
7. captureSmart — la foto del sensor SIEMPRE gana (v6.1)
8. F-DEFER-CROP — captura fluida (v6.2)
9. Motor de visión — worker OpenCV (detección, refinado, warp)
10. Filtros de imagen — matemática exacta
11. Calidad (dos métricas distintas)
12. HEIC — cascada de decodificación
13. Editor — revisión, zoom, recorte con lupa, rotación instantánea
14. OCR — dual, bajo demanda
15. Persistencia — IndexedDB + migraciones
16. Benchmark de dispositivo — tiers
17. Exportar — PDF y TXT
18. Diseño visual (resumen)
19. Disciplina de memoria y rendimiento
20. Tabla de gotchas de compatibilidad (iOS/Android)
21. Historial de versiones (por qué las reglas existen)
22. Plan de construcción recomendado para la réplica
23. Criterios de aceptación (cómo saber que la réplica es fiel)

---

## 1. EL PRODUCTO EN UNA PÁGINA

**Qué es:** aplicación web (PWA-able) de digitalización de documentos estilo Adobe Scan, estética iOS pixel-perfect, que corre 100% en el navegador (sin backend obligatorio; se despliega como sitio estático).

**Flujo de usuario (patrón Adobe Scan):**
1. **Biblioteca** ("Mis documentos"): grid/lista de documentos, búsqueda (incluye texto OCR), filtros (Recientes/Favoritos/A-Z/Manual + etiquetas), papelera de 30 días.
2. **Cámara**: visor con detección de bordes en VIVO (marco azul), auto-captura cuando el encuadre es estable y nítido, linterna, cambio entre fotograma/foto nativa, importar galería.
3. **Editor** (abre AL INSTANTE tras capturar): preview procesado (recorte + filtro aplicados), gestos de zoom/compare/swipe, toolbar (Repetir · Recortar · Rotar · Filtros · Texto · Imagen · Eliminar), multi-página ("Seguir escaneando"), "Guardar PDF".
4. **Ajustes**: Mejora automática (default ON), OCR automático (default **OFF**), calidad de export (default "máxima"), benchmark del dispositivo, búsqueda interna.

**Decisiones de producto que definen el feel:**
- Capturar → editar es INSTANTÁNEO; todo lo pesado (detección de bordes, filtros) es asincrónico y aterriza solo.
- La foto guardada SIEMPRE es la de resolución del sensor (nunca un fotograma de preview).
- El recorte automático funciona en background; si el usuario recorta a mano, su decisión manda al píxel.
- Todo degrada con gracia: sin worker → canvas local; sin IndexedDB → memoria; sin servidor → Tesseract local. Nada lanza errores al usuario por falta de soporte.

---

## 2. STACK Y ARQUITECTURA

| Capa | Elección |
|---|---|
| Framework | Next.js 16 App Router + TypeScript estricto (export estático `output: "export"`, basePath `NEXT_PUBLIC_BASE_PATH` para GitHub Pages) |
| Estado | Zustand (una sola store, vistas: `library/camera/editor/settings`) |
| UI | Tailwind CSS 4 + framer-motion + sonner (toasts) + vaul (sheets) + lucide-react |
| Visión | **Web Worker clásico** (`public/scanner/detection-worker.js?v=7`) con **OpenCV.js 4.5.5 self-hosted** (`public/vendor/opencv-4.5.5.js`, fallback CDN `https://docs.opencv.org/4.5.5/opencv.js`) |
| OCR | Tesseract.js UMD **5.1.1** desde `https://cdn.jsdelivr.net/npm/tesseract.js@5.1.1/dist/tesseract.min.js` (spa+eng, OEM 1) |
| HEIC | `heic2any@0.0.4` con **import() dinámico** (chunk ~1,4 MB solo cuando hace falta) |
| PDF | jsPDF (`{compress:true}`) |
| Persistencia | IndexedDB `escaner-ios` v2 (imágenes como **Blob**) + localStorage (ajustes) |

**Módulos** (`src/lib/scanner/`): `image-processor.ts` (pipeline), `detector-client.ts` (cliente del worker, singleton self-healing), `quality.ts` (score del obturador), `frame-loop.ts` (CameraFrameLoop + SyntheticCamera), `ocr.ts`, `device-capability.ts`, `page-store.ts` (IndexedDB), `pdf-export.ts`, `text-export.ts`, `types.ts`, `store.ts` (Zustand). Componentes: `CameraView.tsx`, `EditorView.tsx`, `SettingsView.tsx`, `LibraryView` etc.

**QA hooks de navegador:** `window.__cameraChoice`, `window.__cameraTelemetry`, `window.__scannerPrecision()` → `{ready, dead}`, `window.__deviceCapability()`, `window.__scannerStore()`.

---

## 3. MODELO DE DATOS EXACTO (`types.ts`)

```ts
type ScannerView = "library" | "camera" | "editor" | "settings";  // NO hay vista "detail"
type PageFilter = "original" | "text" | "bw";   // EXACTAMENTE 3 (error #13: 8 presets rompían todo)
// FILTER_PRESETS: original/"Original"/"Sin cambios" · text/"Texto claro"/"Papel blanco, tinta marcada" · bw/"B/N adaptativo"/"Blanco y negro puro, inmune a sombras"
const PNG_FILTERS = new Set(["original", "text", "bw"]);  // el procesado SIEMPRE sale PNG (R-10)

interface Point { x: number; y: number }                 // normalizado 0–1
type Quad = [Point, Point, Point, Point];                // orden FIJO: TL, TR, BR, BL
function defaultQuad(): Quad;                            // [{0.08,0.10},{0.92,0.06},{0.95,0.92},{0.05,0.95}]

interface PageQuality { level: "excellent"|"good"|"fair"|"poor"; sharpness: number; brightness: number; contrast: number; label: string; }
interface PagePrecision { engine: "worker"|"canvas"; refined: boolean; cornersSolid: number|null; width: number; height: number; elapsedMs?: number; }

interface ScanPage { id: string; original: string; processed: string; thumbnail: string;
  filter: PageFilter; quad: Quad; quadManual?: boolean; rotation: 0|90|180|270;
  quality: PageQuality; precision?: PagePrecision; ocrText?: string; ocrDone: boolean; createdAt: number; }

interface ScanDocument { id: string; title: string; pages: ScanPage[]; favorite: boolean; tags?: string[];
  createdAt: number; updatedAt: number; deletedAt?: number; }   // deletedAt ⇒ papelera, 30 días

interface CapturePage extends-esencia-de-ScanPage {  // sesión de captura (pre-guardado)
  processed?: string; processedKey?: string; thumbnail?: string; ocrText?: string; ocrDone?: boolean;
  autoQuadPending?: boolean;   // v6.2, runtime-only: NO persiste, NO viaja a ScanPage
}

interface ScannerSettings { enhance: boolean; ocrEnabled: boolean; exportQuality: "standard"|"alta"|"máxima"; }
const DEFAULT_SETTINGS = { enhance: true, ocrEnabled: false, exportQuality: "máxima" };
```

**Clave de estado (ÚNICA fuente, la comparten preview/merge/rotación):**
```ts
capturePageKey(p) = `${p.id}|${p.quad.map(q => `${q.x.toFixed(4)},${q.y.toFixed(4)}`).join(";")}|${p.filter}|${p.rotation}`
```
El contrato `page.processedKey === capturePageKey(p)` significa "la procesada persistida corresponde a ESTE estado exacto". Rotar/filtrar/recortar cambia la clave → cache-miss → reproceso. TODO el sistema de previews depende de esto.

**Migración de filtros legacy** (`normalizePageFilter`): `auto|document|whiteboard→text`, `blackwhite→bw`, `natural|color|grayscale→original`, `default→bw`.

**IDs:** `` nextId(prefix) = `${prefix}-${Date.now().toString(36)}-${++uid}` ``.

---

## 4. REGLAS DURAS (violación = producto roto)

1. **Resolución y legibilidad intocables.** La foto guardada es la del sensor; jamás un fotograma de preview de 720p/1080p puede sustituirla (v6.1). Los caps de procesado (4032/3200) NUNCA bajan para "rendir más" (el low tier estuvo en 2560 y el dueño lo prohibió: texto ilegible).
2. **El recorte manual manda al píxel.** `quadManual: true` ⇒ warp SIN refine sub-píxel y SIN shrink de 3.5 px (no mover las esquinas que el humano puso, no recortar contenido).
3. **SOLO 3 filtros.** Añadir presets = error #13 repetido.
4. **Procesado siempre PNG** (R-10); JPEG solo donde la lista lo dice (§9–10).
5. **`toBlob` primero, `toDataURL` solo como respaldo** (error #22: `toDataURL` sobre canvas grande revienta memoria en Safari/iOS). Encodes async siempre.
6. **Backpressure por DESCARTE, nunca cola** (los ImageBitmap transferidos no se pueden duplicar).
7. **Nunca 2 cámaras abiertas a la vez** (sondas secuenciales; bug v3: `NotReadableError` en Android).
8. **Over-constraining prohibido en getUserMedia:** SOLO `width: {ideal: 3840}` (E3). El alto lo negocia el navegador.
9. **El OCR es BAJO DEMANDA por defecto** (`ocrEnabled: false`, v6.1) — memoria en gama baja.
10. **Detección de bordes FUERA del camino crítico de captura** (v6.2): el editor abre al instante.
11. **Todo degrada en silencio:** ninguna función de persistencia/pipeline lanza al usuario por falta de soporte.
12. **Cero re-decodes evitables:** la ruta crítica de captura decodifica la foto UNA sola vez.

---

## 5. CÁMARA V2 — APERTURA, SONDA SECUENCIAL, LINTERNA

### 5.1 Constantes
```
IDEAL_CAPTURE_WIDTH = 3840        // E3: SOLO ancho ideal (sin height/frameRate/aspectRatio)
BACK_CAMERA_RE  = /back|rear|environment|trasera|posterior|arri[eè]re/i
FRONT_CAMERA_RE = /front|delantera|anterior|face|facial|selfie/i
REAL_AF_MODES   = Set("continuous", "single-shot")   // regla D3: único AF "real"
```

### 5.2 Flujo de apertura (el "mecanismo v2" — port del codigo-test del dueño)
1. Sin `getUserMedia` o sin `isSecureContext` → cámara sintética directo.
2. **`openMainCamera()`** (camino F-LENS v4):
   - **Desbloqueo de labels:** `getUserMedia({video:true})` + `stop()` inmediato (sin permiso previo `enumerateDevices` da labels vacíos).
   - **Sondeo SECUENCIAL:** para cada `videoinput`: abrir con `deviceId: {exact}` **SIN constraints de resolución** (medir el sensor pelado), leer `getCapabilities()` → `focusModes[], torch, width.max, height.max`; en `finally` **cerrar SIEMPRE el stream antes de sondear la siguiente** (causa raíz del bug v3: sondas con stream vivo → `NotReadableError` → nada funcionaba).
   - **`chooseMainProbe`:** traseras que no matcheen FRONT; si alguna tiene AF real → la de **mayor resolución de sensor** (torch desempata); si NINGUNA tiene AF (típico iOS): filtrar labels con palabras de lente `lensWords = /ultra|gran angular|wide|angular|tele|teleobjetivo/i` (quedan las "simples" = la principal), ordenar por menos palabras del label, luego por resolución.
   - **Abrir SOLO la ganadora** con cascada: `[{deviceId:{exact}, width:{ideal:3840}} → {deviceId:{exact}} → {video:true}]`.
   - **Zoom a 1×:** si `capabilities.zoom.min < 1 && settings.zoom < 1` → `applyConstraints({advanced:[{zoom:1}]})` (equivale al 0.5× del mismo track; errores ignorados).
3. Si `openMainCamera` falla → **`openWithCascade()`** (E3/E4): `[{facingMode:{exact:"environment"}, width:{ideal:3840}} → {facingMode:"environment", width:{ideal:3840}} → {facingMode:"environment"} → {video:true}]`.
4. Si todo falla con permiso denegado → aviso persistente con botón "Activar cámara" (retry real en caliente); siempre caer a cámara sintética (demo) → escena estática simulada.
5. `track.onended` → toast "Se perdió la cámara"/"Reconectando…" + re-arranque completo del flujo (nonce). `stop()` NO dispara "ended".
6. **iOS/Safari:** `typeof ImageCapture === "undefined"` → `canTakePhoto = false`; el shutter abrirá la **cámara nativa** vía `<input type="file" accept="image/*" capture="environment">` (el click del shutter ES el gesto que iOS exige). Toast único por sesión: *"En iPhone: para máxima calidad dispara manualmente (foto nativa)"* (6000 ms).

### 5.3 Linterna (F-FLASH v3) — la verdad se descubre APLICANDO
- El botón flash está **habilitado siempre en cámara real** (hay Chrome que NO anuncia `torch` en capabilities pero SÍ lo aplica). Sin `disabled` (para poder mostrar el toast), solo `opacity-40`.
- `setTorchState(on)`: 1) `applyConstraints({advanced:[{torch:on}]})` (advanced = best-effort, no rechaza); 2) **verificar `getSettings().torch`** (encender exige verificación real — existía el "flash fantasma"); 3) si falló al encender → **deshacer** con `{torch:false}` para no dejar LED fantasma.
- **Reintentos tras abrir:** en `0 / 250 / 700 / 1500 ms` (varios Android rechazan justo tras getUserMedia) + re-intento con el evento `playing` del `<video>`.
- Mensaje de fallo exacto (toast 8000 ms, icono 🔦): *"No se pudo controlar la linterna aquí. Causas típicas: navegador sin soporte (en iPhone usa Safari 17.4 o posterior; Chrome/Firefox de iOS no lo permiten), cámara abierta sin LED (gran angular o macro) o la app corre dentro de otra app (Instagram, WhatsApp…). Cierra el escáner y vuelve a abrirlo; si persiste, prueba en otro navegador."*

---

## 6. FRAME LOOP Y AUTO-CAPTURA k-de-n

**`CameraFrameLoop`** (por frame NUEVO del sensor — `requestVideoFrameCallback`, fallback rAF):
1. Si worker no ready o **busy** → **descartar frame** (`dropped+1`), NO encolar, NO emitir null (el overlay conserva el último quad; emitir null = parpadeo del marco a ~5 Hz).
2. Downscale del frame a lado mayor **400 px** (`createImageBitmap(..., {resizeQuality:"low"})`) → `detect` al worker (bitmap transferido).
3. Histograma de exposición del MISMO frame en main thread: miniatura de **96 px** → 256 bins luma BT.601 (0.299/0.587/0.114).
4. Telemetría a UI **throttled 100 ms (~10 Hz)**; historiales `quadHistory/scoreHistory`: ventana **2000 ms**, cap **24**.
5. Sin detección > **8000 ms** → aviso "No detecto el documento · acércalo más al encuadre", **máx 2 por sesión** (B9), reloj reinicia con cada detección.

**Score compuesto (`quality.ts`):**
```
score = (0.4·sharpness + 0.3·exposure + 0.3·stability) × eccentricity      // clamp 0–1
sharpness  = min(1, Var(Laplacian)/300)            // SHARPNESS_NORM = 300; isBlur si lapVar crudo < 100
exposure   = 1 − (under+over)/total                // under < 30, over > 225; especular > 248 (warn si > 3%)
stability  = 1 − clamp(mediaVarianzas/20)          // ventana 600 ms POR TIMESTAMP (el índice mienta con descartes)
eccentricity = MIN de las 4 esquinas de dMin/(0.05·ladoCorto)   // penaliza documento pegado al borde
```

**Disparo (k-de-n):**
```
SHUTTER_SCORE = 0.8    SHUTTER_K = 4    SHUTTER_N = 6    SHUTTER_SPAN_MS = 1200    CAPTURE_COOLDOWN_MS = 1500
```
Dispara si: la ÚLTIMA muestra > 0.8 **y** ≥ 4 de las últimas 6 dentro de 1200 ms > 0.8. Tras capturar (`notifyCaptured`): borrar `scoreHistory` y exigir **rearm** (score ≤ 0.8 o perder detección) para volver a armar. **NO tocar `quadHistory`** (vaciarla haría caer stability y liberaría el rearm en falso).
- Fallback sin worker: ciclo estable 2600 ms ON / 9800 ms OFF; al volverse estable: `setTimeout(1500)` → cooldown 7000 ms → capturar. Quad de decoración con jitter ±0.01 cada 700 ms.
- Auto-captura es **estado local de sesión** (default ON); toggle en top bar (icono Scan, amarillo #ffd60a activo) y en menú ⋮.

---

## 7. captureSmart — LA FOTO DEL SENSOR SIEMPRE GANA (v6.1, F-RES-PRIORITY)

Contexto del bug que esto mató: en gama de entrada, si la foto full-sensor salía "blanda", el gate de nitidez de la v5 prefería un **frame de preview de 720p "más nítido"** → las páginas salían ilegibles. **La resolución es intocable: una foto algo blanda a 3264 px siempre supera a un frame perfecto de 720p (el enhance afina).**

Flujo exacto:
1. Guardas: `processingRef` y `cooldownRef` (1500 ms). `notifyCaptured()` + cooldown.
2. **Frame A** = `snapshotVideo()` (canvas a resolución exacta del track) + medidas (downscale ≤400 px: exposición under<30/over>225, lapVar Laplaciano 3×3).
3. **Foto hi-res**: si hay ImageCapture → `Promise.race([takePhoto(), timeout 8000 ms → "takePhoto timeout 8s"])` (eran 5 s y en gama baja nunca llegaba).
4. **Si NO hay foto** (colgó >8 s o no existe ImageCapture): toast warning *"Cámara lenta: baja resolución esta vez — la foto de alta resolución no respondió; se guardó el fotograma de vista previa."* → usar el mejor frame por lapVar (encode `toBlob("image/jpeg", 0.92)` del GANADOR solo; `releaseFrame` de los perdedores). El fotograma es "un recurso, no el estándar".
5. **Con foto:** la foto gana SIEMPRE (no hay gate). Frames A y B se liberan con **R-14** (`canvas.width = canvas.height = 0` — soltar el backing store YA, no esperar al GC).
6. `handleCaptureDataUrl(photoUrl)`.

**Shutter por plataforma:** con stream + `canTakePhoto` → captureSmart; Safari/iOS (sin ImageCapture) → cámara nativa `capture="environment"`; escena simulada → página demo generada.

---

## 8. F-DEFER-CROP — CAPTURA FLUIDA (v6.2)

**Antes (lento en gama baja):** capturar → esperar decode + detección OpenCV → abrir editor (segundos mirando el spinner).
**Ahora:**

`handleCaptureDataUrl(rawDataUrl)` — camino crítico:
1. Flash blanco + `vibrate(30)`.
2. **Decode ÚNICO** (`loadImage`) — antes la foto de 12 MP se decodificaba 3 veces.
3. `downscaleImage(decoded, max 4032)` — solo si excede; `toBlob("image/jpeg", 0.95)` async; `null` si no hace falta (flujo normal de captura: sin re-encode).
4. `evaluateQuality(source)` — SÍ va en camino crítico: es barata (canvas **120 px**).
5. Crear página con **`quad: defaultQuad()` (marco provisional) + `autoQuadPending: true`** y `filter = settings.enhance ? "bw" : "original"`.
6. `addCapturePage` → **`setView("editor")` INMEDIATO**.
7. `void applyAutoQuad(page.id, source)` — fire-and-forget.

`applyAutoQuad` (background):
1. `detected = await detectDocumentEdges(src)`.
2. Si la página ya no existe (Repetir/limpiar) → nada.
3. Si `page.quadManual` → solo `autoQuadPending: false` (**la decisión manual manda**).
4. Si no → `apply({ quad: detected, autoQuadPending: false })` — cambia `capturePageKey` → el preview se re-procesa solo (mecánica del editor).
5. Si falla → liberar pill; queda el marco provisional (ajustable en «Recortar»).

**Editor:** pill flotante **"Ajustando recorte…"** mientras `autoQuadPending && mode==="review"` (posición `bottom-[176px]`; el pill de OCR va en `bottom-[128px]` — no chocan). La galería hereda el mismo pipeline (mismo `handleCaptureDataUrl`).

---

## 9. MOTOR DE VISIÓN — WORKER OPENCV

### 9.1 Cliente (`detector-client.ts`)
- Worker: `${NEXT_PUBLIC_BASE_PATH ?? ""}/scanner/detection-worker.js?v=7` (cache-buster `?v=7`; basePath para Pages).
- **Singleton self-healing:** si el cliente `isDead` (timeout de arranque **25 s**, error de carga) → `getScannerWorker()` crea uno NUEVO. `waitReady()` espera `{type:"ready"}`.
- Serialización: promise-chain = **1 mensaje en vuelo** (`busy` si hay pendientes). El frame loop descarta si busy (los bitmaps transferidos no se pueden duplicar).
- El worker cierra SIEMPRE el bitmap de entrada en `finally`; los Mats OpenCV se registran y `.delete()` en LIFO (`withMats`); los datos salen COPIADOS del buffer WASM.

### 9.2 Protocolo
- **In:** `{detect, bitmap, ts}` · `{warp, bitmap, quad:Float32Array(8) fracciones, manual?, ts}` · `{enhance, bitmap, mode, maxLongSide?, opts?, ts}` · `{config, maxWarpLongSide?}` (sin respuesta).
- **Out:** `{boot, pct}` · `{ready, probe, opencvUrl}` · `{busy, ts}` · `{error, message}` · `{result, corners:Float32Array(8)|null, qualityInput:{laplacianVar, cropMean, cropStdDev}}` · `{warped, bitmap (transferido), w, h, refined, fellBack}` · `{enhanced, blob, mime, w, h, elapsedMs, memory}`.
- El cap de warp es ajustable EN CALIENTE vía `config { maxWarpLongSide }` (acepta ≥256): tier high→4032, medium/low→3200.

### 9.3 Detección (`processFrame`) — cascada multi-umbral
Pre: RGBA→gris → `GaussianBlur(5×5)`. Candidatos: contornos ≥ **0.5%** del frame, **top 8 por área**, `approxPolyDP(eps = epsRatio·arcLength)`, solo **4 vértices**.

**6 pasadas (se corta en la primera que encuentre quad válido):**
```
{low:50, high:150, dilate:0, eps:0.02}     // pipeline histórico (coste cero en el caso feliz)
{low:50, high:150, dilate:2, eps:0.02}     // dilate 3×3×2 puentea bordes rotos
{low:50, high:150, dilate:2, eps:0.05}
{low:30, high: 90, dilate:0, eps:0.012}
{low:30, high: 90, dilate:2, eps:0.02}
{low:20, high: 60, dilate:2, eps:0.03}
```

**Scoring del quad (`selectQuad`):**
```
score = areaRelativa + 0.15·(aspectMatchaPrior ? 1:0) + 0.25·blancuraInterior
// blancura por píxel: luma/255 · (1 − saturación·2.2), promediada en bbox∩quad
```
Priors de aspecto (config; la app queda siempre en `auto` = sin prior): documento-largo h/w 1.3–4.5 · página long/short 1.2–1.6 · tarjeta h/w 0.6–0.8.

**Validación del quad:** convexo estricto, sin auto-intersección, área > **10%** del frame (F5-CROP: el 25% histórico no llenaba el encuadre a distancia normal), cada lado ≥ 5% del lado mayor. Ordenación de esquinas: por sumas/diferencias con fallback angular.

### 9.4 Refinado sub-píxel (`refineQuad`, solo warp NO manual)
- Bandas rectangulares por lado: ancho `max(30 px, 1.5% del lado)`.
- Por banda: `Canny(50,150)` → puntos (stride para tope 2000) → si ≥ **20 puntos**: **RANSAC** (inlier ≤ 5 px, máx 24 modelos, least-squares final); si no → least-squares recortando 12% por extremo.
- Esquinas = intersección de líneas adyacentes; las que fallan caen al quad original (`fellBack[i]`).
- Cota de movimiento por esquina: dentro de **5% de la diagonal** de la foto y desplazamiento ≤ `2·max(bandW_anterior, bandW_propia, 30 px)`; fuera → esquina original.

### 9.5 Warp
- **`SHRINK_QUAD_PX = 3.5`** por lado (solo quad NO manual; cap de desplazamiento 35 px por esquina) — para no incluir fondo del borde.
- Dims de salida: `w0 = max(|TL→TR|,|BL→BR|)`, `h0 = max(|TR→BR|,|TL→BL|)`, `scale = min(1, cap/max)` (**nunca upscalar**), mínimos 16 px.
- `getPerspectiveTransform` → `warpPerspective` **INTER_CUBIC** → RGBA **PURO sin unsharp** (F5-RAW-2: la foto es la fuente de verdad; el realce va según el filtro después).

### 9.6 Ruta fallback (sin worker) — misma matemática en canvas 2D
- Homografía **DLT de 4 puntos** (sistema 8×8, gauss con pivoteo) + muestreo **BILINEAL** con centros +0.5 y edge-replicate.
- Si la homografía degenera → recorte bbox del quad (con scale UNIFORME en rotación — bug E6: un `ctx.scale` no uniforme ESPEJABA en 90/270).
- Salida PNG; `precision.engine = "canvas"`.
- Encode seguro: `toBlob` con fallback `toDataURL` (error #22).

---

## 10. FILTROS DE IMAGEN — MATEMÁTICA EXACTA

Mapeo producto→motor: `original→raw` · `text→text` · `bw→bw`. Modelo de sombras COMÚN (todos menos raw):
- Luma entera `gray = (77R+150G+29B)>>8`.
- Mapa de iluminación: downscale a lado mayor **800 px** → **close morfológico separable** (min/max deslizante O(n)) con kernel `odd(max(3, 800/32)) = 25`.
- Ganancias `mean(map)/map[i]` aplicadas **BILINEAL** (F-TEXT-SMOOTH: nearest producía bloques ~5 px "píxeles feos").
- Unsharp ANTES del filtro (todos menos raw): `GaussianBlur(k=7, σ=1.5)` + `src·1.5 − blur·0.5`.

| Filtro | Pipeline | Salida |
|---|---|---|
| **original** (raw) | passthrough + alpha=255, **SIN unsharp** (opción `unsharpOriginal` → unsharp cliente 0.5/1.5/k7 + PNG) | PNG |
| **text** «Texto claro» | sombras → white-point stretch **p85** → S-curve `out=((v−0.72)·1.8+0.72)·255` → black point **0.2** → ganancia por píxel `min(4, ink/gray)` (TEXT_GAIN_MAX=4) → aplicar factor total **min(8)** conservando CROMA | PNG |
| **bw** «B/N adaptativo» | sombras → **Bradley-Roth**: ventana **w/12** impar, media por imagen integral, negro si `v ≤ m·(1−0.15)` (BW_T=0.15) → **despeckle** flood-fill 4-conectividad, componentes < **3 px** → blanco | PNG |

Constantes: `TEXT_CLARO_WHITE_PCT=0.85, TEXT_CLARO_CONTRAST=1.8, TEXT_CLARO_PIVOT=0.72, TEXT_CLARO_BLACK_POINT=0.2, TEXT_GAIN_MAX=4, APPLY_FACTOR_MAX=8, BW_T=0.15, BW_WINDOW_RATIO=1/12`.
(Sauvola opcional `k=0.34, r=128`; modo `color` con Lab L* + CLAHE 8×8 clip 2 existe en el motor pero el producto no lo expone.)
Encode del enhance: `raw|text|bw → PNG`; resto JPEG **0.9**. Thumbnails: **160 px JPEG 0.8**.

---

## 11. CALIDAD — DOS MÉTRICAS DISTINTAS (¡no confundir!)

1. **Score del obturador** (§6, `quality.ts`): decide la auto-captura en vivo (0.4/0.3/0.3 × eccentricity, k-de-n).
2. **Badge de la página** (`evaluateQuality`): canvas de **120 px** de ancho; lapVar Laplaciano 4-neighborhood sobre luma:
```
sharpness  = min(100, round(σ_lap · 2.2))
contrast   = min(100, round(σ_gray · 1.4))
brightness = min(100, round(mean/255 · 100))
score = sharpness·0.45 + brightness·0.25 + contrast·0.3
niveles: ≥80 excellent "Excelente" · ≥62 good "Buena" · ≥45 fair "Aceptable" · si no poor "Baja"
```
Badge visible si nivel poor/fair: pill naranja *"Calidad {label.toLowerCase()} · mantén el pulso y repite la captura"*. Error → `{good, 70, 70, 70, "Buena"}` (silencioso).

---

## 12. HEIC — CASCADA DE DECODIFICACIÓN (`fileToCaptureDataUrl`)

1. **Nativa:** `createImageBitmap(blob, {imageOrientation:"from-image"})` (EXIF) → si no soporta opciones → `createImageBitmap(blob)` → si falla → `objectURL → <img> → canvas` (el navegador aplica EXIF al pintar). Salida: JPEG **0.92** capado a **4032**.
2. **Rescate HEIC:** `heic2any({blob, toType:"image/jpeg", quality:0.92})` (**import() dinámico** — chunk ~1,4 MB solo cuando una imagen lo necesita; libheif inline sin .wasm, funciona en hosting estático) → **re-decode nativa completa** del JPEG resultante.
3. Errores: si `looksHeic = /heic|heif/i.test(type) || /\.hei[cf]$/i.test(name)` → *"No se pudo convertir la imagen HEIC"*; si no → *"No se pudo decodificar la imagen (formato no soportado o archivo corrupto)"*.
4. Toast de progreso SOLO si `looksHeic`: *"Convirtiendo HEIC… (puede tardar unos segundos)"*. Los «.jpg» de iPhone que traen contenido HEIC caen al rescate igualmente (heurística por extensión/MIME para el toast).

---

## 13. EDITOR

### 13.1 Modo revisión
- Fondo negro; header con título/fecha/compartir; stage de preview; pill "Página X de Y" con flechas; carrusel de miniaturas plegable (thumbs 64×48, activa con borde azul + glow; `CSS_FILTERS` cosmético sobre la original: `original:"none", text:"brightness(1.12) contrast(1.35)", bw:"grayscale(1) contrast(2.6) brightness(1.05)"`).
- **Preview:** cache **LRU de 12 entradas** (`Map` + expulsar el más viejo) — sin tope, los data URLs PNG grandes provocaban **jetsam en iOS**. Cache-miss → `setPreview(null)` ANTES de procesar (evita que el OCR lea el preview de la página ANTERIOR) + loader → `processImage(original, quad, filter, rotation, {manual: quadManual===true, maxLongSide: getMaxProcessedLongSide()})`; error → mostrar la original.
- **Toolbar de 7:** Repetir (rojo; elimina y vuelve a cámara) · Recortar · Rotar · Filtros (sheet) · Texto (OCR; activo si `ocrDone`) · Imagen (descarga PNG/JPG de la página) · Eliminar (rojo). Botones grandes: "Seguir escaneando" (o "Añadir página" en modo documento) + **"Guardar PDF"** (flex 1.25, azul).
- **Rotación instantánea (~0,2 s):** rotar 90° la procesada en cache es píxel-idéntico al reproceso completo (los filtros son conmutativos con rotaciones de 90°) → rotar el data URL en canvas, re-encode PNG async, sembrar la cache con la NUEVA clave, una sola `updateCapturePage({rotation, processed, processedKey, thumbnail})`. El preview/guardado NO vuelven a llamar al worker.
- Overlay de éxito al guardar (auto-cierra 3400 ms con acciones / 1750 ms sin) con "Escanear otro" (modo lote) y "Descargar PDF".
- Merge en modo documento: snapshot de ids ANTES de los awaits (B4); geometría idéntica → conservar tal cual con OCR; `processedKey` vigente → fusionar sin reprocesar (corrigiendo width/height de precision si la rotación intercambió ejes); si no → reprocesar. El texto OCR viejo NO se borra al re-recortar (queda pendiente, decide el usuario).

### 13.2 Gestos (F-ZOOM)
```
ZOOM_MIN=1  ZOOM_MAX=6  ZOOM_DOUBLE_TAP=2.5  TAP_SLOP=8px  SWIPE_MIN_PX=56  SWIPE_RUBBER=72px  COMPARE_HOLD_MS=350
```
- **Pinza** 2 dedos 1×–6× anclada al punto medio; al quedar 1 dedo → re-arranca como pan.
- **Pan** con 1 dedo a >1×, clampeado a los bordes de la IMAGEN; a 1× → **swipe de páginas** (decisión de eje una sola vez: `|dx| > |dy|·1.2`), goma elástica 72 px, spring de retorno (stiffness 500, damping 42).
- **Doble toque** (<320 ms entre taps, movimiento ≤8 px y <280 ms) alterna 1× ↔ 2.5× centrado en el punto.
- **Rueda escritorio:** factor `exp(−deltaY·0.0016)`, anclada al cursor, listener NATIVO `{passive:false}` (el `onWheel` de React es pasivo — B10).
- **Compare antes/después:** mantener pulsado a 1× (350 ms) → vibrate(18) + overlay con la ORIGINAL (0.14 s). Gotcha B1: el guard `e.target === e.currentTarget` NUNCA se cumplía (el stage siempre está cubierto por hijos) → capturar punteros salvo `target.closest("button")`.
- `setPointerCapture` siempre en try/catch (algunos navegadores lanzan si el puntero ya se soltó).

### 13.3 Modo recortar (crop)
- Overlay del quad sobre la **ORIGINAL** (con filtro CSS cosmético), contenedor que rota con la rotación de la página.
- 4 handles de esquina (pad táctil **44×44**, anillo pulsante) mueven UN vértice; 4 puntos medios trasladan la arista completa. `pointerToNormalized` invierte la rotación CSS (matriz de ejes, Y abajo).
- **Lupa 3×** (F4, validada por humano): radio **84 px**, ampliación **3×**, aparece en el lado VERTICAL OPUESTO al dedo, con crosshair amarillo marcando el punto de corte real.
- **Al SOLTAR** un handle: persistir en el store `{quad: display, quadManual: true}` (no al Aplicar). Por eso "Cancelar" restaura un **snapshot tomado al entrar** en crop (sin él, Cancelar ≡ Aplicar).
- Botón "Detección automática": `detectDocumentEdges(page.original)` → aplica + `quadManual: false` + badge "Bordes detectados" (visible 4 s, anclado a la esquina TL).
- Cambio de quad (no dragging) → tween rAF **280 ms** easeOutCubic.

---

## 14. OCR — DUAL, BAJO DEMANDA

**Default OFF** (`ocrEnabled: false`, v6.1) + migración one-time `escaner-settings-v3-ocr-manual` (§15). El usuario extrae texto con el botón **«Texto»**; si re-activa el automático, un efecto de fondo lo hace con gates.

**`requestOcr(image)` — dual:**
1. Si data URL > **10 MB** → directo a local.
2. **Servidor** (si existe y `NEXT_PUBLIC_STATIC !== "1"`): `POST {basePath}/api/ocr` (vision-model; mejor calidad). En builds estáticos el gate evita un round-trip 404 antes de CADA OCR (A2).
3. **Fallback local** (Tesseract.js UMD **5.1.1**, script inyectado una vez, reintentable si falla la red): `createWorker("spa+eng", 1)` (OEM LSTM); imagen reducida a lado mayor **1600 px** JPEG 0.92; `worker.terminate()` SIEMPRE en finally.
4. `OCR_NO_TEXT = "(sin texto legible)"`; `ocrTextIsValid` rechaza vacío, el sentinela, y respuestas que empiezan con "lo siento"/"i'm sorry"/"i apologize" (el vision-model a veces se disculpa en vez de admitir que no hay texto).

**Auto-OCR de fondo** (solo si el usuario lo activó): gates `ocrEnabled && page && !ocrRunning && !previewLoading && !autoOcrBusy && ocrDone!==true && processedKey===undefined && !autoOcrTried.has(page.id)` (solo páginas recién capturadas; máximo 1 OCR en vuelo; 1 intento por página). Pill "Reconociendo texto…" (`bottom-[128px]`).

**Sheet «Texto»:** texto con búsqueda resaltada `<mark>` amarillo #ffd60a, chips de palabras/caracteres/coincidencias, "Reconocer de nuevo", "Copiar texto", y con N>1: "OCR en todas las páginas" (secuencial con progreso {i,n}) + "Copiar texto de las N páginas" (`── Página i ──` separadores, UTF-8 con BOM en el .txt).

---

## 15. PERSISTENCIA

**IndexedDB** `escaner-ios` v2: stores `documents` (keyPath id) y `meta` (claves `initialized`, `manualOrder`). Imágenes como **Blob** (`StoredDocument`). Abrir/cerrar la DB por operación; TODO best-effort con catch silencioso (sin IndexedDB: memoria; nunca lanza).

**`dataUrlToBlob` SIN `fetch()`:** parse manual — base64 (`atob→Uint8Array→Blob`) y **percent-encoded** (SVG de los mocks; `atob` sobre SVG percent-encoded lanza `InvalidCharacterError` y se tragaba el error: renombrar/favoritos en docs demo NO persistían).

**Carga:** `loadAllDocuments()` → `null` = instalación nueva (conserva mocks/datos demo); `[]` = vacía real. Página corrupta → se omite. Migración de filtros al hidratar. Orden: `manualOrder` gana (ids nuevos al final por updatedAt); si no, updatedAt desc. Papelera: purga a los **30 días** al hidratar.

**Ajustes (localStorage `escaner-settings-v1`):** merge defensivo + **migraciones one-time con claves separadas** (una clave por migración, o la nueva nunca correría en instalaciones viejas):
- `escaner-settings-v2-maxq`: `exportQuality "alta" → "máxima"`.
- `escaner-settings-v3-ocr-manual` (v6.1): `ocrEnabled = false` en TODAS las instalaciones.
- Tras su única pasada, el valor persistido vuelve a ser sagrado.

---

## 16. BENCHMARK DE DISPOSITIVO (`device-capability.ts`)

```
score = round(100 · (0.55·cpuScore + 0.20·coreScore + 0.25·memScore))
cpuScore: 12 rondas de Float32Array 256×256 con sqrt+sin (escala logarítmica 20 ms→1.0, 500 ms→0.0)
coreScore: navigator.hardwareConcurrency/8        memScore: (deviceMemory−1.5)/6.5   (iOS no expone deviceMemory → 0.6)
canvas test: 1024×1024, 4 gradientes + toBlob JPEG 0.9 (proxy del warp/encode)
tiers: score ≥66 → high (cap 4032) · ≥33 → medium (3200) · si no low (3200)   // low antes 2560 — PROHIBIDO bajar (v6.1)
```
Cache **7 días** en localStorage (`escaner-device-cap-v1`); una vez por sesión con `requestIdleCallback`; mientras no hay medida, 4032. El cap se aplica en caliente al worker (`config {maxWarpLongSide}`) y lo usa el editor en cada preview. Textos de Ajustes: high "Tu dispositivo procesa a resolución completa (4032 px)." · medium "Tu dispositivo procesa bien a 3200 px — calidad impecable y más fluida." · low "Dispositivo ajustado: 3200 px, más lento pero el texto siempre es legible."

---

## 17. EXPORTAR

**PDF (`buildDocPdf`):** jsPDF `{compress:true}`; página por página con tamaño ADAPTATIVO (lado mayor **297 mm**, orientación según aspecto). Presupuesto de bytes: `min(max(N,3), 8) MB` (~1 MB/página). **Escalera de re-encode** (primer intento que cabe gana):
```
máxima:   [{0, 0.95} → {0, 0.9} → {2600, 0.82} → {2200, 0.78}]
alta:     [{0, 0.9}  → {2600, 0.82} → {2200, 0.78}]
standard: [{2600, 0.82} → {2200, 0.78}]
```
PNG de filtros del producto se embebe SIN re-encode. JPEG sobre canvas con **fondo blanco** y `toBlob` async (C8). Metadatos XMP (título, autor "Escáner", keywords=tags). "Exportar todo" de la biblioteca: presupuesto 30 MB, portada A4 con índice + miniaturas + outline.

**TXT:** UTF-8 **con BOM** (`\uFEFF` — para Excel/Notepad), separadores `── Página i ──` (i = posición real 1-based), share nativo con truncado a 4000 chars.

**Imagen:** descarga la página procesada vigente (cache), nombre `{título}-página-{n}.{png|jpg}`; si aún procesa → toast "Aún se está procesando la página…".

---

## 18. DISEÑO VISUAL (resumen — spec completa en `design-specs.md` del repo original)

- Paleta iOS OBLIGATORIA: `#007AFF` acento · `#F2F2F7` fondo claro · `#FF3B30` destructivo · `#34C759` éxito · `#8E8E93` gris secundario · `#1C1C1E/#2C2C2E/#3A3A3C` dark cards · blanco.
- Tipografía sistema (-apple-system/SF Pro): títulos 17px/600, labels toolbar 11px, cuerpo 13–14px. Cards radius 12px.
- **Phone frame** centrado en desktop (max-w-420px, h-100svh); móvil real a pantalla completa. `viewport-fit=cover` + `100dvh` (sin huecos con barras del navegador).
- Cámara: fondo negro, top bar (X, pill "IA AUTO" con punto azul pulsante, auto-captura, ⋮), visor 65–70%, overlay del quad (SVG, trazo #007AFF 2px no-scaling, puntos blancos en vértices, oscurecido exterior con evenodd), shutter central 72×72 elevado −20px, meter de calidad (verde >80, azul >55, gris), toasts pill oscuros con blur.
- Editor: dark, toolbar 7 columnas de icono+label, sheets vaul con handle, transiciones framer-motion sutiles (0.22–0.3 s).

---

## 19. DISCIPLINA DE MEMORIA Y RENDIMIENTO

1. **Decodes:** la ruta crítica decodifica 1 vez (era 3). `ensureDecoded` solo decodifica strings.
2. **Encodes:** SIEMPRE `toBlob` async (error #22; C13: el sync bloqueaba ~1 s con imports de 48 MP).
3. **R-14:** liberar canvas perdedores YA (`width=height=0`), no esperar GC.
4. **Bitmaps transferidos** se cierran en el worker (`finally`); bitmaps de decoders auxiliares se cierran en `finally` del llamador.
5. **Backpressure por descarte** (§6); 1 mensaje en vuelo al worker.
6. **Caches con tope:** previews LRU 12 (jetsam iOS); historiales del frame loop cap 24 / ventana 2 s.
7. **Canvas iOS ~16,7 MP** máx: por eso 4032×3024 ≈ 12,2 MP es el tope de procesado.
8. OffscreenCanvas reutilizados en el worker (entrada/warp/enhance) en vez de crear por frame.
9. Warm-up del worker al montar la app (`waitReady` en paralelo con el arranque de la cámara).

---

## 20. TABLA DE GOTCHAS (iOS/Android) — causa → solución

| # | Síntoma | Causa raíz | Solución obligatoria |
|---|---|---|---|
| 1 | Cámara no abre en algunos Android | Over-constraining (height/frameRate) | SOLO `width:{ideal:3840}` (E3) |
| 2 | Frontal en vez de trasera en iPhone | `facingMode` no confiable | Re-selección por label+deviceId exact (E4) |
| 3 | No cambia lente ni flash (v3) | Sondas con stream vivo → NotReadableError | Sondas SECUENCIALES cerrando cada stream (F-LENS) |
| 4 | Toast "flash encendido" con LED apagado | Confiar en capabilities | Aplicar + verificar `getSettings().torch` + deshacer fantasma (F-FLASH) |
| 5 | iOS nunca usa takePhoto | Safari no implementa ImageCapture | Detectar y usar cámara nativa `capture="environment"` (HQ-iOS) |
| 6 | Crash/muerte por memoria en Safari | `toDataURL` sync sobre canvas grande | `toBlob` async siempre + liberar canvas (R-14) |
| 7 | Página ilegible en gama de entrada | Gate de nitidez prefería frame 720p ante foto "blanda" | La foto del sensor SIEMPRE gana + timeout 8 s (v6.1) |
| 8 | Captura lenta en gama de entrada | Detección bloqueaba la revisión | Detección en background tras abrir editor (v6.2) |
| 9 | Marco parpadea ~5 Hz | Emitir null durante backpressure | Conservar último quad; descartar frames |
| 10 | Auto-captura infinita con doc estático | Sin re-arm | `rearmNeeded` (score ≤0.8 o pérdida de detección) |
| 11 | Zoom rueda no funciona | `onWheel` de React es pasivo | Listener nativo `{passive:false}` (B10) |
| 12 | Gestos del stage muertos | Guard `target===currentTarget` nunca verdadero | Capturar punteros salvo en `button` (B1) |
| 13 | OCR pega texto en página equivocada | Preview stale durante reproceso | `setPreview(null)` antes de procesar + OCR exige `!previewLoading` |
| 14 | Página demo no persiste cambios | `atob` sobre SVG percent-encoded lanza | Parse manual de data URLs (base64 y percent) |
| 15 | Imagen ESPEJADA al rotar (fallback) | `ctx.scale` no uniforme en 90/270 | Scale uniforme en cropQuad (E6) |
| 16 | "Píxeles feos" en el papel | Mapa de ganancias nearest + ganancia sin tope | Bilineal + TEXT_GAIN_MAX 4 + APPLY_FACTOR_MAX 8 |
| 17 | Documento a distancia normal no se detecta | Área mínima 25% | Bajar a 10% + dilate 3×3×2 (F5-CROP) |
| 18 | HEIC no abre en Android/Chrome | Sin decodificador nativo | Cascada nativa→heic2any dinámico→re-decode (F-HEIC) |
| 19 | 404 antes de cada OCR en Pages | Fetch absoluto `/api/ocr` | Gate `NEXT_PUBLIC_STATIC` (A2) |
| 20 | Pérdida de cámara a mitad | Otra app roba el track | `onended` → toast + re-arranque completo (B3) |
| 21 | Disparo perdido en silencio | Encode/frames fallan sin feedback | Toasts "No se pudo capturar · Inténtalo de nuevo" (B8) |
| 22 | Avisos "no detecto" infinitos | Reloj nunca se re-armaba | Máx 2 por sesión; reloj desde la ÚLTIMA detección (B9) |
| 23 | Doble tap en Guardar cae en "No hay páginas" | saving se liberaba antes del await | Botón deshabilitado DURANTE la descarga + "seguro de vida" en finally |
| 24 | Cancelar recorte ≡ Aplicar | El drag ya escribió el store | Snapshot al entrar en crop; Cancelar restaura el snapshot |

---

## 21. HISTORIAL DE VERSIONES (por qué las reglas existen)

- **v1–v4:** base (3 vistas, filtros, detección worker, PDF, lote, biblioteca). v3/v4 fijaron cámara v2 (sondas secuenciales, linterna verificada).
- **v5:** calidad, zoom, split OCR, benchmark de dispositivo, papelera. Dejó el gate de nitidez en captura (fallo que v6.1 mató).
- **v6 (post-auditoría de 20+ ítems):** HEIC completo, fixes de cámara (B2 permiso, B3 reconexión, B8, B9), quick wins (F1 labels, F2 flash a toolbar), SW nuevo, limpieza del repo.
- **v6.1:** F-RES-PRIORITY (la foto del sensor siempre gana; timeout 8 s) + **OCR bajo demanda por defecto** (migración one-time) + cap low 2560→3200. Motivado por prueba real en un Tecno de entrada: "la imagen se veía horrible y no se podía reconocer el texto".
- **v6.2:** F-DEFER-CROP — detección de bordes en background; editor instantáneo; pill "Ajustando recorte…"; recorte manual protegido. Motivado por: "que la captura sea fluida" sin perder el auto-recorte.

---

## 22. PLAN DE CONSTRUCCIÓN RECOMENDADO (para la IA que replica)

Fase 1 — Esqueleto: Next.js estático + Zustand + types.ts completos + phone frame + navegación de 4 vistas + toasts/sheets. 
Fase 2 — Cámara básica: getUserMedia E3/E4 + visor + shutter + captura a store (sin worker aún: quad = defaultQuad). 
Fase 3 — Motor de visión: worker + OpenCV self-hosted + protocolo + cascada de detección + scoring + validaciones; fallback Sobel/canvas DLT. Verificar con la página demo (factura generada in-memory 1920×2580). 
Fase 4 — Pipeline completo: warp (refine+shrink) + filtros (matemática §10) + thumbnails + cache de previews + capturePageKey. 
Fase 5 — Editor completo: gestos §13, crop con lupa, filtros sheet, rotación instantánea, multi-página, guardar. 
Fase 6 — Persistencia + biblioteca + papelera + migraciones. 
Fase 7 — OCR dual + HEIC + exports PDF/TXT. 
Fase 8 — v6.1 + v6.2 (§7–8) + benchmark de dispositivo + toques finales (pills, badges, overlays). 
Fase 9 — Endurecimiento: recorrer la tabla §20 una por una y verificar cada gotcha con pruebas reales (mínimo: Chrome Android + iPhone Safari).

**Orden de prioridad si hay presupuesto limitado:** motor de visión fiel > reglas de captura (§7–8) > filtros (§10) > gestos del editor > todo lo demás.

---

## 23. CRITERIOS DE ACEPTACIÓN (la réplica es fiel cuando…)

1. Capturar en un gama de entrada: el editor abre en <1 s, la página guarda la resolución del sensor y el texto se lee (probar una foto ligeramente blanda: NO debe sustituirse por un frame 720p).
2. El recorte automático aterriza en background (pill visible → desaparece → preview se actualiza solo) y un recorte manual posterior se respeta AL PÍXEL tras re-filtrar/rotar.
3. Con el documento quieto 4/6 muestras >0.8 en 1200 ms → auto-captura dispara UNA vez (cooldown 1,5 s; re-arm exige re-encuadre).
4. La linterna funciona en Android aunque `capabilities.torch` no la anuncie; el toast de fallo lista las causas exactas.
5. Un HEIC de iPhone abre solo (convertido) y un «.jpg» con contenido HEIC también.
6. Los 3 filtros producen: text = papel blanco + tinta marcada sin bloques feos; bw = B/N inmune a sombras (Bradley-Roth) sin motas <3 px; original = foto pura sin realce.
7. Rotar una página procesada tarda ~0,2 s (sin reproceso del worker).
8. Matar el worker (o bloquearlo) NO rompe la app: fallback canvas con la misma matemática.
9. Sin IndexedDB (modo privado) todo funciona en memoria sin errores.
10. El OCR está apagado por defecto; el botón «Texto» extrae con Tesseract spa+eng y permite copiar; una instalación vieja con OCR activado se migra a OFF una sola vez.
11. El PDF sale ≤8 MB con calidad "máxima" y las páginas conservan nitidez (PNG de filtros embebido sin re-encode).
12. Ninguna prueba de la tabla §20 reproduce su síntoma.

---

*Documento generado a partir del código fuente real (v6.2, commit 08a8d97) del proyecto. Extraído con revisión línea a línea de: image-processor.ts, detection-worker.js, quality.ts, frame-loop.ts, CameraView.tsx, EditorView.tsx, store.ts, page-store.ts, ocr.ts, device-capability.ts, pdf-export.ts, text-export.ts, types.ts, detector-client.ts, design-specs.md.*
