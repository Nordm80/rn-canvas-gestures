---
name: rn-canvas-gestures
description: Use when building or debugging interactive canvas / image-editor UIs in React Native that need pinch + pan + zoom of a background plus draggable / resizable / rotatable items, especially when targeting iOS + Android + web from the same codebase. Triggers when user mentions Reanimated worklets, react-native-gesture-handler, Gesture.Pinch / Gesture.Pan / Gesture.Manual, Android pinch broken, ScrollView pinch, cross-platform canvas, image editor, floor plan editor, draggable markers, pinch-to-resize, drag-to-rotate, RNGH + Reanimated composition issues, "bg moves during item drag", or "setGestureState in non-worklet function" warnings.
---

# Cross-platform canvas gestures (RN + RNGH + Reanimated)

A battle-tested recipe for an interactive canvas with:

- Background image you can **pinch-zoom + pan**
- Items rendered on top that you can **drag, resize, rotate**
- All three: **iOS + Android + Web** from one codebase

It captures the non-obvious decisions that took many failure cycles to find. If you follow it, you get a working canvas in one pass. If you find existing code that violates these rules, fix it — the violations are the bugs.

## When this applies

- Targeting Expo SDK 56+ / RN 0.74+ with the New Architecture
- You want one engine for both floor-plan-style editors and photo-annotation-style editors
- The app must run on iOS, Android, **and** web (RN-Web)

## When this does NOT apply

- Read-only image viewer with no item interaction → just use `react-native-zoom-toolkit` or similar off-the-shelf
- You only target web → just use plain SVG + CSS transforms
- You only target iOS → ScrollView pinch is fine (this is exactly what fails on Android)

## The one paragraph

Use `Gesture.Simultaneous(itemManual, pinch, pan)` — never `Gesture.Race`. `itemManual` is `Gesture.Manual().runOnJS(true)` so its `onTouchesDown` runs on the JS thread (you need items + canvasSize there for hit-testing). `pinch` and `pan` are worklet-based. They are gated by a shared value `bgEnabled` that defaults to **false** and is only set to **true** by `itemManual`'s JS callback after it confirms the touch landed on background (hit-test miss). The opposite polarity ("itemActive defaults false, set to true when item claimed") looks symmetric but has a 16-32ms timing window where bg gestures sneak in before the JS callback runs — the bg-default-blocked polarity has no such window. On web, RNGH Pan is unreliable for mouse-drag — add DOM-level `mousedown`/`mousemove`/`mouseup` listeners on the container that hit-test items+handles in source space and pan bg via shared-value writes when the click misses everything. Same listener owns `wheel` for cursor-anchored zoom.

## Architecture decisions table

