# Spelling Transition Redesign

**Date:** 2026-04-16
**Status:** Approved for implementation
**Scope:** `index.html` render loop + `params.json` defaults

## Problem

Long lyrics like "Hallelujah" spawn as a phonetic stretch (`Haaallelujah`) so the mouth-burst reads visually before the line resolves. Current behavior: the per-letter time-based transition fires for each letter independently, so leaders finish transitioning and begin fading before trailing letters have even spawned. The result is that the phonetic spelling never visibly collapses into the final spelling on screen — the user only ever sees stretched phonetic or nothing.

Reference behavior we're targeting (karaoke video of "worth" stretching to "wooooorth" and back): as the word traverses a specific stretch of the spline, the extra phonetic letters fade out and the kept letters close up into tight final-word kerning. By the end of the collapse region, the final word is fully formed, and it then continues along the spline and fades normally.

## Goals

1. The collapse is **position-driven** on the spline, not age-driven
2. The word transitions **as a unit**, not per-letter — no leader fades out before trailers spawn
3. **Center of word** stays anchored: leaders slow, trailers speed up, both ends move inward
4. **Monotonic forward** motion — a letter's spline position never decreases frame-to-frame
5. Extras fade in the first half of the collapse window; kept letters close up in the second half (matches the "gap-left-behind" look in the reference)
6. After collapse completes, the final-spelling word rides the spline normally until it fades out at `fadeStart`

## Non-goals

- Changing how phonetic letters are authored in the JSON
- Reworking burst-from-mouth behavior
- Altering non-spelling lyric rendering (lines without a `charMap`)
- Adding new per-particle physics (springs, velocity state)

## Design

### Data flow per frame

```
1. Per-segment pre-pass:
   - Find leader (keptIndex=0) → leader_t (natural spline position)
   - Compute segSpellProgress[segIdx] = clamp01((leader_t − spellTransStart) / (spellTransEnd − spellTransStart))
   - Find middle kept index: midKeptIndex[segIdx] = floor(totalKept / 2)

2. Per-particle in main loop:
   - age = now − spawnTime
   - natural_linearT = age × fontSpeedScale / charLifetime
   - progress = segSpellProgress[segIdx] (shared across the word)
   - If p.isKept:
       delta = keptIndex − midKeptIndex
       boost_secs = progress × delta × (segDuration / totalChars) × fontSpeedScale × spellGrouping
       effective_linearT = (age × fontSpeedScale + boost_secs) / charLifetime
       // Monotonic guard
       if p._prevLinearT > effective_linearT: effective_linearT = p._prevLinearT
       p._prevLinearT = effective_linearT
       t = easeBlended(effective_linearT, linearBlend)
   - If p is an extra (p.isKept === false):
       Use preceding kept letter's boost (via p.prevKeptExtrasBefore logic)
       Fade alpha via sequential gap-index fade (unchanged from Design A),
       with spellFadeSpeed tuned so extras are ~invisible by progress ≈ 0.5

3. Guards:
   - inSpellTransition = (isKept !== undefined) && (progress < 1) →
       blocks fadeStart-based exit-fade until the whole word has formed
   - displayChar = (isKept && finalChar && progress >= 1) ? finalChar : char
```

### Component responsibilities

**`segSpellProgress` pre-pass** — O(n) over particles. Emits a small `{segIdx: progress}` map. Pure function of the current particle snapshot.

**Age-boost application** — inline in the main loop, replaces Design A's `ageBoost` + `prevKeptBoost` blocks. Same shape, new formula (`delta × spawnGap × fontSpeedScale × spellGrouping` instead of `extrasBefore / totalChars × segDuration × spellGrouping`).

**Monotonic guard** — adds one field (`_prevLinearT`) to the particle at render time. First frame initializes to 0. Cheap, stateless otherwise.

**Extras fade** — sequential gap-index fade from Design A stays. Only tuning change is `spellFadeSpeed` default (bump from 3.8 to ~5.0 to concentrate the fade into the first half of progress).

**Fade guard + displayChar swap** — identical to Design A, just keyed on `segSpellProgress[segIdx]` instead of per-letter `spellTransProgress`.

### Parameter changes

