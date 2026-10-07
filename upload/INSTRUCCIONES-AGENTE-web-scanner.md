# 📋 Instrucciones para el agente — web-scanner (post-auditoría)

> ## ✅ ACTUALIZACIÓN — ESTE TRABAJO YA ESTÁ HECHO
> Las modificaciones de este documento **ya fueron implementadas y verificadas**
> (rama `fix/auditoria`, 5 commits sobre `cec4642`): HEIC + Fase 0 + Cámara (B2/B3)
> + quick wins Fase 1 + Fase 2 seguros. Este archivo queda como REFERENCIA
> (contexto, hallazgos pendientes de Fase 2/3 y reglas del proyecto).

**Repositorio:** https://github.com/Jg-Stevan/web-scanner (rama `main`)
**Commit base:** `cec4642` — ya incluye la v5 (perspectiva mejorada, PWA instalable, benchmark de dispositivo, rotación instantánea, guardar-como-imagen, importación robusta, filtro B/N adaptativo, sin perfiles de documento, sin página demo, dimensiones móviles).
**Pendiente:** (a) soporte HEIC (parche listo en el Apéndice, verificado E2E), (b) implementar los 56 hallazgos de la auditoría `escaner-auditoria-estado.md`, (c) arreglar el CI/CD que está roto, (d) verificar y publicar.

> Los números de línea de la auditoría valen para `cec4642` y se desplazan conforme apliques cambios. La auditoría ya tiene "Fix sugerido" por hallazgo; este documento da el ORDEN, el contexto y los detalles que la auditoría no conoce.

---

## 1) Reglas no negociables

1. **NO romper el mecanismo v2 de cámara** (lo más frágil del proyecto, ya verificado en teléfono real): sondas de cámara SECUENCIALES que cierran cada `MediaStream` antes de abrir la siguiente (`probeCamera` / `chooseMainProbe` / `openMainCamera` en `CameraView.tsx`), linterna con reintento `applySavedTorchWithRetry` (×5) y verificación con `getSettings().torch` (×2). Sondear con la cámara ya abierta provoca `NotReadableError` en Android — fue el bug histórico. Si un hallazgo te pide tocar la apertura de cámara, revisa el diff dos veces.
2. **App 100% cliente para GitHub Pages**: `BUILD_STATIC=1 bunx next build` con `output: "export"` y `basePath: /web-scanner`. Nada de server actions. `src/app/api` se excluye del build por el workflow (y en la Fase 0 se limpia del repo).
3. **Toda la UI en español**, estética iOS pixel-perfect (colores sistema: `#007AFF`, `#34C759`, `#FF3B30`, fondos `#f2f2f7`/negro).
4. **NO borrar `public/vendor/opencv-*.js`**: el `detection-worker.js` los carga por ruta relativa y SÍ funcionan en Pages (el `LEEME-SUBIR-A-GITHUB.md` dice lo contrario y está mal — hallazgo A3).
5. **Gestor de paquetes: `bun`** (el workflow usa `bun install --frozen-lockfile`). Si tocas `package.json`, ejecuta `bun install` para regenerar `bun.lock` y sube ambos.
6. **Sin tests automatizados**: se verifica con `bun run lint`, `bunx tsc --noEmit`, build estático y prueba manual en navegador.
7. Nunca commitear `out/`, `.next/`, `dev.log` ni `node_modules/`.

---

## 2) Preparación del entorno

```bash
git clone https://github.com/Jg-Stevan/web-scanner.git
cd web-scanner
bun install
bun run lint                      # debe salir limpio
BUILD_STATIC=1 bunx next build    # prueba de build estático (salida ./out)
rm -rf out .next                  # limpiar tras la prueba
```

> ⚠️ Para probar el build estático usa SIEMPRE `BUILD_STATIC=1 bunx next build`. El script `bun run build` es otro (standalone con servidor) y no sirve para Pages.

---

## 3) Paso 1 — Soporte HEIC (parche listo, aplicar primero)

Las fotos HEIC/HEIF de iPhone fallaban con «No se pudo procesar la imagen» en Android/Chrome. El **Apéndice** de este documento trae un parche verificado E2E (probado con un HEIC real en Chromium, que no decodifica HEIC) sobre `cec4642`:

