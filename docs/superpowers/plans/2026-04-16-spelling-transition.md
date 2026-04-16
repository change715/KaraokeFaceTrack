# Spelling Transition Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rework the phonetic→final-word collapse so long words (e.g. `Haaallelujah`) visibly close up into their final spelling (`Hallelujah`) as the word traverses a specific stretch of the spline, driven by position rather than per-letter age.

**Architecture:** One pre-pass per frame computes a shared `segSpellProgress[segIdx]` from the leader's natural spline-t, plus a shared `segMidKeptIndex[segIdx]`. Each kept letter's age gets a center-anchored additive boost (negative for leaders, positive for trailers) proportional to its distance from the middle kept letter, the segment progress, and `fontSpeedScale`. A per-particle monotonic clamp (`_prevLinearT`) guarantees no letter's spline position ever steps backward. Extras inherit their preceding kept letter's boost and fade via the existing sequential gap-index mechanic, tuned to complete within the first half of the collapse window.

**Tech Stack:** Vanilla HTML5/JS in a single file (`index.html`); runtime params from `params.json`; visual verification via the `preview_*` MCP tools against a local Python HTTP server (already configured in `.claude/launch.json`).

---

## Context For The Engineer

**No test framework exists** in this project — no Jest, Vitest, Playwright, or similar. The app is a real-time WebGL/Canvas render loop that depends on a live webcam for face tracking, so the core spelling-transition behavior can only be verified visually by the user with a camera attached. You will not write automated tests for visual behavior. Instead, for each code change:
1. Use `preview_start` / `preview_console_logs` / `preview_eval` to verify the page still loads, all JS parses, and no `ReferenceError`/`TypeError` appears in the console.
2. Run targeted `preview_eval` snippets that inject test particles into the render context to verify math (where feasible).
3. After all tasks, ask the user to run it against their webcam and confirm the collapse behavior matches the spec.

**File touch list (the whole plan):**
- Modify: `index.html:2214-2229` — replace per-segment pre-pass
- Modify: `index.html:2252-2269` — replace kept-letter boost + extras boost formulas; add monotonic clamp
- Modify: `index.html:302` — bump `spellGrouping` and `spellFadeSpeed` defaults in `P_HARDCODED`
- Modify: `params.json` — bump saved `spellGrouping` and `spellFadeSpeed` values

**Reference:** The approved spec is at `docs/superpowers/specs/2026-04-16-spelling-transition-design.md`. Read it before starting.

---

### Task 1: Rewrite the per-segment pre-pass (leader-t anchor + midKeptIndex)

**Files:**
- Modify: `index.html:2214-2229`

- [ ] **Step 1: Read the current pre-pass**

Open `index.html` and read lines 2210-2230. The current code anchors progress to the last-spawned letter's `baseT`. You'll replace it with a leader-anchored computation plus a midKeptIndex map.

- [ ] **Step 2: Replace the pre-pass block**

