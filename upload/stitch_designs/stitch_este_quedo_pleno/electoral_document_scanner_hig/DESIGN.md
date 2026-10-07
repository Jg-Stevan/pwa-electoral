---
name: Electoral Document Scanner HIG
colors:
  surface: '#faf9fe'
  surface-dim: '#dad9df'
  surface-bright: '#faf9fe'
  surface-container-lowest: '#ffffff'
  surface-container-low: '#f4f3f8'
  surface-container: '#eeedf3'
  surface-container-high: '#e9e7ed'
  surface-container-highest: '#e3e2e7'
  on-surface: '#1a1b1f'
  on-surface-variant: '#414755'
  inverse-surface: '#2f3034'
  inverse-on-surface: '#f1f0f5'
  outline: '#717786'
  outline-variant: '#c1c6d7'
  surface-tint: '#005bc1'
  primary: '#0058bc'
  on-primary: '#ffffff'
  primary-container: '#0070eb'
  on-primary-container: '#fefcff'
  inverse-primary: '#adc6ff'
  secondary: '#006e28'
  on-secondary: '#ffffff'
  secondary-container: '#6ffb85'
  on-secondary-container: '#00732a'
  tertiary: '#bc000a'
  on-tertiary: '#ffffff'
  tertiary-container: '#e2241f'
  on-tertiary-container: '#fffbff'
  error: '#ba1a1a'
  on-error: '#ffffff'
  error-container: '#ffdad6'
  on-error-container: '#93000a'
  primary-fixed: '#d8e2ff'
  primary-fixed-dim: '#adc6ff'
  on-primary-fixed: '#001a41'
  on-primary-fixed-variant: '#004493'
  secondary-fixed: '#72fe88'
  secondary-fixed-dim: '#53e16f'
  on-secondary-fixed: '#002107'
  on-secondary-fixed-variant: '#00531c'
  tertiary-fixed: '#ffdad5'
  tertiary-fixed-dim: '#ffb4aa'
  on-tertiary-fixed: '#410001'
  on-tertiary-fixed-variant: '#930005'
  background: '#faf9fe'
  on-background: '#1a1b1f'
  surface-variant: '#e3e2e7'
typography:
  headline-xl:
    fontFamily: Inter
    fontSize: 34px
    fontWeight: '700'
    lineHeight: 41px
  headline-xl-mobile:
    fontFamily: Inter
    fontSize: 28px
    fontWeight: '700'
    lineHeight: 34px
  headline-lg:
    fontFamily: Inter
    fontSize: 28px
    fontWeight: '700'
    lineHeight: 34px
  headline-lg-mobile:
    fontFamily: Inter
    fontSize: 22px
    fontWeight: '700'
    lineHeight: 28px
  headline-md:
    fontFamily: Inter
    fontSize: 20px
    fontWeight: '600'
    lineHeight: 25px
  headline-sm:
    fontFamily: Inter
    fontSize: 17px
    fontWeight: '600'
    lineHeight: 22px
  body-lg:
    fontFamily: Inter
    fontSize: 17px
    fontWeight: '400'
    lineHeight: 22px
  body-md:
    fontFamily: Inter
    fontSize: 15px
    fontWeight: '400'
    lineHeight: 20px
  body-sm:
    fontFamily: Inter
    fontSize: 13px
    fontWeight: '400'
    lineHeight: 18px
  label-lg:
    fontFamily: Inter
    fontSize: 15px
    fontWeight: '600'
    lineHeight: 20px
  label-md:
    fontFamily: Inter
    fontSize: 13px
    fontWeight: '500'
    lineHeight: 18px
  label-sm:
    fontFamily: Inter
    fontSize: 11px
    fontWeight: '500'
    lineHeight: 13px
  caption:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: '400'
    lineHeight: 16px
rounded:
  sm: 0.25rem
  DEFAULT: 0.5rem
  md: 0.75rem
  lg: 1rem
  xl: 1.5rem
  full: 9999px
spacing:
  gutter: 1rem
  gutter-tablet: 1.25rem
  gutter-desktop: 1.5rem
  margin: 1rem
  margin-tablet: 1.5rem
  margin-desktop: 2rem
  space-xs: 0.25rem
  space-sm: 0.5rem
  space-md: 0.75rem
  space-lg: 1rem
  space-xl: 1.25rem
---

## Brand & Style

This design system translates the Apple Human Interface Guidelines into an authoritative, mission-critical document-scanning utility for tally sheets, actas, and election certificates (Form E-14). Designed for field electoral delegates, poll workers, and audit observers, the interface prioritizes absolute clarity, fast optical capture feedback, and zero cognitive friction during high-stress verification workflows.