```bash
# guarda el bloque del Apéndice como heic.patch (solo el diff, sin las líneas ```)
git apply heic.patch && rm heic.patch
bun install                       # instala heic2any y actualiza node_modules
bun run lint
```

Qué incluye el parche:
- `package.json` + `bun.lock`: dependencia `heic2any@^0.0.4` (libheif a JS puro, sin .wasm externo → compatible con Pages).
- `src/lib/scanner/image-processor.ts`: cascada nativa extraída a `nativeDecodeToCappedDataUrl()` + rescate **F-HEIC** con `import("heic2any")` DINÁMICO (el chunk de ~1,4 MB solo se descarga cuando una imagen lo necesita) + re-decodificación del JPEG convertido. Cubre HEIC/HEIF y los «.jpg» con contenido HEIC mal etiquetados.
- `src/components/scanner/CameraView.tsx`: `onFilePicked` con `toast.loading("Convirtiendo HEIC…")` y errores diferenciados.
- `README.md`: nota de novedades.

Prueba manual: «Importar» un `.heic` → debe verse el toast «Convirtiendo HEIC…» y la página quedar en el editor.

---

## 4) Paso 2 — CI/CD

> ⚠️ CORRECCIÓN: el trigger del workflow (`branches: [main, master]`) **siempre
> estuvo bien** — un artefacto de terminal hizo parecer roto el YAML. Verificado
> a nivel de bytes. Los pushes SÍ disparan el deploy.

Real (ya aplicado en la rama `fix/auditoria`):
- **A4**: `actions/upload-pages-artifact@v3` → `@v4` (v3 depende de upload-artifact v3 deprecado).
- **A8 (parcial)**: step `bunx tsc --noEmit` tras `Install dependencies` (pasa limpio: los únicos errores de tipos estaban en `DocumentDetailView.tsx`, eliminado).
- Env `NEXT_PUBLIC_STATIC: "1"` añadido (lo consume el hallazgo A2).

---

## 5) Paso 3 — Fase 0 · Pre-publicación (orden interno sugerido)

| ID | Qué hacer (detalle verificado en el código) |
|----|---------------------------------------------|
| **A1** | `SettingsView.tsx` línea ~62: `PROJECT_ZIP_STATIC = \`/downloads/...\`` → prefijar con basePath: `` `const BASE = process.env.NEXT_PUBLIC_BASE_PATH ?? "";` `` y `` `${BASE}/downloads/${PROJECT_ZIP_NAME}` ``. Revisar también el otro uso del zip (~línea 599). Si decides eliminar el zip (A5), elimina el botón y ambos usos. |
| **A2 + D1 + D4** | Decisión coherente recomendada: **borrar `src/app/api` COMPLETO** (el deploy es Pages puro y el OCR ya cae a Tesseract local). En `ocr.ts` (~línea 131): si `process.env.NEXT_PUBLIC_STATIC === "1"` → saltar directo a Tesseract sin `fetch`; si no, prefijar `${process.env.NEXT_PUBLIC_BASE_PATH ?? ""}/api/ocr`. |
| **A3 + F1** | Borrar del repo: `LEEME-SUBIR-A-GITHUB.md`, `parche-fusion.patch`, `worklog.md`, `src/components/scanner/DocumentDetailView.tsx` (2.002 líneas sin importar), `scripts/`. **NO borrar `public/vendor/`.** |
| **A5** | Con lo anterior el repo baja ~11 MB. El zip `public/downloads/web-scanner-repo.zip` es opcional: o se elimina (y con él el botón de A1), o se queda y se corrige su ruta (A1). Elegir UNA opción y ejecutarla completa. |
| **B2** | `CameraView.tsx`: detectar `NotAllowedError` en la cascada `getUserMedia` → pantalla/toast de error con CTA «Activar cámara» + instrucciones de ajustes del navegador. La demo/simulado SOLO como modo explícito, nunca fallback silencioso de un permiso denegado. |
| **B3** | Suscribir `track.onended` (y `track.onmute`) del stream activo → detener frame loop, liberar `streamRef`, volver a `idle` y avisar con toast accionable. |
| **F3** | README: 3 filtros reales (`original`, `text` «Texto claro», `bw` «B/N adaptativo»), sección «Privacidad: todo se procesa en tu dispositivo», estructura real de carpetas, quitar menciones a rutas API eliminadas. |

---

## 6) Paso 4 — Fase 1 · Estabilidad (orden interno sugerido)

