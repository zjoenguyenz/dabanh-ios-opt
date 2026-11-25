# FIX-147: MU rigid fullscreen turn and portrait-first close (R2)

- **Base:** dabanh-ios-opt main `9cc4b16`. Its PlayerView.swift and MUPlayerChrome.swift are byte-identical to the handed copies. "base:L" below means a line in the base files.
- **Branch:** `claude/mu-rigid-turn`.
  - R1 = `ce81398`.
  - R2 = the commit that adds this file.
- **Device status:** nothing here has been run on a device. Every acceptance item still has to be measured (see the checks at the end).

R2.1 change (C-R2-1): the one retry now covers the board route as well as the close. Before, it was close-only. Its discipline is unchanged.

R2 changes:
- Finding 1: the turn clock is now time-continuous.
- Finding 2: the after-exit path is decided at the settle only.
- Finding 3: the route-driven close is covered, and the guarantee is stated exactly.
- New files touched: `DaBanhApp.swift` (OrientationHelper) and `ContentView.swift` (the two portrait-request points). Both changes are MU-only.

---

## PART 1: Audit of the code and of the diagnosis

### 1. Verdict on the diagnosis

**ENTER: confirmed (unchanged from R1).**
- `videoView` (`PlayerLayerView`) has the `AVPlayerLayer` as its backing layer.
- The FIX-143 alongside pass writes `videoView.frame`, so UIKit animates the layer's bounds and position.
- AVPlayerLayer lays out its private video sublayer against the layer's **model** bounds, without animation. At the first commit the picture is therefore already mapped for the final box, and only the box interpolates. That is the one-frame footprint step.
- "Remote layer" is the wrong label for this. The fault is an unanimated sublayer layout driven by model values. A transform on the same layer is composited and never re-lays the sublayer out.

**Why the logs look clean.** Every value logged is a MODEL value (base MUPlayerChrome.swift:948-955; base PlayerView.swift:1036-1041). The model is at the final value from pass 1 on, so a step can never appear in those lines.

**EXIT is not explained by the snap alone.**
- Snap-to-final predicts the picture *shrinking*; it was measured *growing* by 35–42%.
- The code candidate is base MUPlayerChrome.swift:1583-1589. The dismissal completion re-inserts the view at the enter-time **portrait** frame while the host is still landscape (D2, fixed).
- Status: PLAUSIBLE. Only a presentation-layer trace can confirm it.

**Rejection at close (updated by Finding 3).**
- The rejection convicted on device was **route/surface-driven**: the player surface was closed or replaced while hosted.
- The portrait request fired from ContentView's two points while a presentation transition was still in flight:
  - the cover's `onDismiss` (base ContentView.swift:268);
  - the route-to-nil branch (base ContentView.swift:303-312).
- During such a transition the topmost controller's supported set collapses to the current orientation: "Requested: portrait; Supported: landscapeRight" (DaBanhApp.swift:88-92).
- R1 only covered the player's own close button. R2 covers those two points directly (see Finding 3).

**Crash candidates (item 3).**
- (a), a foreign VC root view inside the transition container, is the more consistent explanation. Look for a `UIViewControllerHierarchyInconsistency`-type exception in the report.
- (b) is unlikely to crash on its own.
- In this design the only thing that flies is `videoView`, a plain subview. No root view moves and no animator is used.

### 2. Additional defects in this region

