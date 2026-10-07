# CONTEXTO DEL PROYECTO — Digitalizador E-14 (digielect)

> Este archivo fue generado como guía de contexto para análisis por una IA externa.

## 1. ¿Qué es este proyecto?

Módulo **Digitalizador de Actas E-14** (escáner de documentos electorales) construido en **Next.js 16 + TypeScript**, que replica la experiencia de un web scanner móvil profesional para digitalizar actas electorales colombianas (formato E-14).

El flujo completo es: **Captura con cámara → Detección de bordes → Recorte automático → Filtro B/N adaptativo → Análisis con GLM (IA) → Revisión → Envío**, con pestañas de **Escanear / Actas / Resumen**.

- Repositorio de referencia del proyecto original: `https://github.com/Jg-Stevan/digielect`
- Este código es la réplica/mejora de la parte del **digitalizador** únicamente.

## 2. Stack técnico

| Capa | Tecnología |
|---|---|
| Framework | Next.js 16 (App Router) + React 19 + TypeScript 5 |
| Estilos | Tailwind CSS 4 + shadcn/ui (New York) + Lucide icons |
| Estado | Zustand (cliente) |
| Base de datos | Prisma ORM + SQLite (`db/custom.db`, schema en `prisma/schema.prisma`) |
| IA | `z-ai-web-dev-sdk` (GLM) — SOLO en backend, para análisis de actas |
| Detección de documento | OpenCV.js en Web Worker (`public/e14/deteccion-worker.js`) |

## 3. Estructura clave del código

```
src/
├── app/
│   ├── page.tsx                      # Única ruta visible (renderiza DigitalizadorApp)
│   └── api/
│       ├── actas/route.ts            # CRUD de actas
│       ├── actas/analizar/route.ts   # Análisis con GLM (backend)
│       ├── actas/[id]/imagen/route.ts
│       └── bootstrap/route.ts
├── components/digitalizador/
│   ├── DigitalizadorApp.tsx          # Shell principal con pestañas
│   ├── PantallaCaptura.tsx           # Cámara, captura, detección de bordes
│   ├── PantallaRevision.tsx          # Editor de imagen + revisión
│   ├── PantallaControl.tsx           # Pestaña Actas
│   ├── PantallaResumen.tsx           # Pestaña Resumen
│   ├── PantallaExito.tsx / PantallaContingencia.tsx / shared.tsx
└── lib/digitalizador/
    ├── escaner.ts                    # Lógica del escáner (recorte, filtros)
    ├── quality.ts                    # Métricas de calidad (score 0-10)
    ├── use-camara.ts                 # Hook de cámara
    ├── store.ts                      # Estado global Zustand
    ├── reglas.ts / actas-reales.ts / types.ts
public/
├── e14/deteccion-worker.js           # Web Worker OpenCV (detección de documento)
└── actas/                            # Actas E-14 REALES de prueba (+ versiones mini/)
upload/                               # Diseños y especificaciones de referencia
├── stitch_designs/stitch_este_quedo_pleno/   # Pantallas de diseño (screen.png + code.html)
├── INSTRUCCIONES-AGENTE-web-scanner.md
├── ESPECIFICACION-REPLICA-ESCANNER.md
└── "Guía de construcción desde cero — Escáner de documentos web.md"
```

## 4. Cómo ejecutarlo

```bash
bun install          # o npm install
bun run db:push      # sincronizar schema Prisma con SQLite
bun run dev          # servidor de desarrollo en puerto 3000
```

## 5. Estado actual y TAREAS PENDIENTES (lo más importante)

El usuario ya validó la implementación base ("quedó bien"), pero dejó **5 correcciones pendientes** (probablemente la razón de este análisis):

1. **Quitar los filtros de imagen del editor**: siempre se usará B/N, el selector de filtros debe eliminarse para que el acta se vea bien.
2. **Información del acta como notificación temporal (~5 seg)**: el bloque de info del acta que aparece arriba debe comportarse como notificación flotante que desaparece a los ~5 segundos, y después mostrar la "información pequeña" del diseño de referencia (ver pantallas `digitalizador_e_14_revisi_n_con_notificaci_n_flotante_*` en `upload/stitch_designs/`).
3. **Análisis GLM en segundo plano**: quitar de la UI visible todo lo relacionado al "análisis con GLM"; debe ejecutarse en background sin estorbar la vista del acta.
4. **Consistencia de lente de cámara**: la captura se está haciendo con "gran angular" pero el preview en vivo muestra "la angular". La captura debe usar el mismo lente que el preview (la angular) para mejor calidad.
5. **Recorte automático no funciona**: la detección/recorte automático de bordes no se está aplicando a la imagen capturada. Revisar `public/e14/deteccion-worker.js` y `src/lib/digitalizador/escaner.ts`.

## 6. Reglas del proyecto (restricciones del usuario)

- **NO restaurar**: bosquejo guía (sketch overlay), función QR, ni imágenes demo/recreadas.
- Usar **actas reales** (incluidas en `public/actas/`), aunque el escáner debe detectar **cualquier documento**.
- Solo modificar la pestaña **Escanear** + editor de imagen. **NO tocar** las pestañas Actas y Resumen (ya están bien organizadas).
- Mantener **colores claros** (tema light) y réplica fiel de los diseños de Stitch (`upload/stitch_designs/`).
- El SDK `z-ai-web-dev-sdk` (GLM) debe usarse **solo en backend** (API routes), nunca en el cliente.

## 7. Notas técnicas relevantes

- El score de calidad va de 0–10: <5 rechazada (repetir obligatorio), 5–8 advertencia, >8 enviada correctamente (ver pantallas de diseño).
- `worklog.md` en la raíz contiene el registro histórico de desarrollo de cada agente (útil para entender decisiones tomadas).
- La DB SQLite (`db/custom.db`) ya contiene datos de actas de prueba.
- Las imágenes de las actas usan el patrón de nombre `E14_XXX_X_88_..._XXX-N.jpg`.