1. **B1 (crítico)** `EditorView.tsx` ~655: el guard `if (e.target === e.currentTarget)` nunca deja activar el compare — cambiar a: activar salvo que `e.target.closest("button")` exista. Arreglar en la misma zona el `setPointerCapture` (~619-625) para que el swipe no se corte.
2. **C2 (alto)** `EditorView.tsx` ~1896: miniaturas del carrusel → `src={p.thumbnail}` (hoy decodifican fotos de 12 MP para pintar 48×64 px).
3. **C3 (alto)** `EditorView.tsx` ~1570 + `PresentationView.tsx` ~606: pasar la miniatura real de 160 px a la presentación (hoy usa originales como thumbs).
4. **B4** `CameraView.tsx` ~791-803 + `store.ts` ~421-424: snapshot de ids a fusionar ANTES del `await`; al terminar limpiar solo esas páginas (no vaciar todo `capturePages`).
5. **B5** `EditorView.tsx` ~1342/1380: un `ocrInFlightRef` compartido entre el OCR manual, el automático y el por-lote; revalidar la clave de la página antes de cada `updateCapturePage` (evita persistir texto stale con `ocrDone:true`).
6. **B6** `ocr.ts` ~114-125 + `LibraryView.tsx` ~585-636: worker Tesseract PERSISTENTE por lote (createWorker 1 vez, terminate al final) + botón «Cancelar» que corte el bucle.
7. **B7** `EditorView.tsx` ~1569: en `presentationPages`, no usar `p.original` de fallback: procesar on-demand o mostrar placeholder «sin procesar».
8. **B8** `CameraView.tsx` ~629-631/644-646/710-713: chequear `processingRef` ANTES del cooldown (para no descartar toques en silencio) y `toast.error` si el encode devuelve null.
9. **B9** `frame-loop.ts` ~114/283-290: re-armar `firstAttemptMs` tras la PRIMERA detección exitosa; máximo 1-2 avisos «No detecto el documento» por sesión.
10. **B10** `EditorView.tsx` ~781: replicar el listener nativo `wheel` con `{passive:false}` (como `PresentationView.tsx` ~392) — el `onWheel` de React es pasivo y `preventDefault` es no-op.
11. **B11** `LibraryView.tsx` ~200-223/1686/1922 + `PresentationView.tsx` ~213-219: ignorar el `pointerdown` del long-press si `e.target.closest("button")`.
12. **B15** `pwa.ts` ~105-109: si `document.readyState === "complete"` al montar el efecto → registrar el SW inmediatamente (no depender del evento `load`, que ya pasó).
13. **C4** `page.tsx` ~72: diferir `warmUpScannerWorker()` hasta la primera entrada a cámara/editor (o `requestIdleCallback` + respetar `navigator.connection.saveData`). El OpenCV de 8,2 MB no debe bajar al abrir la biblioteca.
14. **C5** `LibraryView.tsx` ~345-391: debounce ~200 ms en la búsqueda + `Map` caché del texto normalizado por página (la normalización NFD ×2 por tecla tartamudea).

---

## 7) Paso 5 — Fase 2 · Gama media-baja (orden interno sugerido)