| Decision | Choose | Avoid |
|---|---|---|
| Background pinch | `Gesture.Pinch` (RNGH worklet) | `ScrollView` `maximumZoomScale` (iOS-only, breaks Android pinch on RN 0.74+) |
| Background pan | `Gesture.Pan` (RNGH worklet) | Native `ScrollView` scrolling |
| Item drag/select/resize | `Gesture.Manual().runOnJS(true)` | Pure worklet (needs items as shared values — heavy refactor) |
| Composition | `Gesture.Simultaneous(...)` + shared-value gate | `Gesture.Race` (timing is unreliable with runOnJS); `requireExternalGestureToFail` (does not propagate fail correctly with runOnJS) |
| BG enable/disable | One shared value defaulting to `false` | Defaulting to `true` (16-32ms bleed window on item touch) |
| Resize math under rotation | Pixel space throughout | Normalized space (rotation in normalized space distorts on non-square canvases) |
| Hit-test under bg transform | Source space (invert bg transform first) | World space (hit-test misses when bg is panned/zoomed) |
| Web mouse drag for bg pan | DOM-level listener with hit-test routing | Relying on RNGH Pan (minDistance + runOnJS race makes it unreliable on web) |
| Web wheel zoom | DOM-level listener with `preventDefault` | RNGH Pinch (mouse can't trigger it; trackpad pinch is intercepted by browser zoom without preventDefault) |
| 2-finger transform | Scale + rotate + translate around item center, capture midpoint + angle + dist at touch-down | Pinch-only (no rotation; doesn't match common "manipulator" UX) |

## 1. Dependency setup

Reanimated 4 + worklets are often already in `node_modules` transitively via Expo / RNGH / Expo Router. They just need to be **declared** in `package.json` so they don't disappear on `npm prune` or new clones, and the babel plugin needs to be registered.

```bash
# Check first — these may already be transitively installed
find node_modules/react-native-reanimated -maxdepth 2 -name package.json 2>/dev/null | head -3
find node_modules/react-native-worklets -maxdepth 2 -name package.json 2>/dev/null | head -3

# If present transitively, add as explicit deps (versions should match transitive)
# Adjust versions to match what's already installed:
npm pkg set dependencies.react-native-reanimated="~4.3.1"
npm pkg set dependencies.react-native-worklets="~0.8.3"
npm install
```

Add the worklets plugin to `babel.config.js` — **must be last** in the plugins list:

```js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
      'nativewind/babel',
    ],
    plugins: [
      'react-native-worklets/plugin', // MUST be last
    ],
  };
};
```

Reanimated 4 requires the New Architecture, which is the default in Expo SDK 56+. No `newArchEnabled` flag needed unless you previously turned it off.

## 2. The gesture composition (canonical pattern)

```typescript
// gestures.ts
import { Gesture } from 'react-native-gesture-handler';
import { useSharedValue } from 'react-native-reanimated';
import type { SharedValue } from 'react-native-reanimated';

export function useCanvasGestures({
  bgScale,   // SharedValue<number> — current bg zoom
  bgTx,      // SharedValue<number> — current bg translate X
  bgTy,      // SharedValue<number> — current bg translate Y
  itemDrag,  // JS-thread drag state machine (see below)
}: BuildArgs) {
  const startScale = useSharedValue(1);
  const startTx = useSharedValue(0);
  const startTy = useSharedValue(0);

  // The gate. Default false = bg-blocked.
  const bgEnabled = useSharedValue(false);

  // Pass bgEnabled into itemDrag so it can flip it.
  // ...wire itemDrag with bgEnabled here...

  const itemManual = useMemo(() =>
    Gesture.Manual()
      .runOnJS(true)
      .onTouchesDown((e, state) => {
        const n = e.numberOfTouches;
        const t1 = e.allTouches[0];
        if (!t1) { state.fail(); return; }
        const t2 = n >= 2 ? e.allTouches[1] : undefined;
        const claimed = itemDrag.onTouchDown(
          { x: t1.x, y: t1.y }, n,
          t2 ? { x: t2.x, y: t2.y } : undefined,
        );
        if (claimed) state.activate();
        else state.fail();
      })
      .onTouchesMove(e => {
        const n = e.numberOfTouches;
        const t1 = e.allTouches[0];
        if (!t1) return;  // guard — RNGH fires this even mid-cancel
        const t2 = n >= 2 ? e.allTouches[1] : undefined;
        itemDrag.onTouchMove(
          { x: t1.x, y: t1.y }, n,
          t2 ? { x: t2.x, y: t2.y } : undefined,
        );
      })
      .onTouchesUp((_e, state) => { itemDrag.onTouchEnd(); state.end(); })
      .onTouchesCancelled((_e, state) => { itemDrag.onTouchEnd(); state.end(); }),
    [itemDrag]);

  const pinch = useMemo(() =>
    Gesture.Pinch()
      .onStart(() => { 'worklet'; startScale.value = bgScale.value; })
      .onUpdate(e => {
        'worklet';
        if (!bgEnabled.value) return;  // the gate
        bgScale.value = Math.max(MIN, Math.min(MAX, startScale.value * e.scale));
      })
      .onEnd(() => {
        'worklet';
        if (bgScale.value <= MIN + 0.001) {
          bgScale.value = MIN; bgTx.value = 0; bgTy.value = 0;
        }
      }), []);

  const pan = useMemo(() =>
    Gesture.Pan()
      .minPointers(1).maxPointers(2)
      .onStart(() => {
        'worklet';
        startTx.value = bgTx.value; startTy.value = bgTy.value;
      })
      .onUpdate(e => {
        'worklet';
        if (!bgEnabled.value) return;  // the gate
        bgTx.value = startTx.value + e.translationX;
        bgTy.value = startTy.value + e.translationY;
      }), []);

  return Gesture.Simultaneous(itemManual, pinch, pan);
}
```

## 3. The bgEnabled gate (the key insight)

```typescript
// itemDrag.ts — JS-thread drag state machine
// Inside onTouchDown:

// Item claim path (body, handle, or pinch-resize):
dragRef.current = { ...claim };
bgEnabled.value = false;   // explicit, redundant but safe
return true;

// Bg miss path:
onSelect(null);
bgEnabled.value = true;    // unlock bg gestures
return false;

// In onTouchEnd:
dragRef.current = emptyDrag;
bgEnabled.value = false;   // reset for next touch
```

**Why default false:**

```
Default TRUE (wrong):
  touch_down → Pinch.onUpdate fires (bgEnabled is true) → bg moves a few
  pixels → JS callback finally runs → sets bgEnabled false → bg stops.
  Result: visible bg "twitch" when tapping items, especially with close
  fingers (high scale-change-per-pixel makes it dramatic).

Default FALSE (right):
  touch_down → Pinch.onUpdate fires (bgEnabled is false) → returns
  immediately → no bg movement → JS callback runs → if miss, sets
  bgEnabled true and bg gestures unlock normally.
  Cost: a ~16-32ms delay before legit bg gestures can start. Imperceptible.
```

## 4. Source-space hit-testing under bg transform

When the background is panned/zoomed, items are RENDERED at their source positions (inside an Animated.View that has the bg transform applied). But touch coordinates from RNGH arrive in **world / view space** (untransformed). Hit-test must invert the transform first.

```typescript
function toSourcePoint(raw, canvasSize, bgScale, bgTx, bgTy) {
  const cx = canvasSize.width / 2;
  const cy = canvasSize.height / 2;
  const s = bgScale || 1;
  // Transform-origin is the View's center for default RN transforms
  return {
    x: (raw.x - cx - bgTx) / s + cx,
    y: (raw.y - cy - bgTy) / s + cy,
  };
}

// Then hit-test items against the source-space point.
// Drag deltas: also divide by scale, so dragging 100px on screen at
// scale=2 moves the item 50px in source space.
```

This applies equally on web — the DOM mouse-event handlers must also convert to source space before hit-testing.

## 5. Web: DOM-level wheel + mouse-drag listeners

RNGH Pan on web with mouse-drag is unreliable: `minDistance` plus the runOnJS roundtrip on Manual creates a window where Pan doesn't activate cleanly. Skip RNGH for web bg gestures and do it directly in DOM.

```typescript
// Inside ImageCanvas.tsx, after mounting the container View:
useEffect(() => {
  if (Platform.OS !== 'web') return;
  const el = containerRef.current as unknown as HTMLElement | null;
  if (!el) return;

  const hitsItemOrHandle = (relX, relY) => {
    // Convert to source space, hit-test items + selected item's handles
    // (handles can sit outside body — e.g. rotation handle 30px above top)
    // ...
  };

  const onWheel = (e: WheelEvent) => {
    e.preventDefault();   // stops Mac trackpad-pinch zooming the page
    // Cursor-anchored zoom: keep what's under cursor stable
    const rect = el.getBoundingClientRect();
    const cursorX = e.clientX - rect.left;
    const cursorY = e.clientY - rect.top;
    const centerX = rect.width / 2;
    const centerY = rect.height / 2;
    const oldScale = bgScale.value;
    const zoomFactor = Math.exp(-e.deltaY * 0.01);
    const newScale = clamp(oldScale * zoomFactor, MIN, MAX);
    if (newScale === oldScale) return;
    const ratio = newScale / oldScale;
    const dx = cursorX - centerX;
    const dy = cursorY - centerY;
    bgScale.value = newScale;
    bgTx.value = dx - (dx - bgTx.value) * ratio;
    bgTy.value = dy - (dy - bgTy.value) * ratio;
  };

  let panning = false, lastX = 0, lastY = 0;
  const onMouseDown = (e: MouseEvent) => {
    if (e.button !== 0) return;
    const rect = el.getBoundingClientRect();
    if (hitsItemOrHandle(e.clientX - rect.left, e.clientY - rect.top)) return; // let RNGH handle
    panning = true; lastX = e.clientX; lastY = e.clientY;
    el.style.cursor = 'grabbing';
    e.preventDefault();
  };
  const onMouseMove = (e: MouseEvent) => {
    if (!panning) return;
    bgTx.value += e.clientX - lastX;
    bgTy.value += e.clientY - lastY;
    lastX = e.clientX; lastY = e.clientY;
  };
  const onMouseUp = () => { panning = false; el.style.cursor = 'grab'; };

  el.style.cursor = 'grab';
  el.addEventListener('wheel', onWheel, { passive: false });
  el.addEventListener('mousedown', onMouseDown);
  document.addEventListener('mousemove', onMouseMove);
  document.addEventListener('mouseup', onMouseUp);
  return () => {
    el.removeEventListener('wheel', onWheel);
    el.removeEventListener('mousedown', onMouseDown);
    document.removeEventListener('mousemove', onMouseMove);
    document.removeEventListener('mouseup', onMouseUp);
  };
}, [bgScale, bgTx, bgTy]);
```

Wheel handler intentionally ignores `ctrlKey` — all wheel events become zoom, since mouse-drag already covers pan. Mac trackpad pinch sends `wheel` with `ctrlKey=true` and small deltaY; ordinary mouse wheel sends large deltaY; the same `exp(-deltaY * 0.01)` curve handles both.

## 6. Rotation math: always in pixel space

If your canvas can be non-square (almost any phone screen), rotation math in normalized coordinates **distorts**. A 90° rotation of normalized `(0.1, 0)` becomes `(0, 0.1)` — but in pixels on an 800×1245 canvas that's `(0, 124.5px)` instead of the correct `(0, 80px)`.

```typescript
// applyHandleDrag for resize handles under rotation
const rotation = item.rotation || 0;
const radInv = -rotation * Math.PI / 180;
const localDelta = rotateLocal(delta.dx, delta.dy, radInv);  // px → px

const hwPx = (item.w * canvasSize.width) / 2;
const hhPx = (item.h * canvasSize.height) / 2;
let leftL = -hwPx, rightL = hwPx, topL = -hhPx, bottomL = hhPx;

// Apply local delta and clamp to keep opposite edge fixed at min size:
const MIN_PX = Math.min(canvasSize.width, canvasSize.height) * 0.02;
if (handleId === 'nw') {
  leftL = Math.min(rightL - MIN_PX, leftL + localDelta.x);
  topL = Math.min(bottomL - MIN_PX, topL + localDelta.y);
}
// ...repeat for ne / se / sw with appropriate clamps

// Center shift = midpoint of new bounds (in local px)
const centerLocalPx = { x: (leftL + rightL) / 2, y: (topL + bottomL) / 2 };
// Rotate back to world px:
const radFwd = rotation * Math.PI / 180;
const worldShiftPx = rotateLocal(centerLocalPx.x, centerLocalPx.y, radFwd);

// Convert to normalized only at the boundary:
return {
  ...item,
  cx: clamp01(item.cx + worldShiftPx.x / canvasSize.width),
  cy: clamp01(item.cy + worldShiftPx.y / canvasSize.height),
  w: (rightL - leftL) / canvasSize.width,
  h: (bottomL - topL) / canvasSize.height,
};
```

The clamp-at-delta-application step (`leftL = Math.min(rightL - MIN_PX, …)`) is what keeps the opposite corner fixed when the user drags past minimum size. Clamping `w`/`h` only at the end leaves the center drifting.

## 7. 2-finger pinch = scale + rotate + translate

Capture at touch-down:

```typescript
const midpoint = { x: (t1.x + t2.x) / 2, y: (t1.y + t2.y) / 2 };
const startDist = Math.max(1, Math.hypot(t2.x - t1.x, t2.y - t1.y));
const startAngle = Math.atan2(t2.y - t1.y, t2.x - t1.x);
```

On move:

```typescript
const dist = Math.hypot(t2.x - t1.x, t2.y - t1.y);
const scaleFactor = dist / startDist;
const rotationDelta = Math.atan2(t2.y - t1.y, t2.x - t1.x) - startAngle;
const centerDelta = {
  dx: (t1.x + t2.x) / 2 - midpoint.x,
  dy: (t1.y + t2.y) / 2 - midpoint.y,
};
// Apply to item: scale w/h, add rotation, translate cx/cy by centerDelta.
// Rotation is around the item's own center (not the gesture midpoint).
```

## 8. Critical gotchas

- **Worklets cannot read `useRef().current`.** They capture stale values at worklet-init time. Mirror via `useSharedValue` or pass through the gesture's event object. *Symptom:* worklet silently uses zero / initial values, gesture-pipeline appears broken downstream.
- **Reanimated 4 warning `setGestureState in non-worklet function`** is **not cosmetic** — fires because `Manual.runOnJS(true)` callbacks call `state.activate()/fail()` from the JS thread, and Reanimated 4 expects them on the UI thread. The JS↔UI marshalling window (~8ms idle, ~60-100ms under GC / slow Android / accessibility tools) means `state.activate()` can land *after* the gesture has already advanced natively (touch lifted, second finger arrived, another gesture cancelled it). Symptoms at scale: stuck Manual gestures, lost taps, bg panning under an item drag, `onTouchesUp` firing on a FAILED gesture. Several patterns in this recipe — the default-false `bgEnabled` gate, `Pan.minDistance` synced to Manual's defer threshold, the `Simultaneous`-with-gate-instead-of-`Race` choice, and likely the freehand `onTouchEnd` guard — exist specifically to compensate for this thread-hop. **Acceptable for spikes and single-device dev testing. Not acceptable as a long-term state for production apps at scale** — schedule a worklet-ize-Manual pass (mirror `items` + `canvasSize` + selection as shared values, mark every `ItemType.hitTest` `'worklet'`, replace the registry Map with a worklet-callable dispatch, drop `runOnJS(true)`). See §11 for the upgrade path. **Don't suppress with `LogBox.ignoreLogs`** — it hides a signal you need for debugging the latent races.
- **`Gesture.Race` is unreliable with `runOnJS(true)`.** The JS-thread fail() doesn't propagate fast enough to cancel the worklet-thread Pinch/Pan winners. Use `Simultaneous` + shared-value gate instead.
- **`requireExternalGestureToFail(itemManual)` also unreliable with runOnJS.** Pinch/Pan stay blocked forever because Manual's fail-state on JS thread doesn't propagate to the dependent gestures' worklet thread.
- **Android `ScrollView.maximumZoomScale` is iOS-only.** If existing code uses ScrollView for bg pinch, that's why Android is broken. Rip it out, use `Gesture.Pinch`.
- **`Gesture.Pinch` on web requires a trackpad** (or `ctrlKey + wheel`). A plain mouse can't trigger it. The DOM-level wheel listener fills the gap and also intercepts Mac trackpad-pinch from zooming the whole page.
- **`pointerEvents="none"` on Background + ItemLayer is correct.** Touches must reach the parent GestureDetector. If you drop this, RNGH stops receiving events.
- **`onTouchesMove` can fire with `e.allTouches[0]` undefined** (mid-cancel sequences). Always guard `if (!t1) return;`.
- **`e.target === el` is unreliable on RN-Web** for distinguishing bg-touch from item-touch. Use coordinate hit-testing in source space instead.
- **2-finger touch-down may arrive as a single `onTouchesDown(n=2)` event** (fingers within one frame) OR sequentially as `n=1` then `n=2`. Handle the `n>=2` case **first** in your hit-test logic, before the 1-finger fallback. Hit-test BOTH `t1` AND `t2` (`hit(t1) || hit(t2)`) so close-finger pinch on an item is recognized.
- **When the user has a selection and lands 2 fingers OUTSIDE the selected item, deselect** (the user clearly isn't operating on it). Stale `selectedIdRef` from rapid sequential touch-downs can cause "phantom resize" of a just-deselected item if you don't handle this.
- **Mouse-drag on web for bg-pan must call `e.preventDefault()`** in mousedown — otherwise browser text-selection drag fires alongside and ruins the UX.
- **Reset `bgEnabled = false` in `onTouchEnd`**, not just at gesture composition init. Each new touch sequence must start in the gated state.
- **Never call `state.fail()` / `state.end()` mid-gesture in `Manual` handlers.** Only call them on the terminal touch-up (when all fingers have lifted, `remaining.length === 0`). Calling `state.fail()` from `onTouchesDown` (e.g. a priority-3 "bg pinch, hand off to Pinch/Pan" branch) or `onTouchesMove` (e.g. deferred-→-bg-pan commit) desyncs RNGH's internal `trackedTouchCount` — touches are still active on the screen but Manual stopped counting them. *Symptoms:* warning `Ended a touch event which was not counted in trackedTouchCount` on every gesture, AND RNGH suppresses further event delivery to the failed Manual so any shared values Manual sets later in the gesture (like `bgEnabled`) appear stuck at their pre-fail value when read by Pinch/Pan worklets. **Fix:** for "I don't want to claim, let Pinch/Pan take over" cases return a `'defer'` / `'continue'` decision and leave Manual in BEGAN; clear `dragState` synchronously so subsequent `onTouchesMove` no-ops via your existing `if (!drag.itemId)` guard. Clean up and call `state.fail()` only when `onTouchesUp` fires with `remaining.length === 0`. With the old `runOnJS(true)` shape this rule was hidden because mid-gesture fails were silently dropped (see the `setGestureState in non-worklet function` bullet above); it only surfaces after the worklet-ize pass.

## 9. Verification checklist

Per platform, after wiring everything up:

**iOS dev-client**:
- [ ] 2-finger pinch on bg zooms
- [ ] 1-finger drag on bg pans (when scale > 1)
- [ ] Tap on item selects (pink/visible selection); tap outside deselects
- [ ] 1-finger drag on item body moves it
- [ ] 1-finger drag on a corner handle resizes (opposite corner stays fixed)
- [ ] 2-finger gesture on selected item: scale + rotate + translate around item center, bg stays still
- [ ] Bg doesn't twitch when starting an item drag with close fingers

**Android dev-client**: same checklist. *Pinch breaks here first if ScrollView is in the path.*

**Web (Chrome with mouse only)**:
- [ ] Cursor shows `grab` over canvas, `grabbing` while panning
- [ ] Scroll wheel zooms (cursor-anchored)
- [ ] Mouse-drag on bg pans
- [ ] Click on item selects
- [ ] Click+drag on item moves it; drag on handle resizes
- [ ] Click+drag on rotation handle rotates around item center

**Web (Mac trackpad)**:
- [ ] Trackpad pinch zooms (does NOT zoom the browser page)
- [ ] Two-finger swipe zooms (same handler) — if you want it to pan instead, add the `ctrlKey` discriminator from §5

## 10. When you find existing code violating this recipe

Common refactors to apply:

| Existing pattern | Fix |
|---|---|
| `<ScrollView maximumZoomScale=…>` wrapping the canvas | Replace with `<GestureDetector gesture={...}>` and add `Gesture.Pinch`/`Gesture.Pan` |
| `Gesture.Race(itemManual, ...)` | Switch to `Gesture.Simultaneous(...)` + add `bgEnabled` shared-value gate |
| Hit-test using raw `touch.x` against item source positions | Add source-space transformation that inverts bg transform |
| Resize math operating on normalized `cx/cy/w/h` directly | Move math to pixel space; convert to normalized only at the return boundary |
| `if (scale <= MIN) return` in Pan worklet | Replace with clamping at translation level (or just allow free pan) |
| Reading `canvasSizeRef.current.width` inside a worklet | Either mirror as a shared value, or remove the use of that dimension from the worklet |

If you see ALL of these in one file, you're looking at pre-2025 RN gesture code. Rewrite the gesture file, leave the item rendering alone.

## 11. Production-scale upgrade path: worklet-ize `Gesture.Manual`

The recipe in §1-§10 is a working spike. Several of its defensive patterns exist *because* `Gesture.Manual().runOnJS(true)` puts gesture state transitions on the JS thread — see §8 on the `setGestureState in non-worklet function` warning. For an app shipping to thousands of users on diverse devices, schedule a worklet-ize-Manual pass to eliminate the thread-hop and the latent race class it creates.

### What "worklet-ize Manual" means

Drop `runOnJS(true)` and make `Manual`'s `onTouchesDown / onTouchesMove / onTouchesUp` callbacks worklet-compatible. Every value they read/write must be:

- A shared value (`items`, `canvasSize`, `selectedId`, `mode` → mirrored from React state via `useSharedValue` and dual-written on every mutator),
- A pure function annotated `'worklet'` (every `ItemType.hitTest`, `getEditableHandles` — they're already pure-math but need the directive + a closure-audit),
- Or wrapped in `runOnJS(...)` for the one-shot React-thread side-effects that must remain (e.g. `onSelect`, `setItemsLive`, `addItem`, `onBeginEdit`).

The gesture state transitions (`state.activate()`, `state.fail()`) then happen synchronously on the UI thread next to the native gesture lifecycle; React-thread side-effects fire async without blocking the gesture.

### What can be removed once Manual is worklet-based

| Defensive pattern | Decision after worklet-ize |
|---|---|
| `bgEnabled` default-false gate | Simplify — pinch/pan worklets can read items + sel directly and decide inline. Likely still keep a single boolean but the default-false-then-unlock pattern goes away. |
| `Pan.minDistance(DEFER_COMMIT_RAW_PX)` synced to Manual's defer threshold | Remove — Pan can activate immediately, the gate evaluates synchronously. |
| `Gesture.Simultaneous(itemManual, pinch, pan)` + gate | Re-evaluate — `Gesture.Race` should now work since Manual's `state.fail()` propagates synchronously. Race is the cleaner architecture if it tests cleanly. |
| Freehand `onTouchEnd` guard for `state.fail()`-ed Manual still firing `onTouchesUp` | Test empirically. RNGH platform quirk may persist; if so, keep the guard. |

### Concrete risks

- **Hit-test closures.** Audit every `ItemType.hitTest` to confirm it reads only its args. Capturing module state / React refs in a worklet silently fails.
- **`getItemType` registry Map → worklet dispatch.** Map access from worklets is unsupported. Replace with a worklet-callable switch on `kind`.
- **Dual-writing items.** Every Provider mutator (`addItem`, `updateItem`, `removeItem`, `undo`, `setItemsLive`, `trashAll`) must write through both React state and the shared-value mirror. One missed write = silently desynced hit-testing.
- **Loss of React debuggability.** Worklet logic is harder to step through than JS-thread code. Document this in the PR description so future contributors aren't lost.

### Verification

Reproduce the original race by deliberately stalling the JS thread for 80ms (`const end = Date.now() + 80; while (Date.now() < end) {}`) right before a touch sequence and verify gesture state stays coherent. Run device verification on at least one slow Android (≥ 3 years old) — that's where the latent races show up first.

### What does NOT change

Source-space hit-testing (§4), rotation math (§6), 2-finger pinch math (§7), web DOM listeners (§5) — all unchanged by worklet-ize. The web path bypasses RNGH already (web JS is single-threaded; no thread-hop) so web has no upgrade work.

### Empirical: what survived the pass (2026-05-28, Expo SDK 56)

Confirmed on iOS + Android dev-client after a clean worklet-ize-Manual pass:

| Defensive pattern | Outcome |
|---|---|
| `bgEnabled` SV gate | **Kept.** Could be replaced by inline `dragSv` + `selectedIdSv` + `modeSv` reads in Pan/Pinch `onUpdate` worklets, but no behaviour win — `bgEnabled` is clearer as an explicit synchronous gate that Manual handlers set as part of their state-machine decision. |
| `Pan.minDistance(DEFER_COMMIT_RAW_PX)` synced to Manual's defer threshold | **Kept.** Without it, Pan activates during the pre-threshold deferred-tap window and shifts the bg before itemDrag commits. Safe removal needs a careful Pan/Manual handoff protocol; not worth the risk. |
| `Gesture.Simultaneous` instead of `Gesture.Race` | **Kept.** Race is plausibly cleaner now that `state.fail()` is synchronous, but needs operator device verification across all four gestures before the swap. The inline `bgEnabled` gate is correct on its own. |
| Freehand `onTouchEnd` `if (!st.active) return` guard | **Kept.** Confirmed still an RNGH platform quirk (failed Manual can still fire `onTouchesUp`), not a thread-hop artifact. |

**The single critical new lesson:** see the `state.fail() / state.end()` mid-gesture bullet in §8. It only surfaces after the pass — the OLD `runOnJS(true)` shape silently dropped the mid-gesture state transitions, which made the pattern accidentally "work". With proper UI-thread state transitions, calling `state.fail()` from `onTouchesDown` triggers `Ended a touch event which was not counted in trackedTouchCount` AND breaks the very SV-gate propagation the worklet-ize was supposed to fix.