The aesthetic fuses **Modern iOS Native Precision** with functional **Glassmorphism**:
- **Clarity over ornament:** Uncluttered layouts with primary emphasis placed on the physical document viewport, OCR boundary highlights, and numerical tallies.
- **Immediate tactile affirmation:** Controls provide instant visual feedback mirroring native iOS haptics and touch responses, ensuring confidence during rapid document logging.
- **Civic integrity & authority:** Balanced neutral grounds, crisp system typography, and strict semantic color enforcement ensure all election tallies and audit flags are unambiguous under harsh lighting conditions.

## Colors

The color palette adheres strictly to standard iOS semantic palettes, calibrated to meet or exceed WCAG 2.2 AA contrast requirements (minimum 4.5:1 for standard text, 3:1 for graphical elements and large text).

### Semantic Roles
- **Primary (`#007AFF` - System Blue):** Used for standard interactive controls, active pagination indicators, selection states, camera shutter rings, and key call-to-action triggers.
- **Secondary (`#34C759` - System Green):** Reserved exclusively for verified states, such as valid cryptographic signatures, complete checksum matches, accepted OCR reads, and successful document transmissions.
- **Tertiary (`#FF3B30` - System Red):** Reserved for unreadable crop corners, checksum discrepancies, vote count mismatches, transmission failures, and destructive actions.
- **Neutral Accent (`#8E8E93` - System Gray):** Used for non-active icons, camera bounding guides in search mode, and secondary metadata labels.

### Surface System
- **Light Mode Canvas:** `#F2F2F7` (System Grouped Background Light) acts as the base platform surface. Cards and grouped lists sit elevated on `#FFFFFF`.
- **Dark Mode Canvas:** `#000000` (System Background Dark) with grouped modules resting on `#1C1C1E` (Secondary System Background Dark) and elevated modals on `#2C2C2E`.
- **Translucent Overlays:** Semi-transparent backdrops (`rgba(255, 255, 255, 0.78)` in light, `rgba(28, 28, 30, 0.82)` in dark) apply to floating header bars, bottom toolbars, and active scanner viewfinder HUDs with `backdrop-filter: blur(20px) saturate(180%)`.

## Typography

Inter serves as the primary typographic engine, leveraging its systematic metric execution, neutral geometry, and high legibility to mirror Apple's native San Francisco (SF Pro) ecosystem across multi-platform web deployments.

### Typographic Hierarchy Rules
- **Large Titles (`headline-xl` / `headline-xl-mobile`):** Reserved for screen entry headers and high-level verification dashboards.
- **Section Headers (`headline-sm` / `label-lg`):** Standard uppercase grouped list headers use `label-md` with `letter-spacing: -0.08px` and neutral secondary tone.
- **Numeric & Tally Data:** Vote counts, mesa codes, and OCR extracted digits must be rendered with tabular numeric alignment (`font-variant-numeric: tabular-nums`) to prevent layout shift during optical live-scanning updates.
- **Micro-labels & Badges:** `label-sm` is strictly enforced for OCR verification status chips, confidence interval indicators, and bottom tab bar titles.

## Layout & Spacing

The layout philosophy implements a strict iOS fluid-grouped model, utilizing system margins and safe area insets:

- **Touch Targets:** All interactive triggers (shutter controls, segmented pill switches, list rows, form stepper buttons) maintain a mandatory minimum dimension of `44px x 44px` to fulfill WCAG 2.2 AA target size criteria.
- **Mobile Grid (0–767px):** Single-column fluid presentation. Side screen margins default to `1rem` (16px). Content is clustered in iOS-style grouped cards with internal row gutters of `0.75rem`.
- **Tablet & Split-View (768–1023px):** 2-column layout. Viewport left hosts the live camera view or high-resolution document preview; viewport right hosts OCR data entry tables and tally audit sheets. Side margins expand to `1.5rem`.
- **Desktop / Auditor Console (1024px+):** Fixed central content container (maximum 1200px width) or 3-column split view (Batch Queue, Document Viewer, Data Reconciliation) with `2rem` margins and `1.5rem` grid column gaps.
- **Safe Area Insets:** All top navigation bars incorporate `env(safe-area-inset-top)` and floating toolbars reserve `env(safe-area-inset-bottom)` to ensure hardware-level notch and home-indicator clearance.

## Elevation & Depth

Visual hierarchy does not rely on heavy drop shadows; it relies on structured surface tones and glassmorphism.