1. **C1 (la mejora estrella — hacerla al final de la fase, es un refactor grande)** `page-store.ts` ~218-274 + `store.ts` ~710: hidratar SOLO metadatos + thumbnails; `original`/`processed` lazy por documento (object URLs) con LRU global. Hoy 20 docs × 4 págs ≈ 400-560 MB de strings data URL → jetsam en iOS. E8 (skeletons) va de la mano.
2. **A6** `public/sw.js`: tope de entradas en `RUNTIME_CACHE` (~80, purgar FIFO) + excluir `/downloads/` del cacheo.
3. **A7** `public/sw.js` ~82: solo `cache.put` si `fresh.ok`.
4. **C6** `EditorView.tsx` ~553/673: chip de % de zoom aislado con `useMotionValueEvent` (o throttle ~10 Hz) — hoy cada `pointermove` re-renderiza el árbol completo.
5. **C7** `EditorView.tsx` ~1078/189-224: usar `cachePreviewEntry` (LRU) en vez de `previewCache.set` directo; capar la rotación a 2048 px o moverla al worker.
6. **C8** `pdf-export.ts` ~76/106: migrar `toDataURL` síncrono → `toBlob` + FileReader.
7. **C9** `pdf-export.ts` ~456-499: un solo pase con degradación al vuelo por bytes estimados (hoy hasta 4 reconstrucciones con todo en memoria).
8. **C10** `CameraView.tsx` ~606-691: burst A/B a ≤1080p (el gate mide a 400 px igualmente) o pool de canvases.
9. **C11** `store.ts` ~508-514 y similares: acciones masivas únicas (`toggleFavoriteMany`, eliminación por lote) con un solo `set()` + una persistencia.
10. **C12** `LibraryView.tsx` ~1155/1223: quitar el `key={`grid-${sort}`}` (remonta todo) y virtualizar la vista de lista.
11. **C13** `CameraView.tsx` ~238: `downscaleImage` → patrón `toBlob` (como ~582-601).
12. **C14** `page-store.ts`: conexión IndexedDB singleton; en toggles persistir solo metadatos; hidratación paralela acotada (`Promise.all` con límite).
13. **E1** `BottomNav.tsx` ~52/101 + `OnboardingView.tsx` ~106/164/215: gris `#8e8e93` → `#6d6d72` / `#58585b` en modo claro (sobre oscuro vale el actual).
14. **E2** `layout.tsx` ~56-57: decidir zoom de página (permitir, o documentar la decisión «app-like» — WCAG 1.4.4).
15. **E3** `TagsDialog.tsx` ~79 y demás: padding invisible hasta 24-44 px en controles pequeños.
16. **E4** `LibraryView.tsx` ~1652-1690/1898-1926/1998-2010 + `EditorView.tsx` ~1991-2036: reestructurar roles (no `role="button"` con botones anidados); asa de reordenar y handles de recorte con teclado (flechas).
17. **B14** `LibraryView.tsx` ~1125-1135/2315: envolver el listado en `AnimatePresence` o quitar el `exit` muerto.

---

## 8) Paso 6 — Fase 3 · Pulido oficial (opcional, tras publicar)

- **A8**: documentar por qué `reactStrictMode: false` (doble-montaje rompe el frame loop de cámara) — no activarlo sin probar.
- **B12** `TagsDialog.tsx` ~198-208: aplicar tags a los que caben + toast «N documentos omitidos».
- **B13** `LibraryView.tsx` ~350/431-440: prestar `pendingFindQuery` si hay match en OCR aunque también matchee el título.
- **B16** `mock-data.ts` ~9: banner «Documentos de ejemplo · Empezar de cero» + fecha relativa real (BASE_TIME es 2026 → «hace un momento» eterno).
- **D2**: sondas `window.__*` condicionadas a `process.env.NODE_ENV !== "production"` (CUIDADO: `__cameraChoice` es el diagnóstico de cámara — mantener en dev).
- **D3**: SRI en CDNs + meta CSP (`script-src 'self' cdn.jsdelivr.net docs.opencv.org`).
- **E5** foco/trampa en presentación + `aria-live` en pills + `useReducedMotion()`. **E6** SaveSuccessOverlay: tap para cerrar + barra de progreso siempre.
- **E7/E8/F5/F6**: ideas (2 columnas desktop, skeletons, WebP + manifest rico, Lighthouse en CI).
- **F2** (poda de ~14 paquetes): hacerlo AL FINAL y con build de prueba tras cada tanda — `@dnd-kit×3`, `next-auth`, `next-intl`, `@tanstack/react-table`, `@mdxeditor`, `react-syntax-highlighter`, `uuid`, `zod`, `date-fns`, stack `prisma`… y podar `src/components/ui` no importados. Regenerar `bun.lock`.
- **F4**: crear `DEPLOY.md` (workflow, basePath, verificación post-deploy) sustituyendo al LEEME borrado.

---

## 9) Verificación local (obligatoria antes de subir)

```bash
bun run lint                # 0 errores
bunx tsc --noEmit           # 0 errores de tipos
BUILD_STATIC=1 bunx next build
# revisar ./out: index.html existe, assets bajo rutas /web-scanner/, no hay src/ absoluto
rm -rf out .next
```

Prueba manual en `bun run dev` (o abriendo `out/` en un servidor estático):
- Cámara: permiso concedido → preview + detección; **linterna** (si hay torch); permiso denegado → aviso claro (B2).
- Importar JPG, PNG y **HEIC** → los tres entran al editor.
- Editor: rotar (rápido, sin reprocesar), filtros, recortar, guardar imagen, guardar PDF.
- OCR de una página (con red y sin red — debe caer a Tesseract local).
- PWA: instalar desde el navegador, abrir sin conexión.
- Biblioteca: búsqueda, etiquetas, favoritos, papelera.

---

## 10) Subir y actualizar el deploy

