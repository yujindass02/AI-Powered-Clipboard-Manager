# Gravity-Free Refugee/Returnee Visualization System Design

## 1) Product Goal

Create an immersive digital dashboard where each refugee record is visualized as a floating card in a calm, zero-gravity 3D space. The system should support exploration, filtering, updates, and operational monitoring while preserving clarity, accessibility, and security.

---

## 2) Experience Summary

- **Data cards float gently** in 3D with soft drift and low angular rotation.
- **Cards are color-coded by status**:
  - **Blue** = Refugee
  - **Green** = Returnee
- **Card click expands details** in place (or side panel on smaller screens).
- **Status updates animate** with color transition, upward lift, and timestamp pulse.
- **Users can search, filter, drag, and zoom** with smooth interactions.
- **Dashboard mode clusters cards** by Camp, Year of Return, or District.

---

## 3) High-Level Architecture

```text
┌────────────────────────────── Frontend (React + Three.js) ──────────────────────────────┐
│                                                                                            │
│  UI Overlay Layer (React): Search, Filter Bubbles, Dashboard Toggles, Details Drawer      │
│  3D Scene Layer (react-three-fiber + custom shaders): Floating Card Meshes + Particles    │
│  Physics Adapter (Cannon.js / Rapier): low-gravity drift, damping, gentle collisions      │
│  State Layer (Zustand/Redux): records, physics handles, selection, filters, clustering    │
│  Realtime Client: WebSocket/SSE for status updates                                         │
│                                                                                            │
└───────────────────────────────▲────────────────────────────────────────────────────────────┘
                                │
                      REST + WebSocket/SSE
                                │
┌────────────────────────────────┴────────────────────────────────────────────────────────────┐
│ Backend API                                                                                 │
│ - Refugee record CRUD                                                                        │
│ - Filter/search endpoints                                                                     │
│ - Status transition events                                                                    │
│ - Aggregation endpoints for cluster summaries                                                 │
│ - AuthN/AuthZ, audit logs, encryption                                                        │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Recommended stack

- **3D Rendering:** Three.js via `@react-three/fiber`
- **Physics:** Cannon.js (`@react-three/cannon`) or Rapier (`@react-three/rapier`)
- **UI:** React + lightweight component system
- **Data:** REST + WebSocket for live updates
- **Performance:** Instanced rendering + worker-based physics step (if needed)

---

## 4) Data Model

```ts
type RefugeeStatus = 'Refugee' | 'Returnee';

type RefugeeRecord = {
  id: string;
  fullName: string;
  age: number;
  gender: 'Female' | 'Male' | 'Other';
  camp: string;
  districtSriLanka: string;
  countryOfAsylum: string;
  yearOfReturn?: number;
  status: RefugeeStatus;
  updatedAt: string; // ISO timestamp
  tags?: string[];
};
```

### Derived runtime fields (frontend)

```ts
type CardRuntime = {
  recordId: string;
  bodyId: string;            // physics body key
  position: [number, number, number];
  rotation: [number, number, number];
  velocity: [number, number, number];
  isExpanded: boolean;
  glowIntensity: number;
  timestampPulseUntil?: number;
};
```

---

## 5) Scene & Visual Design

### World settings

- **Background:** deep-space gradient (`#05070D` → `#0B1020`)
- **Ambient light:** soft low-intensity
- **Directional fill light:** gentle cool tone
- **Subtle fog:** very low density to add depth
- **Particle layer:** tiny slow particles with minimal opacity

### Card styling

- Rectangular plane with rounded corners (shader mask or geometry)
- Soft emissive edge glow:
  - Refugee: blue glow (`#4DA3FF`)
  - Returnee: green glow (`#56D364`)
- Minimal text density on collapsed state
- Expanded state displays full profile and action controls

---

## 6) Physics Behavior (Calm Antigravity)

### Core constraints

- Use **near-zero gravity** (e.g., `gravity = [0, 0, 0]`)
- Add **micro-drift force** (Perlin noise / low-frequency random vectors)
- Add **high linear damping** and **moderate angular damping**
- Keep velocities clamped to low values

### Collision settings

- Use box colliders matching card dimensions
- Low restitution (no bounce chaos)
- Slightly increased contact damping and friction
- Optional broadphase partitioning for 1,000+ cards

### Example tuning (starting point)

```ts
physics: {
  gravity: [0, 0, 0],
  solverIterations: 8,
  defaultMaterial: {
    friction: 0.2,
    restitution: 0.05
  },
  cardBody: {
    linearDamping: 0.92,
    angularDamping: 0.85,
    mass: 0.8,
    maxSpeed: 0.35,
    maxSpin: 0.25
  }
}
```

---

## 7) Interaction Design

## 7.1 Floating data objects

- Hover: increases glow + subtle scale-up
- Click: card expands (depth offset to avoid overlap)
- Mouse motion: scene receives mild attraction field around cursor

## 7.2 Drag-and-release

- Pointer down captures a kinematic constraint
- Drag follows cursor-projected plane in world space
- Release applies very small carry velocity and restores dynamic body

## 7.3 Search and filters

- Floating top search bar (React overlay)
- Filter bubbles:
  - Camp
  - Year
  - Gender
  - District