| Param | Old semantics | New semantics | Default |
|---|---|---|---|
| `spellTransStart` | `baseT` (age-space) at which per-letter transition begins | Spline-`t` at which segment collapse begins (keyed on leader's natural t) | 0.18 (same value, different meaning but visually coincident for small t) |
| `spellTransEnd` | `baseT` at which per-letter transition ends | Spline-`t` at which segment collapse completes | 0.41 |
| `spellGrouping` | Multiplier on extrasBefore-based ageBoost | Multiplier on (delta × spawnGap × fontSpeedScale); 1.0 ≈ tight spacing at 1 spawn-gap-per-letter | 1.0 (bump from 0.8) |
| `spellFadeSpeed` | Power on extra fade | Same (no mechanic change, retune default) | 5.0 (bump from 3.8) |

No new params introduced. `params.json` gets the two default bumps above; other user tunings untouched.

### Edge cases

- **Leader not yet spawned:** no `leader_t`, so `segSpellProgress[segIdx]` is absent → lookup yields `undefined || 0` → progress 0, phonetic state. Correct.
- **Leader dies mid-transition:** leader is oldest letter; dies when its own age > charLifetime. Once dead, `segSpellProgress` for the segment disappears. Any surviving kept/extra particles fall back to progress=0 → stops adjusting positions. In practice `spellTransEnd < 0.95` so leader will still be alive and the word will be past `fadeStart` by then; this is a graceful no-op.
- **Middle kept letter in particle state but not yet spawned:** irrelevant — `midKeptIndex` is derived from `p.totalKept` (pre-baked at spawn), not from a living particle.
- **Segment with no `charMap` (regular lyric):** `p.isKept === undefined`, so no pre-pass entry, no boost applied, no fade guard, no displayChar swap. Zero impact on non-spelling lyrics.
- **Very short word (1–2 kept letters):** `midKeptIndex` is `0` or `1`, delta range is narrow, boost magnitude is small, collapse is minimal and graceful.
- **Leader's `natural_t` already past `spellTransEnd` when the particle spawns late in frame:** `progress` clamps to 1 → full collapse immediately. Letter snaps into position once per render; since letters enter via burst, visually the snap is absorbed into the burst blend. Acceptable; worst case we can add a `progress` ramp-up, but YAGNI until we see it misbehave.

### Testing

Manual verification in the live preview with a webcam:
1. **Happy path:** "Hallelujah" visible as `Haaallelujah` right after spawn; by the time the leader reaches ~41% along the spline, the word reads as `Hallelujah` with tight kerning. No letter exits screen before collapse completes.
2. **Extras fade:** At ~halfway through the collapse window, the 'a' extras should be near-invisible and there's a visible gap where they were, pre-closure.
3. **No reverse motion:** The leader's spline position only advances. If it appears to step back, the monotonic guard has a bug.
4. **Non-spelling lyrics unaffected:** regular lines (no `phoneticWord`) render exactly as before.

No automated tests — this is visual behavior in a real-time render loop; regression coverage is manual eyeball + param sliders.

### Implementation sketch

Touch-points in `index.html`:

- **L2215–2230 (per-segment pre-pass):** replace the current `segLastSpawn` / `segSpellProgress` block (which anchors to last-spawned letter's baseT) with the new leader-t-based computation, including `midKeptIndex[segIdx]` derived from `p.totalKept`
- **L2247–2249 (per-letter `spellTransProgress`):** already a segment lookup — no change beyond the pre-pass rewrite
- **L2254–2268 (ageBoost + prevKeptBoost):** replace both formulas with the delta-based version that includes `fontSpeedScale`. Add `p._prevLinearT` monotonic clamp to the kept branch.
- **L2321–2342 (fade guard + extras fade):** mechanically unchanged; extras fade already reads the shared progress.
- **L2402 (displayChar):** mechanically unchanged.

`params.json`: bump `spellGrouping` 0.8 → 1.0, `spellFadeSpeed` 3.8 → 5.0.

## Open questions

None at this point — flagged in the brainstorm and resolved:
- Anchor: center-of-word, **but** motion is monotonic-forward (leaders slow, can hover, never reverse)
- Progress driver: leader's natural spline-t
- Extras-fade timing: compress into first half of collapse window (`spellFadeSpeed` bump)