```bash
git checkout -b fix/auditoria
git add -A
git commit -m "HEIC + auditoría: fase 0-2, CI/CD arreglado"
git push -u origin fix/auditoria
# → abrir Pull Request → merge a main (o push directo si no hay protección)
```

Después:
1. Pestaña **Actions**: el workflow debe dispararse SOLO con el push (si no ocurre, el trigger de la línea 8 sigue mal — revisar el Paso 2). Si el job falla por `upload-pages-artifact@v3`, subir a `@v4` y re-correr.
2. Abrir `https://jg-stevan.github.io/web-scanner/` en incógnito y en el móvil.
3. El Service Worker puede servir versión vieja: recargar 2 veces; si persiste, borrar datos del sitio (Configuración → ver sitio → Borrar datos) y recargar.
4. Publicar de forma incremental: **Fase 0 → publicar y probar en el teléfono → Fase 1 → publicar → Fase 2…** (no acumular todo en un solo megapush sin probar).

---

## 11) Checklist final

- [ ] Deploy verde disparado por push (trigger arreglado)
- [ ] HEIC importable en Chrome/Android (toast de conversión + página en editor)
- [ ] Regresión de cámara: captura + linterna OK en teléfono real
- [ ] OCR funciona sin conexión (Tesseract local, sin 404 previo)
- [ ] PWA instala y funciona offline
- [ ] Sin `src/app/api`, sin `DocumentDetailView`, sin parche/worklog/LEEME
- [ ] README con 3 filtros + sección privacidad
- [ ] `bun run lint` y `bunx tsc --noEmit` limpios

---

## Apéndice — heic.patch (verbatim sobre `cec4642`)

> Guarda TODO el bloque diff siguiente (sin las líneas de fence) como `heic.patch` y aplica con `git apply heic.patch`.