| # | Where (base) | What it does on the device | Status |
|---|---|---|---|
| D1 | MUPlayerChrome.swift:948-955; PlayerView.swift:1036-1041 | Every rotation log is model-only, so a step can never be seen in the logs. | New `mu rigid flight` lines record start, end, scale and the animation source. A presentation-layer sample is still needed. |
| D2 | MUPlayerChrome.swift:1583-1589 | The exit restore writes the enter-time portrait frame while the host is landscape, so the slot is mis-sized for at least one commit. | **Fixed**: a view that filled its host slot at enter is re-filled from the current `host.bounds`. |
| D3 | MUPlayerChrome.swift:1590-1594 | The board-after-exit opens in the dismissal completion, before the turn back, so the sheet races the turn. | **Fixed**: it opens at the portrait settle. |
| D4 | ContentView.swift:268, 303-312, reached from PlayerView.swift:1554 or from a route/surface change | A portrait request issued during an in-flight presentation transition is rejected. | **Fixed** (Finding 3). |
| D5 | PlayerView.swift:919 | `viewDidLayoutSubviews` writes the picture frame a second time on every MU pass. | **Fixed**: under MU the page function is the only writer. |
| D6 | PlayerView.swift:877 | The autoresizing mask adds a system-driven animated frame write to the picture during the turn. | **Handled**: the flight removes those animations before applying itself. |
| D7 | PlayerView.swift:431-452 | The slate's close while in fullscreen state. | **Rejection now covered**: it reaches the ContentView points, which wait out the transition. It is still *visually* a close in landscape followed by a turn of the board, because the slate does not go through portrait-first. Recommended follow-up. |
| D8 (plausible) | adoption on source swap | A successor adopted while hosted can't get `viewDidAppear`, so it can go through a dismiss + re-present cycle with a possible flash. | Not changed. Check on device. |

---

## PART 2: Mechanism

### (a) The rigid picture: `placeMUVideo` / `armMURigidFlight` / `retireMURigidFlight` (MUPlayerChrome.swift)

1. `viewWillTransition` arms the flight before any in-flight pass. It captures the settled picture frame and the coordinator's duration and curve.
2. The first in-flight pass runs without animation:
   - it removes the system's animations on `videoView` (D6);
   - it sets **bounds to the final size**, so AVPlayerLayer maps the picture once, at the size it will keep;
   - it sets the centre to the start centre;
   - it sets `transform = scale(k)`. k = start picture width ÷ final picture width, from `presentationSize` and the gravity. Both picture rects have the video's aspect, so the picture is unchanged at t = 0.
3. Inside the inherited alongside animation it writes the centre → final and the transform → identity. If no animation is inherited, it uses an explicit `UIView.animate` over the remaining time on the coordinator's curve.

What this answers:
- The pass-1 snap is gone: nothing is re-mapped mid-flight.
- Scale is monotone, and the footprint follows the angle.
- Picture and chrome turn as one rigid body: they share one curve and one clock.
- No root view moves and no animator is used.
- It builds on FIX-143 rather than replacing it.
- At rest the function is exactly `videoView.frame = target`, and `retireMURigidFlight` guarantees identity at every settle. The settled pages are therefore unchanged.

### (b) Turn duration: the turn clock (`MUTurnClock`, end of MUPlayerChrome.swift)

- During an interface turn the window layer runs ×`muTurnSpeed` (1.4): 0.37 s → about 0.26 s, about 8 frames.
- Set `muTurnSpeed = 1` to disable it.
- See Finding 1 for the timing invariant.

### (c) Close and board after exit

See Findings 2 and 3.

---

## FINDING 1: the turn clock is now time-continuous

**What was wrong.**
- R1 restored the saved `beginTime` and `timeOffset` verbatim at the end of the turn.
- During the turn the window's local time advances 1.4× faster.
- Replaying the pre-turn values stepped the local time **backwards** by about 0.4·T (about 0.1 s) at every settle. Any animation spanning the restore would jump.

**What changed.**
- The clock is a process-wide `MUTurnClock`. Every switch, on and off, goes through one function, `switchRate`, which at a single instant `now` does:
  ```
  timeOffset ← layer.convertTime(now, from: nil)   // local time at now
  beginTime  ← now
  speed      ← new rate
  ```
- Only the **rate** is restored exactly (`savedSpeed`). The old `beginTime` and `timeOffset` are deliberately not written back.

**Invariant.**
- The layer's local time is t_local = (t − beginTime)·speed + timeOffset.
- At each switch, t_local(now) is the same before and after the write. Local time is therefore a continuous, monotone function of media time across the whole turn, and nothing in the window jumps either way.
- After a turn of length T, the window's local time correctly leads its pre-turn mapping by 0.4·T. No app code computes layer times directly: a grep finds no `beginTime`, `timeOffset` or `CACurrentMediaTime` use outside this clock and the flight.