- Filtered-out cards reduce opacity and drift to peripheral volume

## 7.4 Zoom

- Scroll wheel modifies camera distance (bounded min/max)
- Keep UI overlays fixed in viewport

---

## 8) Status Transformation Animation

When `status: Refugee -> Returnee`:

1. Trigger event in state store.
2. Animate color interpolation (blue to green, 450–700ms, easing-out).
3. Apply soft upward impulse (`y += 0.08` velocity cap-safe).
4. Show timestamp chip near card (`Updated 14:22:11`) for ~2 seconds.
5. Persist final color and record in audit trail.

Pseudo-flow:

```ts
onStatusChange(recordId, nextStatus) {
  updateStore(recordId, { status: nextStatus, updatedAt: nowISO() });

  animateColor(recordId, fromBlue, toGreen, 600);
  applyImpulse(recordId, [0, 0.08, 0]);
  showTimestampPulse(recordId, 2000);
}
```

---

## 9) Dashboard Mode Clustering

### Modes

- Cluster by **Camp**
- Cluster by **Year of Return**
- Cluster by **District (Sri Lanka)**

### Approach

- Compute 3D anchor points for cluster centers
- For each card, blend from free-drift target to cluster anchor target
- Preserve local separation with repulsion and collision checks
- Animate transition over 800–1600ms

### Visual cues

- Cluster labels as floating billboards
- Optional hull glow / ring around each cluster
- Badge counts on labels

---

## 10) Performance Plan (1,000+ Cards)

1. **Instanced meshes** for collapsed cards.
2. Promote only selected/hovered cards to fully interactive meshes.
3. Frustum culling + distance-based detail reduction.
4. Batch text rendering (SDF font atlas).
5. Throttle expensive state sync between physics and React.
6. Run physics in fixed timestep (`1/60`) with interpolation.
7. Optionally offload physics to worker thread.
8. Debounce search/filter recomputation.

---

## 11) Security & Privacy

- Token-based authentication (short-lived JWT + refresh).
- Role-based access controls (viewer/editor/admin).
- Encrypt all traffic via TLS.
- Encrypt sensitive fields at rest.
- Structured audit logging for record updates.
- PII masking in collapsed card mode (show only minimal info).
- Configurable retention and export policy.

---

## 12) API Contract (Example)

### REST

- `GET /api/refugees?query=&camp=&year=&gender=&district=`
- `PATCH /api/refugees/:id/status`
- `GET /api/aggregates?groupBy=camp|year|district`

### Realtime event

```json
{
  "type": "refugee.status.updated",
  "recordId": "rf_1029",
  "prevStatus": "Refugee",
  "nextStatus": "Returnee",
  "updatedAt": "2026-02-19T09:12:45.000Z",
  "actor": "user_44"
}
```

---

## 13) Suggested Frontend Module Layout

```text
src/
  app/
    App.tsx
    providers.tsx
  features/refugeeScene/
    RefugeeScene.tsx
    CardInstancedLayer.tsx
    ExpandedCard.tsx
    ParticleField.tsx
    physics.ts
    clustering.ts
    transitions.ts
  features/filters/
    FilterBar.tsx
    FilterBubbleGroup.tsx
    SearchInput.tsx
  features/dashboard/
    ClusterModeToggle.tsx
    ClusterLabels.tsx
  state/
    useRefugeeStore.ts
  api/
    refugeeApi.ts
    realtime.ts
```

---

## 14) Minimal Interaction Pseudocode

```ts
// per frame
for each card {
  if (!dashboardMode) {
    applyMicroDrift(card);
  } else {
    steerTowardClusterAnchor(card, clusterAnchor[card.groupKey]);
  }

  clampVelocity(card, maxSpeed);
  damp(card, linearDamping, angularDamping);
}

// on click
setExpanded(recordId, true);
focusCard(recordId);

// on drag
bindPointerConstraint(recordId);

// on filter
applyFilterState(filters);
animateVisibilityTransitions();
```

---

## 15) Accessibility & Usability

- Keyboard navigation for card selection and expansion.
- Reduced-motion mode disables spin and minimizes drift.
- High-contrast palette option.
- Tooltips and explicit legends for status colors.
- Consistent text sizing in overlays.

---

## 16) Delivery Roadmap

### Phase 1 (MVP)

- Floating cards, color coding, click expand
- Search + core filters
- Basic low-gravity collisions
- Live status update + transformation animation

### Phase 2

- Dashboard clustering modes + labels
- Performance tuning to 1,000 cards
- Audit view and role controls

### Phase 3

- Advanced analytics overlays
- Historical playback timeline
- Geographic linking and export tooling

---

## 17) Acceptance Criteria Mapping

- ✅ Floating rectangular cards with controlled drift and rotation
- ✅ Color coding (blue refugee, green returnee)
- ✅ Expand-on-click profile detail view
- ✅ Animated status transformation with timestamp pulse
- ✅ Search, drag-release, filters, and zoom controls
- ✅ Dashboard clustering by camp/year/district
- ✅ Calm physics behavior with gentle collisions
- ✅ Professional dark visual style with subtle particles
- ✅ Scalable strategy for 1,000+ objects
- ✅ Secure API-ready architecture for live updates