```diff
diff --git a/README.md b/README.md
index e294b7b..f77466d 100644
--- a/README.md
+++ b/README.md
@@ -2,12 +2,27 @@
 
 Aplicación web de digitalización de documentos con estética **pixel-perfect de iOS**, construida como reproducción fiel de 4 diseños originales (Cámara, Editor de perspectiva, Digitalización y Biblioteca). Toda la interfaz está en **español**.
 
-![Versión](https://img.shields.io/badge/versión-3.0.0-007AFF)
+![Versión](https://img.shields.io/badge/versión-5.0.0-007AFF)
 ![Framework](https://img.shields.io/badge/Next.js-16-black)
 ![Licencia](https://img.shields.io/badge/licencia-MIT-34C759)
 
 ---
 
+## 🆕 Novedades v5.0.0
+
+- **PWA instalable**: «Instalar app» desde Ajustes (Android/Chrome: prompt nativo · iOS: guía de «Añadir a pantalla de inicio»), Service Worker network-first (sin caché vieja al actualizar) y funcionar **sin conexión**
+- **Benchmark del dispositivo**: Ajustes › Rendimiento del dispositivo mide CPU/canvas y ajusta la resolución de procesado (4032/3200/2560 px) para que ningún teléfono se cuelgue
+- **Perspectiva mejorada**: cascada de detección multi-umbral (papeles con poco contraste y bordes rotos ya se detectan) y warp a resolución completa del sensor (4032)
+- **Rotación instantánea** (~0,2 s): rotar ya NO reprocesa el documento completo
+- **Guardar como imagen**: botón «Imagen» en el editor descarga la página actual en PNG
+- **Importación robusta**: las fotos de galería/cámara nativa (12–48 MP, EXIF, HEIC) ya no fallan con «No se pudo procesar la imagen»
+- **Soporte HEIC completo**: las fotos HEIC/HEIF de iPhone se convierten solas en el navegador (libheif embebido, descarga bajo demanda) — también en Android/Chrome y en los «.jpg» que en realidad traen contenido HEIC
+- **Filtro por defecto «B/N adaptativo»** en cada captura nueva
+- **Ajustes simplificados**: sin perfiles de documento (detección siempre automática) y sin la página de demo en el editor
+- **Dimensiones móviles**: viewport-fit=cover + dvh (sin huecos con las barras del navegador ni recortes en la PWA instalada)
+
+---
+
 ## ✨ Funcionalidades
 
 ### 📷 Captura (Pantalla 1)
diff --git a/bun.lock b/bun.lock
index 79dec44..648b5e8 100644
--- a/bun.lock
+++ b/bun.lock
@@ -47,6 +47,7 @@
         "date-fns": "^4.1.0",
         "embla-carousel-react": "^8.6.0",
         "framer-motion": "^12.23.2",
+        "heic2any": "^0.0.4",
         "input-otp": "^1.4.2",
         "jspdf": "^4.2.1",
         "lucide-react": "^0.525.0",
@@ -1181,6 +1182,8 @@
 
     "hastscript": ["hastscript@6.0.0", "https://registry.npmjs.com/hastscript/-/hastscript-6.0.0.tgz", { "dependencies": { "@types/hast": "^2.0.0", "comma-separated-tokens": "^1.0.0", "hast-util-parse-selector": "^2.0.0", "property-information": "^5.0.0", "space-separated-tokens": "^1.0.0" } }, "sha512-nDM6bvd7lIqDUiYEiu5Sl/+6ReP0BMk/2f4U/Rooccxkj0P5nm+acM5PrGJ/t5I8qPGiqZSE6hVAwZEdZIvP4w=="],
 
+    "heic2any": ["heic2any@0.0.4", "", {}, "sha512-3lLnZiDELfabVH87htnRolZ2iehX9zwpRyGNz22GKXIu0fznlblf0/ftppXKNqS26dqFSeqfIBhAmAj/uSp0cA=="],
+
     "hermes-estree": ["hermes-estree@0.25.1", "https://registry.npmjs.com/hermes-estree/-/hermes-estree-0.25.1.tgz", {}, "sha512-0wUoCcLp+5Ev5pDW2OriHC2MJCbwLwuRx+gAqMTOkGKJJiBCLjtrvy4PWUGn6MIVefecRpzoOZ/UV6iGdOr+Cw=="],
 
     "hermes-parser": ["hermes-parser@0.25.1", "https://registry.npmjs.com/hermes-parser/-/hermes-parser-0.25.1.tgz", { "dependencies": { "hermes-estree": "0.25.1" } }, "sha512-6pEjquH3rqaI6cYAXYPcz9MS4rY6R4ngRgrgfDshRptUZIc3lw0MCIJIGDj9++mfySOuPTHB4nrSW99BCvOPIA=="],
diff --git a/package.json b/package.json
index cd631d0..ec64d06 100644
--- a/package.json
+++ b/package.json
@@ -55,6 +55,7 @@
     "date-fns": "^4.1.0",
     "embla-carousel-react": "^8.6.0",
     "framer-motion": "^12.23.2",
+    "heic2any": "^0.0.4",
     "input-otp": "^1.4.2",
     "jspdf": "^4.2.1",
     "lucide-react": "^0.525.0",
diff --git a/src/components/scanner/CameraView.tsx b/src/components/scanner/CameraView.tsx
index c49fd78..41eddd5 100644
--- a/src/components/scanner/CameraView.tsx
+++ b/src/components/scanner/CameraView.tsx
@@ -753,10 +753,11 @@ export default function CameraView() {
     captureInputRef.current?.click();
   }, [hasStream, status, captureSmart, captureDemo]);
 
-  // F-IMPORT (robusto): el File se decodifica NATIVAMENTE con
-  // createImageBitmap (aplica orientación EXIF y soporta imágenes enormes
-  // sin pasar por data URL de decenas de MB). Fallbacks en cascada dentro
-  // de fileToCaptureDataUrl; el mensaje de error distingue formato.
+  // F-IMPORT (robusto) + F-HEIC: el File se decodifica con la cascada nativa
+  // (createImageBitmap con EXIF → <img>) y, si el navegador no abre el formato
+  // (HEIC en Android/Chrome, o «.jpg» con contenido HEIC), se convierte con
+  // libheif (heic2any) dentro de fileToCaptureDataUrl. La conversión puede
+  // tardar unos segundos en fotos de 12 MP → toast de progreso.
   const onFilePicked = useCallback(
     (e: ChangeEvent<HTMLInputElement>) => {
       const file = e.target.files?.[0];
@@ -767,16 +768,25 @@ export default function CameraView() {
         toast.error("El archivo seleccionado no es una imagen");
         return;
       }
+      const looksHeic = /heic|heif/i.test(file.type) || /\.hei[cf]$/i.test(file.name);
+      // Heurística: los «.jpg» de iPhone transportados por apps pueden traer
+      // contenido HEIC; si la decodificación nativa falla también irán al
+      // rescate, pero no podemos saberlo de antemano → toast solo si es HEIC.
+      const toastId = looksHeic
+        ? toast.loading("Convirtiendo HEIC… (puede tardar unos segundos)")
+        : undefined;
       void (async () => {
         try {
           const dataUrl = await fileToCaptureDataUrl(file);
+          if (toastId !== undefined) toast.success("Imagen lista", { id: toastId, duration: 1500 });
           await handleCaptureDataUrl(dataUrl);
         } catch (err) {
           const msg = err instanceof Error ? err.message : "";
+          if (toastId !== undefined) toast.dismiss(toastId);
           if (/heic|heif/i.test(msg)) {
-            toast.error("Tu navegador no abre este formato (HEIC). Conviértelo a JPG e inténtalo de nuevo.", { duration: 7000 });
+            toast.error("No se pudo convertir el HEIC", { description: "El archivo parece dañado o protegido. Prueba con otro." });
           } else {
-            toast.error("No se pudo procesar la imagen", { description: "Prueba con un JPG o PNG más pequeño." });
+            toast.error("No se pudo procesar la imagen", { description: "El archivo puede estar corrupto o ser un formato no soportado." });
           }
         }
       })();
diff --git a/src/lib/scanner/image-processor.ts b/src/lib/scanner/image-processor.ts
index 4fabece..2f19e2b 100644
--- a/src/lib/scanner/image-processor.ts
+++ b/src/lib/scanner/image-processor.ts
@@ -118,34 +118,60 @@ function sourceToCappedJpegDataUrl(
 }
 
 /**
- * F-IMPORT — File → data URL de forma ROBUSTA. Los archivos de galería/
- * cámara nativa llegan a 12–48 MP y con EXIF; la ruta vieja (FileReader →
- * data URL gigante → <img>) reventaba con fotos grandes y era el origen del
- * «No se pudo procesar la imagen».
- *
- * Cascada (de más barata a más costosa):
- *  1. createImageBitmap(file, { imageOrientation:"from-image" }) — decode
- *     NATIVO, EXIF aplicado por el navegador, sin data URL intermedia.
- *     Si el lado mayor excede el tope → re-dibujo a canvas reducido.
- *  2. createImageBitmap(file) a secas (navegadores que rechazan opciones).
- *  3. objectURL → <img> (el navegador decodifica y aplica EXIF al pintar;
- *     evita el data URL de decenas de MB de la ruta histórica).
- *
- * El Error resultante distingue HEIC para dar un mensaje accionable.
+ * F-HEIC — decodificador HEIC/HEIF en cliente (libheif vía heic2any).
+ * Chrome/Android NO decodifica HEIC nativamente (las fotos de iPhone pasadas
+ * por WhatsApp/Drive/copys fallaban con «No se pudo procesar la imagen»).
+ * El chunk pesa ~1,4 MB así que se importa DINÁMICAMENTE: solo se descarga
+ * la primera vez que una imagen realmente necesita el rescate (un JPG normal
+ * nunca lo toca). heic2any empaqueta libheif inline — sin .wasm externo,
+ * funciona igual en GitHub Pages.
  */
-export async function fileToCaptureDataUrl(file: File): Promise<string> {
-  const looksHeic = /heic|heif/i.test(file.type) || /\.hei[cf]$/i.test(file.name);
+type Heic2Any = (opts: {
+  blob: Blob;
+  toType?: string;
+  quality?: number;
+}) => Promise<Blob | Blob[]>;
+
+let heic2anyFn: Heic2Any | null = null;
+let heic2anyPromise: Promise<Heic2Any> | null = null;
+
+function loadHeic2Any(): Promise<Heic2Any> {
+  if (heic2anyFn) return Promise.resolve(heic2anyFn);
+  if (!heic2anyPromise) {
+    heic2anyPromise = import("heic2any")
+      .then((mod) => {
+        heic2anyFn = mod.default as Heic2Any;
+        return heic2anyFn;
+      })
+      .catch((err) => {
+        heic2anyPromise = null; // permite reintentar (p.ej. red recuperada)
+        throw err;
+      });
+  }
+  return heic2anyPromise;
+}
 
+/** HEIC/HEIF (o .jpg con contenido HEIC) → JPEG decodificable. */
+async function convertHeicToJpegBlob(source: Blob): Promise<Blob> {
+  const convert = await loadHeic2Any();
+  const out = await convert({ blob: source, toType: "image/jpeg", quality: 0.92 });
+  return Array.isArray(out) ? out[0] : out;
+}
+
+/** Intenta decodificar un Blob con la cascada NATIVA del navegador y
+ *  devolver un data URL JPEG topeado. Devuelve null si nada funciona
+ *  (HEIC en Android llega aquí y devuelve null → activa el rescate F-HEIC). */
+async function nativeDecodeToCappedDataUrl(blob: Blob): Promise<string | null> {
   if (typeof createImageBitmap === "function") {
     const attempts: Array<Promise<ImageBitmap>> = [];
     try {
       attempts.push(
-        createImageBitmap(file, { imageOrientation: "from-image" } as ImageBitmapOptions)
+        createImageBitmap(blob, { imageOrientation: "from-image" } as ImageBitmapOptions)
       );
     } catch {
       /* opciones no soportadas */
     }
-    attempts.push(createImageBitmap(file));
+    attempts.push(createImageBitmap(blob));
     for (const attempt of attempts) {
       let bm: ImageBitmap | null = null;
       try {
@@ -165,25 +191,62 @@ export async function fileToCaptureDataUrl(file: File): Promise<string> {
     }
   }
 
-  // 3) objectURL → <img> → canvas (re-encode JPEG normalizado).
-  const objectUrl = URL.createObjectURL(file);
+  // objectURL → <img> → canvas (el navegador decodifica y aplica EXIF al pintar).
+  const objectUrl = URL.createObjectURL(blob);
   try {
     const img = await loadImage(objectUrl);
     const w = img.naturalWidth || img.width;
     const h = img.naturalHeight || img.height;
-    if (!w || !h) throw new Error("imagen sin dimensiones");
+    if (!w || !h) return null;
     return await sourceToCappedJpegDataUrl(img, w, h);
-  } catch (err) {
-    throw new Error(
-      looksHeic
-        ? "HEIC no soportado por este navegador"
-        : `No se pudo decodificar la imagen${err instanceof Error ? `: ${err.message}` : ""}`
-    );
+  } catch {
+    return null;
   } finally {
     URL.revokeObjectURL(objectUrl);
   }
 }
 
+/**
+ * F-IMPORT — File → data URL de forma ROBUSTA. Los archivos de galería/
+ * cámara nativa llegan a 12–48 MP y con EXIF; la ruta vieja (FileReader →
+ * data URL gigante → <img>) reventaba con fotos grandes y era el origen del
+ * «No se pudo procesar la imagen».
+ *
+ * Cascada (de más barata a más costosa):
+ *  1. Decodificación NATIVA (createImageBitmap con EXIF → bitmap a secas →
+ *     <img> vía objectURL) — ver nativeDecodeToCappedDataUrl.
+ *  2. F-HEIC: si lo nativo falla, conversión con libheif (heic2any) y se
+ *     repite la cascada sobre el JPEG resultante. Cubre HEIC/HEIF real y
+ *     también los «.jpg» que en realidad traen contenido HEIC (típico al
+ *     pasar fotos de iPhone por apps de mensajería/nube). libheif rechaza
+ *     en milisegundos archivos que no sean HEIF (firma ftyp), así que
+ *     intentarlo con un JPEG corrupto no cuesta nada.
+ */
+export async function fileToCaptureDataUrl(file: File): Promise<string> {
+  const looksHeic = /heic|heif/i.test(file.type) || /\.hei[cf]$/i.test(file.name);
+
+  // 1) Ruta nativa (rápida; iOS decodifica HEIC aquí mismo).
+  const direct = await nativeDecodeToCappedDataUrl(file);
+  if (direct) return direct;
+
+  // 2) Rescate F-HEIC: decodificar con libheif y reintentar sobre el JPEG.
+  let heicConversionAttempted = false;
+  try {
+    const jpeg = await convertHeicToJpegBlob(file);
+    heicConversionAttempted = true;
+    const decoded = await nativeDecodeToCappedDataUrl(jpeg);
+    if (decoded) return decoded;
+  } catch {
+    /* no era HEIC o falló la conversión → error abajo */
+  }
+
+  throw new Error(
+    looksHeic || heicConversionAttempted
+      ? "No se pudo convertir la imagen HEIC"
+      : "No se pudo decodificar la imagen (formato no soportado o archivo corrupto)"
+  );
+}
+
 /** Fuente de imagen para el pipeline: data URL o elemento YA decodificado
  *  (HTMLImageElement/HTMLCanvasElement). Pasar el elemento evita re-decodificar
  *  la foto de 12 MP en cada etapa de la captura (downscale→detect→quality).
```

*Fin del documento.*
