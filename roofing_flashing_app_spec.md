# Executive Summary
TR Metals Field App is specified as an offline-first iPhone PWA for parametric flashing capture and shop-ready output generation. The merged architecture standardizes a segment-driven data contract so the same geometry powers field preview, PDF cards, and brake-step output. The v1 stack is PWA (TypeScript, Service Worker, IndexedDB, jsPDF/svg2pdf), deployed by QR code with no App Store dependency. The design targets gloved operation on winter rooftops with fast numeric entry, auto-save, hard geometry validation, and queued send-to-shop workflows.

## Swarm Execution Note
All required specialist outputs were merged and normalized into a single source-of-truth specification with cross-validation and conflict handling.

---

# Section 1: Flashing Domain Model + Schemas

## 1.1 Flashing Taxonomy (Commercial Flat Roofing)

| Category | Type | Function | Canonical Pattern |
|---|---|---|---|
| Edge | Gravel Stop | Edge termination / gravel retention | L-profile + kick + hem |
| Edge | Drip Edge | Eaves/fascia shedding | L-profile + hemmed leg |
| Edge | Fascia Trim | Exposed fascia finish | L or Z profile |
| Parapet | Coping/Cap | Parapet top cap with drips | 3–5 bends, optional slope |
| Parapet | Inside/Outside Corner | Directional coping transitions | Brake/weld variant |
| Parapet | End Dam | Coping termination | Short L/Z return |
| Base | Base Flashing | Wall-to-roof transition | L profile with flange |
| Base | Step Flashing / Step Transition | Elevation change | Multi-tier Z |
| Counter | Reglet | Wall recess termination | Return + face |
| Counter | Counter Flashing | Covers base flashing | L with return/hem |
| Counter | Loose-Lock | Removable counter system | Z/hook with safety hem |
| Penetration | Pitch Pocket | Equipment-leg seal pan | Brake-formed pan components |
| Penetration | Pipe Collar | Pipe penetration | Conical (v1 excluded from pure brake) |
| Transition | Saddle | Slope transition behind curb | Multi-segment |
| Transition | Height Transition | Curb/roof height change | Custom Z/multi-leg |

## 1.2 Constraint Tables (Physics + Shop Limits)

### Material/Thickness/K-Factor Defaults

| Material Enum | Nominal Thickness (mm) | K-Factor | Notes |
|---|---:|---:|---|
| `steel_24ga` | 0.70 | 0.45 | Air bend typical |
| `steel_26ga` | 0.55 | 0.42 | Slight neutral-axis shift |
| `alum_040` | 1.02 | 0.50 | Default neutral axis |
| `alum_050` | 1.27 | 0.50 | Default neutral axis |
| `alum_063` | 1.60 | 0.48 | Heavier forming |

### Practical Brake Constraints