**Every exit path ends it under the same invariant.**
- **Settle:** `finishMURotation` → `endMUTurnClock`.
- **Refusal:** when `animate(alongsideTransition:)` returns false, the code calls `finishMURotation` directly, which takes the same path.
- **Teardown:** the teardown → `endMUTurnClock(reason: "teardown")`.
- **Superseded surface:** the clock is not owned by a surface. Whichever surface settles or tears down next ends it.
- **Last guard:** one one-shot check per turn, at 2× the expected duration + 0.5 s. It is generation-checked and never repeats.

## FINDING 2: the dismissal completion no longer services the after-exit path

**What was wrong.** R1's hosting dismissal completion called `serviceMUAfterExit` while still landscape. That duplicated the exit toggle's own servicing and spent the single retry before any settle existed. A landing that later genuinely needed the retry fell into the 1 s fallback and closed in landscape.

**What changed (MUPlayerChrome.swift: `requestMUCloseAfterExit`, `issueMUAfterExitExit`, `serviceMUAfterExit`, `completeMUAfterExitAtHostingHome`).**
- The after-exit path is decided **only at a settle** (`finishMURotation`, after the parity-queued toggle).
- Each issued exit records `afterExitExitIssued` and `afterExitIssuedAt`.
- Each settle records `lastSettledArmedAt`, the arm time of the transition that just settled.

| At a settle | Action |
|---|---|
| Portrait, a toggle serviced in this settle | Wait (that toggle is an enter). |
| Portrait, hosting not home | Set `afterExitAwaitingHosting` and wait. |
| Portrait, hosting home | Run the close or board action. |
| Landscape, a toggle serviced in this settle | That toggle is the exit. Record it as issued. |
| Landscape, no exit issued yet (a tap mid-turn) | Issue the first exit. This is **not** the retry. |
| Landscape, the transition began **after** the issued exit | The exit's landing failed: spend the **one** retry. This applies whichever action is waiting: close, board or both (C-R2-1). |
| Landscape, the transition began before the exit | Wait for the exit's own turn. |

- The dismissal completion now calls `completeMUAfterExitAtHostingHome()`. That function can only *finish* an action a portrait settle has already approved and parked (`afterExitAwaitingHosting`). It never decides, never issues an exit and never touches the retry.
- The completion clears the hosting records (`presentedController = nil`) before making this call, so the check sees the hosting as home.

**C-R2-1 (R2.1).** In R2 the retry was also guarded by `afterExitClose`. A board-only request whose exit landed in landscape therefore got no retry: it went to the 1 s fallback and opened the board in landscape. That guard is removed. The retry is now spent for any waiting action, still exactly once (`afterExitRetried`), and still only behind `lastSettledArmedAt > afterExitIssuedAt`. Its log line is now `after-exit: the exit's landing settled on the landscape page — the one retry (close=… board=…)`.

**Why the invariant holds.**
- The retry is written in exactly one place, behind `lastSettledArmedAt > afterExitIssuedAt`. That condition is true only for a transition that began after the exit was issued and still landed in landscape, which is a genuinely failed landing.
- In the normal sequence the dismissal completes before the turn's settle:
  - tap → issue at t0;
  - the dismissal completes → no-op;
  - the turn is armed at t1 > t0;
  - the settle in portrait runs the close.
- If the dismissal completes after the settle, the settle parks the action and the completion finishes it.
- The retry is never consumed in either case.

## FINDING 3: the route/surface-driven close is covered, and the guarantee is stated exactly

**What was wrong.** The convicted sequence was route/surface-driven, and its portrait requests (ContentView `onDismiss` and route-to-nil) were untouched. R1's notes also claimed those requests "can't be rejected", while R1's own fallback could still close in landscape.

**What changed.**
- New `OrientationHelper.lockWhenSettled(_:rotateTo:reason:)` in DaBanhApp.swift. Under MU, both ContentView points call it instead of `lock(.all, rotateTo: .portrait)`. The classic and highlight paths keep `lock` unchanged. It works as follows:
  1. It widens the mask **now** (`lock(.all)`, the same mask timing as before).
  2. On the next main-queue turn it walks the window's presentation chain. While any controller is being presented or dismissed, or has a transition coordinator, it waits on that coordinator's completion (or on the next main-queue turn for an unanimated transition), then re-checks. This is capped at 6 hops; past that it requests anyway and logs it.
  3. If the window is already portrait, it makes **no request**.
  4. Any newer `lock()` (a player re-opened) cancels it through `lockGeneration`.
  5. It then issues the geometry request through `lock`, with the same error-handler log line.