### Surface Hierarchy
1. **Level 0 (Base Layer):** Canvas background (`#F2F2F7` Light / `#000000` Dark). Unclickable, holds the grouped structure.
2. **Level 1 (Card & Row Layer):** Form groupings, document summaries, and data cards (`#FFFFFF` Light / `#1C1C1E` Dark). Features fine structural borders: `0.5px solid rgba(0, 0, 0, 0.08)` in light mode; `0.5px solid rgba(255, 255, 255, 0.12)` in dark mode.
3. **Level 2 (Translucent Navigation & Toolbars):** Navigation bars, search bars, and bottom action sheets float above content with blurred transparency (`rgba(255, 255, 255, 0.85)` / `rgba(28, 28, 30, 0.85)` + `backdrop-filter: blur(20px)`). Border-bottom or border-top uses `0.5px solid rgba(60, 60, 67, 0.29)`.
4. **Level 3 (Modals, HUD Overlays & Alerts):** Floating scanner edge-detection tags, live confidence scores, and action sheets. Elevated with subtle ambient diffusion:
   - Light: `0 8px 32px rgba(0, 0, 0, 0.12), 0 1px 2px rgba(0, 0, 0, 0.04)`
   - Dark: `0 8px 32px rgba(0, 0, 0, 0.48), 0 0 0 1px rgba(255, 255, 255, 0.14)`

## Shapes

The design system embraces the Apple squircle aesthetic, parameterized at roundedness level `2`.

- **Cards & Grouped Sections:** Configured with `12px` to `16px` border-radius (`rounded-lg` standard at `1rem` / 16px).
- **Buttons & Interactive Inputs:** Standard action buttons, text fields, and segmented control containers use `10px` to `12px` border-radius.
- **Segmented Control Sliders & Status Badges:** Fully rounded or pill-shaped (`rounded-full`) for status indicators, vote tags, and OCR confidence meters.
- **Document Viewport Overlays:** The live camera bounding box uses an inset squircle corner frame (`8px` corner radius) with high-visibility color transitions (System Red for unaligned, System Green for locked perspective).

## Components

### Buttons
- **Primary Action (Filled):** Height `48px` (touch-optimized), background `#007AFF`, text `#FFFFFF`, typography `label-lg`, border-radius `12px`. Hover/Active state applies an opacity fade to `0.85`.
- **Secondary / Tinted:** Background `rgba(0, 122, 255, 0.12)`, text `#007AFF`, border-radius `12px`.
- **Destructive:** Background `rgba(255, 59, 48, 0.12)`, text `#FF3B30`. Active destructive uses filled `#FF3B30` with white text.
- **Shutter Trigger:** Outer circle `72px x 72px` with a `4px` ring in `#FFFFFF`, internal gap of `3px`, and solid white or red-record center button. Minimum tap boundary `72px`.

### Native iOS Segmented Controls
- **Track:** Height `36px`, padding `2px`, background `rgba(120, 120, 128, 0.16)`, border-radius `9px`.
- **Thumb:** White background (`#FFFFFF` light mode / `#636366` dark mode), radius `7px`, subtle ambient shadow (`0 2px 4px rgba(0, 0, 0, 0.12)`). Smooth 200ms spring-transition positioning on selection.

### Form Cards & Grouped Lists
- Displayed in iOS "Inset Grouped" style. Card corners `16px`, margins `16px`.
- Dividers between table cells inset by `16px` from the left edge to align with label content. Divider color: `rgba(60, 60, 67, 0.15)`.
- Minimum row height: `48px`.

### Inputs & OCR Tally Steppers
- **Vote Count Input:** Large tabular numerical fields, centered, background `#F2F2F7` (Light) / `#2C2C2E` (Dark), minimum height `44px`, border-radius `10px`.
- **Stepper:** Dual pill split buttons (`-` and `+`) with `44px` individual touch footprints.

### Checkboxes, Switches & Radio Controls
- **Toggle Switch:** iOS standard `51px x 31px` pill track. Inactive state `#E9E9EB`, active state `#34C759` (System Green). Thumb is a crisp white circle `27px` with `0 2px 4px rgba(0, 0, 0, 0.2)`.
- **Selection Checks:** Circular radio checks featuring `#007AFF` fill with an inset white checkmark icon.

### Status Badges & Chips
- Padding: `4px 10px`, border-radius `9999px`.
- **Verified / OCR Match:** Background `rgba(52, 199, 89, 0.14)`, text `#248A3D` (accessible high-contrast green), typography `label-sm`.
- **Flagged / Audit Discrepancy:** Background `rgba(255, 59, 48, 0.14)`, text `#D70015` (accessible red), typography `label-sm`.
- **Processing / Pending:** Background `rgba(0, 122, 255, 0.14)`, text `#0051A8`, typography `label-sm`.

### Scanner Viewfinder HUD
- Semi-opaque dark mask (`rgba(0, 0, 0, 0.6)`) framing the target sheet aspect ratio (E-14 legal sheet proportion).
- Dynamic corner guides: `3px` thick lines, radius `8px`. Turns `#34C759` when perspective deskew and autofocus lock succeed.
- Floating glassmorphism indicator at top of camera frame displaying document page index (`E-14: Page 1 of 3`) with `backdrop-filter: blur(16px)`.