Use the Edit tool to replace the block starting `const segLastSpawn = {};` through the closing `}` at line 2229 (the `segSpellProgress` computation). The new block computes both the segment progress (from the leader's natural spline-t) and the middle kept index (from `p.totalKept`).

Replace this:
```js
const segLastSpawn = {};
for (const p of particles) {
    if (p.isKept === undefined) continue;
    if (!(p.segIdx in segLastSpawn) || p.spawnTime > segLastSpawn[p.segIdx]) {
        segLastSpawn[p.segIdx] = p.spawnTime;
    }
}
const segSpellProgress = {};
for (const sIdx in segLastSpawn) {
    const ageOfLast = Math.max(0, nowSec - segLastSpawn[sIdx]);
    const lastLinearT = Math.min(ageOfLast / P.charLifetime, 1.0);
    const lastBaseT = easeOutQuad(lastLinearT);
    segSpellProgress[sIdx] = Math.max(0, Math.min(1,
        (lastBaseT - P.spellTransStart) / (P.spellTransEnd - P.spellTransStart)));
}
```

With this:
```js
// Progress is anchored to the LEADER's natural spline-t (position-based,
// not time-based). Mid-kept index is used below to center the collapse.
const segLeaderT = {};
const segMidKeptIndex = {};
for (const p of particles) {
    if (p.isKept !== true) continue;
    if (p.keptIndex === 0) {
        const pAge = nowSec - p.spawnTime;
        const pLinearT = Math.min((pAge * fontSpeedScale) / P.charLifetime, 1.0);
        segLeaderT[p.segIdx] = easeBlended(pLinearT, P.linearBlend);
    }
    if (!(p.segIdx in segMidKeptIndex)) {
        segMidKeptIndex[p.segIdx] = Math.floor(p.totalKept / 2);
    }
}
const segSpellProgress = {};
for (const sIdx in segLeaderT) {
    segSpellProgress[sIdx] = Math.max(0, Math.min(1,
        (segLeaderT[sIdx] - P.spellTransStart) / (P.spellTransEnd - P.spellTransStart)));
}
```

- [ ] **Step 3: Verify page loads**

Ensure the preview server is running (`preview_start` with `"static"`). Then reload and check console for errors:

Run:
```
preview_eval: window.location.reload()
preview_console_logs: pattern="error|ReferenceError|TypeError|Uncaught"
```

Expected: only `[error] NotAllowedError: Permission denied` (camera) and `[error] [object DOMException]` (camera). No JS errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Rewrite spelling pre-pass: anchor progress to leader's spline-t

The per-segment transition progress is now derived from the leader's
natural position on the spline rather than the last-spawned letter's
baseT. Also emits a segMidKeptIndex map used by the boost formula in
the next commit."
```

---

### Task 2: Replace kept-letter boost formula (delta-based, fontSpeedScale-corrected)

**Files:**
- Modify: `index.html:2252-2261`

- [ ] **Step 1: Read the current kept-letter boost**

Read lines 2252-2261 of `index.html`. The existing code uses `(p.extrasBefore / p.totalChars) * p.segDuration * P.spellGrouping`, which is missing the `fontSpeedScale` multiplier and under-collapses on large font sizes.

- [ ] **Step 2: Replace the kept-letter boost block**

Replace:
```js
// Kept letter: pull forward by its leading extras so the group
// bunches up into the final word as transition progresses.
if (p.isKept && spellTransProgress > 0 && p.extrasBefore > 0) {
    const ageBoost = spellTransProgress * (p.extrasBefore / p.totalChars) * p.segDuration * P.spellGrouping;
    linearT = Math.min((age * fontSpeedScale + ageBoost) / P.charLifetime, 1.0);
    if (spellTransProgress < 1 && linearT > P.fadeStart) {
        linearT = P.fadeStart;
    }
    t = easeBlended(linearT, P.linearBlend);
}
```

With:
```js
// Kept letter: center-anchored age boost. delta = keptIndex − midKeptIndex.
// Leaders (delta<0) slow; trailers (delta>0) speed up; middle stays natural.
// fontSpeedScale is required for the boost to actually land at the
// fontSpeedScale-accelerated linearT rate.
if (p.isKept && spellTransProgress > 0) {
    const midIdx = segMidKeptIndex[p.segIdx] || 0;
    const delta = p.keptIndex - midIdx;
    const spawnGapSec = p.segDuration / p.totalChars;
    const ageBoost = spellTransProgress * delta * spawnGapSec * fontSpeedScale * P.spellGrouping;
    linearT = Math.min(Math.max(0, (age * fontSpeedScale + ageBoost) / P.charLifetime), 1.0);
    if (spellTransProgress < 1 && linearT > P.fadeStart) {
        linearT = P.fadeStart;
    }
    t = easeBlended(linearT, P.linearBlend);
}
```

Notes:
- Dropped the `p.extrasBefore > 0` guard — middle kept letter now has `delta=0` and naturally contributes nothing, and we want leaders (with `extrasBefore=0` but `delta<0`) to slow.
- Added `Math.max(0, ...)` on `linearT` so negative-boost leaders can't drop below 0 if they get a strong early-frame boost.

- [ ] **Step 3: Verify page loads with no runtime errors**

```
preview_eval: window.location.reload()
preview_console_logs: pattern="error|ReferenceError|TypeError|NaN|Uncaught"
```

Expected: only the camera-permission errors.

- [ ] **Step 4: Smoke-test the formula with preview_eval**

The renderer and its closures aren't accessible via eval, but we can verify the math by inlining the formula. Paste this expression into `preview_eval` and check it returns the expected object:

```js
((() => {
  // Spec check: delta-based boost for a 10-kept, 12-char word at progress=1
  const totalChars = 12, totalKept = 10, segDuration = 2;
  const spellGrouping = 1.0, fontSpeedScale = 1.81;
  const midIdx = Math.floor(totalKept / 2); // 5
  const spawnGapSec = segDuration / totalChars; // 0.167
  const progress = 1.0;
  const boostFor = (k) => progress * (k - midIdx) * spawnGapSec * fontSpeedScale * spellGrouping;
  return {
    leader_k0: boostFor(0).toFixed(3),      // expect negative (~-1.510)
    mid_k5:    boostFor(5).toFixed(3),      // expect 0.000
    trailer_k9: boostFor(9).toFixed(3),     // expect positive (~+1.208)
  };
})())
```

Expected return: `{ leader_k0: "-1.510", mid_k5: "0.000", trailer_k9: "1.208" }`.

If the numbers don't match, the formula in the code is wrong.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Rewrite kept-letter boost: center-anchored delta × fontSpeedScale

Replaces the extrasBefore-based ageBoost with a delta-based formula
centered on the middle kept letter. Leaders (delta<0) get negative
boost (slow), trailers (delta>0) get positive boost (speed up), and
fontSpeedScale is now correctly included so the collapse lands at
the accelerated linearT rate instead of under-shooting by ~1.8× on
large font sizes."
```

---

### Task 3: Replace extras boost formula (inherit preceding kept letter's boost)

**Files:**
- Modify: `index.html:2263-2269`

- [ ] **Step 1: Read the current extras boost**

Read lines 2263-2269 of `index.html`. Current code gives extras a `prevKeptBoost` based on `p.prevKeptExtrasBefore`.

- [ ] **Step 2: Derive what extras should do**

With the new kept-letter formula, extras should move with their **preceding kept letter's** boost — that lets them "trail along" with the group as it collapses rather than lagging behind at phonetic positions. But we don't currently store the preceding kept letter's `keptIndex` on extras.

The cleanest fix is to compute the preceding kept letter's `delta` from data we already store:
- `p.prevKeptExtrasBefore` tells us `extrasBefore` of the preceding kept letter
- With the preceding kept letter's `charIndex` (deducible as `p.charIndex − 1 − (extras between them)`) we could derive its `keptIndex`, but that's fragile

Simpler: **store `prevKeptIndex` on the extra at charMap-build time**. Requires a small change in segment parsing.

- [ ] **Step 3: Add `prevKeptIndex` to extras in charMap**

Read around `index.html:1060-1070` (the third pass that assigns `prevKeptExtrasBefore`; the target line is `charMap[ci].prevKeptExtrasBefore = charMap[ki].extrasBefore;`). Modify it to also assign `prevKeptIndex`.

Replace:
```js
// Previous kept (ahead, speeding away)
for (let ki = ci - 1; ki >= 0; ki--) {
    if (charMap[ki].kept) {
        charMap[ci].prevKeptExtrasBefore = charMap[ki].extrasBefore;
        break;
    }
}
```

With:
```js
// Previous kept (ahead, speeding away)
for (let ki = ci - 1; ki >= 0; ki--) {
    if (charMap[ki].kept) {
        charMap[ci].prevKeptExtrasBefore = charMap[ki].extrasBefore;
        charMap[ci].prevKeptIndex = charMap[ki].keptIndex;
        break;
    }
}
```

- [ ] **Step 4: Pass `prevKeptIndex` onto the particle at spawn**

Read around `index.html:1910-1920` (the particle push; the target line is `prevKeptExtrasBefore: cm ? (cm.prevKeptExtrasBefore || 0) : 0,`). Add `prevKeptIndex` next to it.

Replace:
```js
prevKeptExtrasBefore: cm ? (cm.prevKeptExtrasBefore || 0) : 0,
```

With:
```js
prevKeptExtrasBefore: cm ? (cm.prevKeptExtrasBefore || 0) : 0,
prevKeptIndex: cm ? (cm.prevKeptIndex !== undefined ? cm.prevKeptIndex : -1) : -1,
```

The `-1` sentinel means "no preceding kept letter" (first gap of extras before any kept letter); we'll treat that as no boost.

- [ ] **Step 5: Replace the extras boost block in the render loop**

Replace:
```js
// Extras: follow their preceding kept char (ahead, speeding away)
// so they don't fall behind as the group advances.
if (p.isKept === false && p.prevKeptExtrasBefore > 0 && spellTransProgress > 0) {
    const prevKeptBoost = spellTransProgress * (p.prevKeptExtrasBefore / p.totalChars) * p.segDuration * P.spellGrouping;
    linearT = Math.min((age * fontSpeedScale + prevKeptBoost) / P.charLifetime, 1.0);
    t = easeBlended(linearT, P.linearBlend);
}
```

With:
```js
// Extras: inherit the preceding kept letter's center-anchored boost,
// so they ride with the group as it collapses rather than lagging at
// phonetic positions. Fades (below) handle their disappearance.
if (p.isKept === false && p.prevKeptIndex >= 0 && spellTransProgress > 0) {
    const midIdx = segMidKeptIndex[p.segIdx] || 0;
    const delta = p.prevKeptIndex - midIdx;
    const spawnGapSec = p.segDuration / p.totalChars;
    const prevKeptBoost = spellTransProgress * delta * spawnGapSec * fontSpeedScale * P.spellGrouping;
    linearT = Math.min(Math.max(0, (age * fontSpeedScale + prevKeptBoost) / P.charLifetime), 1.0);
    t = easeBlended(linearT, P.linearBlend);
}
```

- [ ] **Step 6: Verify page loads**

```
preview_eval: window.location.reload()
preview_console_logs: pattern="error|ReferenceError|TypeError|NaN|Uncaught"
```

Expected: only camera-permission errors.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Rewrite extras boost: inherit preceding kept letter's center-anchored boost

Extras now travel with their group by mirroring the preceding kept
letter's delta-based boost instead of using a separate prevKeptBoost
formula. Requires a new prevKeptIndex field on extra charMap entries
and a corresponding particle field, both wired up here."
```

---

### Task 4: Add monotonic-forward guard to kept letters

**Files:**
- Modify: `index.html:2252-2261` (kept-letter block, after boost calc)

- [ ] **Step 1: Understand why this is needed**

With center-anchoring, a leader's computed `linearT` at progress>0 is smaller than its natural linearT. Across frames, as `spellTransProgress` rises (leader advances along spline), the negative `ageBoost` magnitude grows. Under certain transition-window settings, the effective linearT could briefly decrease frame-to-frame even though natural linearT is monotonically increasing. The spec forbids any backward motion.

The fix: per-particle cache of the previous frame's effective `linearT`, and clamp the current frame to be `≥` that.

- [ ] **Step 2: Add the monotonic clamp inside the kept-letter block**

Edit the kept-letter block (the one modified in Task 2). After `linearT = Math.min(Math.max(0, ...), 1.0)` but before the fadeStart clamp and the `easeBlended` call, insert the monotonic clamp.

Find:
```js
if (p.isKept && spellTransProgress > 0) {
    const midIdx = segMidKeptIndex[p.segIdx] || 0;
    const delta = p.keptIndex - midIdx;
    const spawnGapSec = p.segDuration / p.totalChars;
    const ageBoost = spellTransProgress * delta * spawnGapSec * fontSpeedScale * P.spellGrouping;
    linearT = Math.min(Math.max(0, (age * fontSpeedScale + ageBoost) / P.charLifetime), 1.0);
    if (spellTransProgress < 1 && linearT > P.fadeStart) {
        linearT = P.fadeStart;
    }
    t = easeBlended(linearT, P.linearBlend);
}
```

Replace with:
```js
if (p.isKept && spellTransProgress > 0) {
    const midIdx = segMidKeptIndex[p.segIdx] || 0;
    const delta = p.keptIndex - midIdx;
    const spawnGapSec = p.segDuration / p.totalChars;
    const ageBoost = spellTransProgress * delta * spawnGapSec * fontSpeedScale * P.spellGrouping;
    linearT = Math.min(Math.max(0, (age * fontSpeedScale + ageBoost) / P.charLifetime), 1.0);
    // Monotonic-forward guard: never let a letter step backward on the spline.
    // First frame: p._prevLinearT is undefined → treat as 0.
    if (p._prevLinearT !== undefined && linearT < p._prevLinearT) {
        linearT = p._prevLinearT;
    }
    p._prevLinearT = linearT;
    if (spellTransProgress < 1 && linearT > P.fadeStart) {
        linearT = P.fadeStart;
    }
    t = easeBlended(linearT, P.linearBlend);
}
```

Note: the guard runs **before** the `fadeStart` clamp so the cached `_prevLinearT` reflects the un-clamped collapse position; the fade clamp is a display-time cap that shouldn't poison the monotonic cache.

- [ ] **Step 3: Consider whether extras need the same guard**

Extras inherit their preceding kept letter's boost, so if the preceding kept is monotonic, extras — which spawn later and have smaller natural age — will also track monotonically. They also fade out within the first half of progress, so any tiny reversal would be invisible. **Skip the guard on extras.** Keep them simple.

- [ ] **Step 4: Verify page loads**

```
preview_eval: window.location.reload()
preview_console_logs: pattern="error|ReferenceError|TypeError|NaN|Uncaught"
```

Expected: only camera-permission errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add monotonic-forward guard on kept-letter linearT

Caches the previous frame's effective linearT on each kept particle
and clamps the current frame to be ≥ it, so leaders can slow or
hover during collapse but can never step backward on the spline."
```

---

### Task 5: Bump `P_HARDCODED` defaults for new calibration

**Files:**
- Modify: `index.html:302`

- [ ] **Step 1: Update the two defaults**

Read `index.html:302` to confirm the current line:
```js
spellTransStart:0.18, spellTransEnd:0.44, spellFadeSpeed:3.80, spellGrouping:1.70,
```

Note: the HEAD baseline default was `spellGrouping:1.70`, `spellFadeSpeed:3.80`. Under the new formula `spellGrouping:1.0` produces tight one-spawn-gap collapse and `spellFadeSpeed:5.0` concentrates the extras fade into the first half of progress (per spec).

Replace:
```js
spellTransStart:0.18, spellTransEnd:0.44, spellFadeSpeed:3.80, spellGrouping:1.70,
```

With:
```js
spellTransStart:0.18, spellTransEnd:0.44, spellFadeSpeed:5.00, spellGrouping:1.00,
```

- [ ] **Step 2: Verify page loads**

```
preview_eval: window.location.reload()
preview_console_logs: pattern="error|ReferenceError|TypeError|Uncaught"
```

Expected: only camera-permission errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Bump P_HARDCODED spellGrouping=1.0 and spellFadeSpeed=5.0

New calibration for the delta×fontSpeedScale formula: spellGrouping=1.0
produces tight one-spawn-gap collapse; spellFadeSpeed=5.0 concentrates
extras fade into the first half of progress so the 'gap left behind'
look matches the reference video."
```

---

### Task 6: Update `params.json` saved defaults

**Files:**
- Modify: `params.json`

- [ ] **Step 1: Read the current saved values**

Read `params.json`. Locate `"spellGrouping"` and `"spellFadeSpeed"` (lines around 22-24 in the current file).

- [ ] **Step 2: Update both values**

Replace:
```json
    "spellGrouping": 0.8,
    "spellTransStart": 0.18,
    "spellTransEnd": 0.41,
    "spellFadeSpeed": 3.8,
```

With:
```json
    "spellGrouping": 1.0,
    "spellTransStart": 0.18,
    "spellTransEnd": 0.41,
    "spellFadeSpeed": 5.0,
```

- [ ] **Step 3: Verify params.json is still valid JSON**

Run `preview_eval`:
```js
fetch('/params.json?' + Date.now()).then(r => r.json()).then(p => ({
  spellGrouping: p.spellGrouping,
  spellFadeSpeed: p.spellFadeSpeed,
  spellTransStart: p.spellTransStart,
  spellTransEnd: p.spellTransEnd,
}))
```

Expected: `{ spellGrouping: 1, spellFadeSpeed: 5, spellTransStart: 0.18, spellTransEnd: 0.41 }`.

If it returns a reject/parse error, the JSON is malformed — check for a missing comma or quote.

- [ ] **Step 4: Commit**

```bash
git add params.json
git commit -m "Update saved spellGrouping=1.0 and spellFadeSpeed=5.0

Matches the new P_HARDCODED calibration for the delta-based
center-anchored collapse formula."
```

---

### Task 7: Visual verification with live webcam

**Files:** None modified.

This task is performed by the user. The agent cannot verify visual behavior without webcam access.

- [ ] **Step 1: Ask the user to test**

Tell the user:

> "Implementation complete. Please open the preview at http://localhost:8000 with your webcam connected and audio playing. Look for:
>
> 1. When a phonetic word like `Haaallelujah` spawns, all letters are initially visible in phonetic form.
> 2. As the word moves along the spline, the duplicate letters fade out in the first half of the collapse window, leaving visible gaps.
> 3. In the second half, the kept letters close up into the final spelling `Hallelujah` with tight kerning.
> 4. The leader never visibly steps backward on the spline.
> 5. The final word remains on screen long enough to be readable before fading out normally.
>
> If anything looks wrong, describe what you see and what you expected — especially: does the collapse complete before the word exits? Are the extras gone by halfway? Is there any visible reverse motion on leaders?"

- [ ] **Step 2: Iterate on tuning if needed**

If the user reports visual issues, the knobs available are:
- `spellGrouping` (slider): increase for tighter collapse, decrease for looser
- `spellFadeSpeed` (slider): increase to fade extras sooner, decrease to keep them around longer
- `spellTransStart` / `spellTransEnd` (sliders): shift or widen the collapse window along the spline

Collect their observations and tune. Do not add new code without a new spec iteration.

- [ ] **Step 3: No additional commit required at this stage**

Tuning happens in the live UI; values are persisted to `params.json` via the existing save mechanism when the user chooses.

---

## Self-Review

1. **Spec coverage:** Every spec goal is covered:
   - Position-driven progress → Task 1
   - Transitions as a unit → Task 1 (shared `segSpellProgress`)
   - Center-of-word anchor → Task 2 (delta from `midKeptIndex`)
   - Monotonic forward → Task 4
   - Extras fade first half → Task 5/6 (`spellFadeSpeed=5.0`)
   - Fade guard + displayChar swap unchanged from existing Design A code — no task needed

2. **Placeholder scan:** All code blocks are complete; no TBDs. Commands are exact.

3. **Type consistency:** `segMidKeptIndex` introduced in Task 1 and consumed in Tasks 2–4 — name matches. `prevKeptIndex` introduced in Task 3 and consumed in the same task — name matches. `_prevLinearT` is a single-task addition (Task 4).

4. **Edge case coverage:** Spec edge cases are handled by existing mechanics (the non-spelling-lyric case is protected by `p.isKept !== undefined` guards; no-leader case falls through to `segSpellProgress[sIdx] || 0`).