- It is one-shot per call and adds no periodic work.

**Close routes covered, exactly.**

| Route | What happens |
|---|---|
| Player close button, landscape page (hosted or free rotation) | Portrait-first: exit, then close at the portrait settle. The cover is dismissed while portrait, so the ContentView request finds the window portrait and makes no request. |
| Player close button, mid-turn | Decided at that turn's settle (see Finding 2). |
| Player close button, portrait page | Closes immediately, as before. The window is portrait, so no request is made. |
| Board tap from fullscreen | Portrait-first; the board opens at the portrait settle. |
| Route → nil while hosted (surface closed or replaced, flip back to the board) | The request waits out every in-flight presentation transition, including the hosting release in teardown and the cover's dismissal. |
| Cover dismissed by any other path (`onDismiss`), including the slate close (D7) | Same as route → nil. |
| Classic player | Unchanged. |

**What the 1 s fallback does and does not guarantee.**
- It **does** guarantee that a waiting close or board action runs within about 1 s of the last issued exit, even if no portrait settle ever arrives (for example, a rejected geometry request). It runs once per request, generation-checked, and logs `after-exit: no portrait turn within 1 s`.
- It does **not** guarantee portrait. In that case the cover may be dismissed in landscape. The portrait request that follows still goes through `lockWhenSettled`, so it is not issued during a presentation transition.
- A rejection is still possible in two cases:
  - the topmost controller left after the transitions does not support portrait (by code it is the root host, which follows `AppDelegate.orientationLock = .all`);
  - the 6-hop cap is exhausted.
- Both cases log. So "zero rejections" is the expected result on these routes, not a proof. A `1 s fallback` line or a `requesting anyway` line in a test log means the run failed that guarantee and must be reported.

**Residual ordering risk.** If SwiftUI tears the player down (and releases the hosting) *after* the deferred request has already checked the chain, the request goes out while fsVC is still on top but not in transition. fsVC supports `.all` (MUPlayerChrome.swift, `supportedInterfaceOrientations`), so it is accepted. The turn may then overlap the unanimated release. This is not a rejection, but check it on device.

---

## Constraints (unchanged and verified in the diff)

- No root-view motion and no custom animator.
- fsVC is still `.all`.
- The toggle's decision block and parity queue are not modified; the code only calls them.
- The classic player is untouched (its branches in `viewDidLayoutSubviews` and `closeTapped`, and both ContentView points under `!isMU`).
- The player and its item are never recreated.
- No DiagLog line was removed.
- No periodic work: only one-shot guards on user actions or turns.

## On-device checks

1. **Every turn:**
   - `mu rigid flight: armed` → `applied … anim=inherited` → `mu turn clock: on ×1.4` → `off (settle)`.
   - These lines must never appear: `in-flight target moved`, `off (guard …)`.
2. **Turn clock (Finding 1):** animate a control fade across a settle and confirm it does not jump. Confirm playback timing is unaffected.
3. **High-speed capture:** ≤9 frames, footprint vs. angle, per-frame scale ≤15%.
4. **Exit growth:** if it survives D2, I need a trace of `videoView.layer.presentation()` and of the host slot.
5. **Close matrix:** run every route in the table above. Each must have zero `orientation geometry update rejected` lines, and the board must appear in portrait. These lines mean that run failed: `1 s fallback`, `requesting anyway`.
6. **Retry (Finding 2, C-R2-1):** a close tapped during an enter turn should log `exit issued (landscape settle after a mid-turn request)`, not `the one retry`. If a board-tap exit lands in landscape, it should log `the one retry (close=false board=true)`, and the board must then open in portrait. The 1 s fallback should not fire.
7. **Regression:** chip switch while in fullscreen (D8), failover, quality/audio sheets, PiP, and pixel comparison of both settled pages.
