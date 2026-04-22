# IGLOO-STYLE PANEL SYSTEM — COMPREHENSIVE FIX PLAN

## CURRENT ISSUES IDENTIFIED

### 1. PANEL SIZE (CRITICAL)
**Current:** `.a-tab { width: 200px !important; }` (line 746)
**Problem:** Takes ~40-50% of mobile screens, blocks visuals
**Protocol Violation:** "Panels must take NO MORE THAN 25-30% of screen"

### 2. PANEL SYNC (CRITICAL)
**Current:** showLabel/hideLabel called in onUpdate loop (line 3054-3078)
**Problem:** Could skip on fast scroll
**Protocol Violation:** "Panels and environment are NOT synced"

### 3. MULTIPLE PANELS BUG (HIGH)
**Current:** No explicit guard against multiple panels active
**Problem:** Fast scrolling could show multiple cards
**Protocol Violation:** "Only ONE panel visible. Always."

### 4. REVERSE SCROLL (CRITICAL)
**Current:** Uses el.active flag and GSAP tweens
**Problem:** State persists, doesn't rebuild from scroll position
**Protocol Violation:** "Must be 100% reversible and deterministic"

### 5. PANEL POSITIONING (MEDIUM)
**Current:** 3D cards positioned at anatomy point Y (cardMeshes3D)
**Issue:** Not side-anchored as per Igloo spec
**Note:** 3D cards are separate from 2D labels - need clarity

---

## ANALYSIS: TWO PANEL SYSTEMS

### System A: 2D Labels (els[])
- DOM overlay elements
- Lines 2274-2313 (buildLabels)
- Positioned via CSS
- Currently HIDDEN via `display: none` in CSS (line 752-755, 783, 794)

### System B: 3D Cards (cardMeshes3D)
- Three.js mesh planes with canvas textures
- Lines 2315-2400 (buildAnatomyCards3D)
- World-space positioned
- Currently ACTIVE

**Decision:** Protocol applies to 3D cards (System B) - these are the visible "panels"

---

## FIX IMPLEMENTATION PLAN

### PHASE 1: IGLOO PANEL SIZING (PROTOCOL 1 - SECTION 2)

**Target:** 3D card canvas dimensions
**Current:** TAB_W = 200px, calculated height

**Fix:**
```javascript
// Line 2271 - update constants
const TAB_W = Math.min(200, window.innerWidth * 0.28); // max 28vw
const TAB_MAX_WIDTH = window.innerWidth * 0.28;
```

**Canvas sizing in buildAnatomyCards3D (line 2330+):**
```javascript
const CW = Math.min(512, window.innerWidth * 0.28 * 2); // 2x for retina
const titleSize = Math.max(32, CW * 0.074); // adaptive
```

---

### PHASE 2: SINGLE SOURCE OF TRUTH (PROTOCOL 3 - SECTION 12.1)

**Problem:** Multiple state sources
- `scrollProg` (line 3010)
- `envIdx` (line 3018)
- `el.active` flags
- `activeAnatomyId` (line 2503)

**Fix:** Derive everything from scroll progress

```javascript
// Single computation per frame
const segment = Math.floor(scrollProg * N);
const localProgress = (scrollProg * N) - segment;

// Reset all states
hideAllLabels(); // wipe slate

// Apply current segment only
if (segment >= 0 && segment < N) {
  const currentAnatomy = anatomy[segment];
  showLabel(currentAnatomy.id);
  setActivePDot(segment);
}
```

---

### PHASE 3: ELIMINATE STATE MEMORY (PROTOCOL 3 - SECTION 12.2)

**Current showLabel issue:**
```javascript
if (!el || el.active) return; // ← MEMORY
```

**Fix:** Remove early returns based on state
```javascript
function showLabel(id) {
  const el = els[id];
  if (!el || !heartReady) return;

  // NO el.active check - always rebuild
  el.active = true; // track for this frame only

  // Set states directly (idempotent)
  el.dot.style.opacity = '1';

  const c3d = cardMeshes3D[id];
  if (c3d) {
    c3d.mesh.visible = true;
    // Use direct assignment, not tweens that remember direction
    c3d.mat.opacity = 0.96;
    c3d.mesh.position.y = c3d.baseY;
  }
}
```

---

### PHASE 4: BIDIRECTIONAL SCROLL (PROTOCOL 3 - SECTION 12.5)

**Replace GSAP timeline with progress-driven:**

```javascript
onUpdate(self) {
  const progress = Math.max(0, Math.min(1, self.progress)); // clamp
  const segment = Math.floor(progress * N);

  // Works forwards AND backwards
  updateSceneFromProgress(progress, segment);
}

function updateSceneFromProgress(progress, segment) {
  // Reset phase
  resetAllPanels();
  resetEnvironment();

  // Rebuild phase
  if (segment >= 0 && segment < N) {
    applySegment(segment, progress);
  }
}
```

---

### PHASE 5: ENVIRONMENT SYNC (PROTOCOL 1 - SECTION 7)

**Current:** Environment updates in same onUpdate (✓ GOOD)
**Fix:** Ensure segment transitions are atomic

```javascript
// Line 3041 - segment boundary detection
if (envIdx !== lastEnvIdx) {
  // Update environment
  // Update panel
  // Update HUD
  // ALL TOGETHER
  lastEnvIdx = envIdx;
}
```

---

### PHASE 6: ANTI-SKIP GUARD (PROTOCOL 1 - SECTION 4)

**Add to onEnter:**
```javascript
onEnter() {
  inSpiral = true;
  currentSegment = 0; // ← FORCE
  lastEnvIdx = -1;

  // Immediately show first panel
  showLabel(anatomy[0].id);
  setActivePDot(0);

  // Set environment
  const [r, g, b, a] = anatomy[0].env.overlayRgba;
  envOverlay.style.backgroundColor = `rgba(${r},${g},${b},${a})`;
}
```

---

### PHASE 7: GLOBAL BUG SWEEP (PROTOCOL 2)

**Check all similar patterns:**
1. Facts section - does it skip? (line 3243+)
2. Finale section - reverse behavior? (line 3359+)
3. Hero section - camera jumps? (line 2990+)
4. All ScrollTrigger.create blocks - consistent guards?

---

## IMPLEMENTATION ORDER

1. ✅ Fix panel canvas sizing (28vw max)
2. ✅ Add progress clamping
3. ✅ Remove el.active early returns
4. ✅ Add segment locking system
5. ✅ Add onEnter first-panel guard
6. ✅ Test forward scroll (all segments)
7. ✅ Test backward scroll (reverse order)
8. ✅ Test fast scroll (no skips)
9. ✅ Global audit (all scroll sections)
10. ✅ Stress test (rapid oscillation)

---

## SUCCESS CRITERIA

- [ ] Panels are ≤28vw wide on all screens
- [ ] Only ONE panel visible at any time
- [ ] Forward scroll: smooth segment progression
- [ ] Backward scroll: exact reverse (C→B→A)
- [ ] Fast scroll: no skipped segments
- [ ] Mid-entry: correct segment shows
- [ ] Environment perfectly synced with panel
- [ ] No flicker, no pop, no desync
- [ ] All three protocols: PASSING

---

**Status:** READY FOR IMPLEMENTATION