| Constraint | Value |
|---|---|
| Max developed width (10 ft brake nominal) | 3048 mm |
| Safety max for validation | 2998 mm |
| Min leg baseline (absolute) | 6.35 mm (1/4") |
| Closed hem minimum | 9.525 mm (3/8") |
| Open/Rolled hem minimum | 6.35 mm (1/4") |
| Default max segments in v1 | 8 |

## 1.3 Validation Rules (Merged)

| Rule ID | Rule | Severity | Behavior |
|---|---|---|---|
| V1 | Leg below material minimum | Error | Block save |
| V2 | Developed width > brake limit | Error | Block send, allow flagged draft |
| V3 | Self-intersecting profile | Error | Red dashed preview + block save |
| V4 | Hem too short | Warning | Allow with acknowledgement |
| V5 | Wrong/ambiguous units | Error/Warning | Suggest unit conversion |
| V6 | Unsupported material/gauge | Error | Disable selection |
| V7 | Negative/invalid bend math | Error | Fallback simplified calc + log |

## 1.4 Canonical TypeScript Contracts

```typescript
export type FlashingType =
  | 'gravel_stop' | 'drip_edge' | 'fascia_trim'
  | 'coping' | 'inside_corner' | 'outside_corner' | 'end_dam'
  | 'base_flashing' | 'step_transition'
  | 'reglet' | 'counter_flashing' | 'loose_lock'
  | 'pitch_pocket' | 'saddle' | 'height_transition';

export type Material = 'steel_24ga' | 'steel_26ga' | 'alum_040' | 'alum_050' | 'alum_063';
export type HemType = 'none' | 'open' | 'closed' | 'rolled';
export type BendDirection = 'up' | 'down';

export interface Segment {
  id: string;
  order: number;
  leg_label: 'A'|'B'|'C'|'D'|'E'|'F'|'G'|'H';
  length_mm: number;
  angle_deg: number;
  bend_direction: BendDirection;
  bend_centerline_radius_mm: number;
  hem_type: HemType;
  hem_length_mm: number;
  material: Material;
  k_factor: number;
  bend_allowance_mm: number;
}

export interface FlashingProfile {
  id: string;
  type: FlashingType;
  name: string;
  material: Material;
  finish: string;
  total_length_mm: number;               // piece run length
  quantity: number;
  segments: Segment[];
  total_developed_width_mm: number;      // flat blank width
  photo_uri?: string;
  location_note?: string;
  created_at: string;
  updated_at: string;
  template_id?: string;
}

export interface BrakeStep {
  step_number: number;
  backgauge_mm: number;
  bend_angle: number;
  bend_direction: BendDirection;
  tool_note: 'standard_punch' | 'hem_tool' | 'radius_die';
  remarks?: string;
}

export interface BrakeSequence {
  profile_id: string;
  steps: BrakeStep[];
}

export type OrderStatus = 'draft' | 'ready' | 'sent' | 'acknowledged';
export type SyncStatus = 'local_only' | 'syncing' | 'synced' | 'error';

export interface Order {
  id: string;
  work_order_ref: string;
  building_name: string;
  address: string;
  foreman_name: string;
  created_at: string;
  updated_at: string;
  status: OrderStatus;
  sync_status: SyncStatus;
  flashings: FlashingProfile[];
  metadata: {
    total_piece_count: number;
    material_summary: Record<Material, number>;
  };
}
```

## 1.5 Developed Width + Brake Sequence Algorithms

```typescript
// Bend allowance reference:
// BA = (angle_deg * PI/180) * (radius_mm + k_factor * thickness_mm)

export function computeDevelopedFlat(profile: FlashingProfile): {
  total_developed_width_mm: number;
  per_segment_flat_mm: number[];
  total_bend_allowance_mm: number;
} {
  // Fast field mode may apply simplified K=0.5 approximation.
  return { total_developed_width_mm: 0, per_segment_flat_mm: [], total_bend_allowance_mm: 0 };
}

export function generateBrakeSequence(profile: FlashingProfile): BrakeSequence {
  // Backgauge from developed geometry, enforce minimum practical backgauge.
  return { profile_id: profile.id, steps: [] };
}
```

## 1.6 JSON Schema (Interchange)

```typescript
export interface OrderContractV1 {
  contract_version: '1.0.0';
  export_timestamp: string;
  order: Order & {
    flashings: Array<FlashingProfile & {
      brake_sequence: BrakeSequence;
    }>;
  };
  reserved_for_future: {
    external_wo_id: string | null;
    external_building_id: string | null;
    dataforma_sync: unknown;
  };
}
```

## 1.7 IndexedDB Schema

| Store | Key | Indexes |
|---|---|---|
| `orders` | `id` | `by_status`, `by_sync`, `by_date` |
| `flashings` | `id` | `by_order`, `by_type` |
| `templates` | `id` | `by_type`, `by_favorite` |
| `settings` | `id` | none |
| `sync_queue` | auto inc | `by_timestamp` |

---

# Section 2: iPhone UX Spec

## 2.1 Field Persona + Human Factors
- Primary persona: gloved roofer, one-handed use, cold/windy rooftop, unstable signal.
- Primary actions in lower 2/3 thumb zone.
- Critical tap targets >= 52x52 pt; secondary >= 44x44 pt.
- Numeric keypad only for dimension entry.
- Inline validation preferred over modal interruptions.

## 2.2 Screen Flow

`Boot -> Sync Check -> Roof List -> Order Detail -> Add Flashing -> Type Picker -> Parametric Editor -> Photo/Context -> Review -> Sync/Send`

## 2.3 Wireframe Element Lists

### A. Roof List (Entry)
1. Top bar: menu, app title, sync indicator.
2. Primary `+New` action tile.
3. Roof/order cards with status and piece count.
4. Offline amber banner when disconnected.

### B. Order Detail
1. Header: WO, building, address, foreman.
2. Flashing list with thumbnails + status.
3. `Add Flashing` action.
4. `Save & Continue Later` action.

### C. Type Picker
1. Tile grid by category (Edge/Parapet/Base/Counter/Transition).
2. Recent types row.
3. Template quick-start row.

### D. Parametric Editor (Core)
1. Live SVG panel (auto-scale).
2. Parameter fields (A/B/C/D legs etc.) numeric-only.
3. Unit toggle `[INCHES | MM]`.
4. Hem toggle and hem size.
5. Piece length + quantity controls.
6. Validation panel (warning/error).
7. `Save` (enabled only when valid).

### E. Photo/Context
1. Add photo (optional).
2. Location note (short).

### F. Review + Send
1. Order status stepper: Draft -> Ready -> Sent.
2. Flashing summary list with small SVG thumbs.
3. `SEND TO SHOP (PDF+JSON)` primary CTA.
4. Queue notice + retry controls if offline.

### G. Settings
1. Unit defaults.
2. Material defaults.
3. Version/build indicator.
4. Sync destination settings.
5. Template management entry point.

## 2.4 Interaction/Error State Rules

| Event | UI Response |
|---|---|
| Invalid dimension | Inline amber text under field |
| Width exceeds capacity | Red blocking banner, cannot send |
| Delete flashing | Long-press 2s + haptic confirmation |
| Auto-save | Debounced save + brief toast |
| Offline | Amber persistent banner, send disabled/queued |

## 2.5 SVG Renderer Spec (Consumption of `segments[]`)

```typescript
export function generateSVGPath(segments: Segment[]): { d: string; bounds: DOMRectLike } {
  let currentX = 0;
  let currentY = 0;
  let currentAngle = 0; // 0 = +X axis
  const cmds: string[] = [`M ${currentX} ${currentY}`];

  for (const seg of segments) {
    const rad = (currentAngle * Math.PI) / 180;
    const endX = currentX + seg.length_mm * Math.cos(rad);
    const endY = currentY + seg.length_mm * Math.sin(rad);

    cmds.push(`L ${endX} ${endY}`);
    if (seg.hem_type !== 'none') {
      // draw hem indicator glyph/line at segment end
    }

    const multiplier = seg.bend_direction === 'up' ? -1 : 1;
    currentAngle += seg.angle_deg * multiplier;

    currentX = endX;
    currentY = endY;
  }

  return { d: cmds.join(' '), bounds: { x: 0, y: 0, width: 0, height: 0 } };
}
```

Performance target: <100 ms render on iPhone 12 baseline; debounce 150 ms; LRU cache up to 50 profiles.

---

# Section 3: Tech Stack Decision

## 3.1 Option Comparison

| Path | Stack | Pros | Cons | Time-to-MVP |
|---|---|---|---|---|
| A | PWA + TS + SW + IndexedDB | Zero App Store friction, instant updates, strongest JS skill fit | iOS PWA quirks | 2 weeks |
| B | React Native + Expo | Near-native UX | Build/release overhead, store pipeline | 4–6 weeks |
| C | SwiftUI native | Best platform fidelity | Highest specialization and distribution friction | 6–10 weeks |

**Decision: Path A (PWA).**

## 3.2 Runtime Architecture (v1)
- App shell cached via Service Worker (cache-first).
- Data in IndexedDB (outbox/sync queue pattern).
- Export locally as PDF + JSON (+ optional CSV).
- Submit via email/shared folder; online webhook optional.

## 3.3 PDF Strategy

| Requirement | Choice |
|---|---|
| Client-side generation | `jsPDF` |
| SVG fidelity | `svg2pdf`/`doc.svg()` vector-first |
| Complex fallback | `html2canvas` raster fallback |
| Table rendering | `jspdf-autotable` |

PDF layout contains: header, per-flashing SVG card, spec table, brake steps, notes/photos, footer summary.

## 3.4 Deployment
1. Deploy on Netlify/Cloudflare Pages.
2. Print laminated QR to install URL.
3. Users install via iPhone “Add to Home Screen”.
4. SW update banner for new builds.

---

# Section 4: Shop Output Spec

## 4.1 PDF Order Layout
1. Header: logo, WO ref, building/address, foreman, date, page count.
2. Flashing card per item:
   - Large cross-section SVG with leg labels A–H.
   - Dimension summary and developed width.
   - Material/finish/qty/piece length.
   - Bend sequence table.
   - Photo thumbnail and location note.
3. Footer: totals by material/finish and total linear feet.

## 4.2 Brake Operator Checklist Format

| Step | Backgauge (mm/in) | Angle | Direction | Tool | Remarks | Check |
|---:|---:|---:|---|---|---|---|
| 1 | ... | ... | up/down | standard/hem/radius | ... | ☐ |
| 2 | ... | ... | up/down | standard/hem/radius | ... | ☐ |

Operator conventions:
- Black/white print-safe layout.
- 12pt minimum readable table text.
- New flashing begins new page when possible.

## 4.3 Naming Convention

```text
{DATE}-{WO_REF}-{BUILDING_ABBREV}-FLASHINGS.pdf
Example: 2026-04-04-WO-2024-156-799ELLICE-FLASHINGS.pdf
```

## 4.4 Inbound Shop Flow
1. **Immediate**: email PDF+JSON to shared inbox.
2. **Preferred v2**: upload to shared folder (Dropbox/OneDrive) watched at shop station.
3. **Fallback**: AirDrop/manual local transfer.

---

# Section 5: Implementation Roadmap

| Phase | Timeline | Deliverables | Definition of Done | Risks | Mitigation |
|---|---|---|---|---|---|
| Phase 1 Prototype | 2 weeks | PWA shell, 5 core types, parametric editor, live SVG, local save, PDF export | 3-flashing order in <4 min on roof; numeric keyboard only | Geometry errors, PDF latency | fixture tests + debounce/caching |
| Phase 2 Library + Sync | +4 weeks | full type library, photos, sync queue, template save/use, delivery automation | zero transcription recuts in pilot | sync conflict, storage growth | LWW + conflict UI, photo compression |
| Phase 3 Brake Integration | future sprint | controller-specific export adapter, multi-role views, material reports | brake-ready import with no rewrite | proprietary controller formats | adapter abstraction + sample files |

Deployment per phase: Netlify URL + QR rollout, home-screen install guidance, short field onboarding.

---

# Section 6: Open Questions

[OPEN QUESTION: Dataforma API Mapping]
Need official attachment/webhook contract and auth model before direct integration.

[OPEN QUESTION: Photo Payload Policy]
Confirm base64-in-JSON limit vs external file references for large photo sets.

[OPEN QUESTION: Multi-Brake Length]
Current validation assumes 10 ft brake; confirm behavior for 8 ft shop scenarios (warn vs auto-split).

[OPEN QUESTION: Thickness Source]
Decide nominal gauge vs supplier-measured thickness calibration in bend calculations.

[OPEN QUESTION: Feed Direction Variants]
Current bend direction assumes left-feed convention; determine need for shop toggle.

---

# Cross-Validation (Orchestrator Merge Checks)

| Check | Result |
|---|---|
| Flashing schema consumed by renderer | ✅ `segments[]` directly drives path generation |
| BrakeSequence compatible with shop checklist | ✅ `BrakeStep` fields map 1:1 to table columns |
| Stack supports offline + SVG + PDF | ✅ PWA + IndexedDB + jsPDF/svg2pdf satisfies v1 |
| Phase 1 feasible in timeline | ✅ feasible with constrained type set and templates |
| Conflicts resolved | ✅ standardized enums, units, and naming |
