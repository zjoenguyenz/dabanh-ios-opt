# FIX-150: enter order, turn first and host after

- **Base:** branch head `ffd337f` (FIX-148).
- **Branch:** `claude/mu-rigid-turn`. The code commit is `875b50d`, and its iOS CI build is green with no new warnings.
- **Size:** one file, MUPlayerChrome.swift, +97 / −3.
- **Device status:** none of this has been run on a device.

---

## 1. The refutation: ACCEPTED

The FIX-148 premise was wrong. I inferred that UIKit delivered the enter's turn to the fullscreen controller, but I inferred it from the controller hierarchy, not from a log. The 1.49 log shows the override never fired, and nothing else received a transition either. FIX-148's forward is **inert on enters**. It is kept only because it is harmless and its log lines are now part of the record. The leader may drop it at port time.

### My revised account of the shipped enter

These are the 1.49 timestamps:

| Time | Event |
|---|---|
| +0.000 | The toggle's FIX-139 decision calls `lock(.landscape, rotateTo: .landscapeRight)`. The mask and supported set are updated, and the geometry request is sent. |
| +0.004 | In the **same main-thread turn**, `beginMUFullscreenPresentation` moves the view and presents the controller `.fullScreen`, unanimated. The cover's host leaves the window, and the fullscreen controller becomes topmost. |
| +0.412 | The first and only layout runs, already at the landscape size (956x440). |
| +0.419 | `viewDidAppear` and the present completion run, with the window 956x440 but `interfaceOrientation` still portrait (1). |

**Inference (consistent with every line above, but not documented by Apple):**
- The scene's landscape geometry was granted while the full-screen presentation was still in progress.
- UIKit did not run a coordinated rotation for it. It applied the new window size as part of finishing the presentation, with no `viewWillTransition` to anyone.
- The ~0.41 s between presenting and completing is the presentation's completion held while the scene's geometry changed.
- The interface orientation was updated **afterwards**, with no size change left to make, so again no `viewWillTransition`.

**What that means on screen:**
- **The cut:** the canvas jumps to the fully styled landscape page in one frame. Nothing was deferred, because the surface never knew a turn was happening.
- **The swing:** the interface orientation catches up later and moves visibly with no size change, about 84° out and back over ~15 frames.

**The contrast that matters: exits.** On exit, the decision's geometry request goes out while the dismissal makes the cover's host topmost again, and no presentation is in flight when the turn runs. Every exit in the same log got an ordinary coordinated turn, delivered directly to the player, carried by the flight, with no swing. FIX-144's own refusal-rollback comment records the same for the enter when no presentation happens: "the window rotates in place and the coordinator path settles it, exactly the 1.43 fallback".

## 2. Candidate directions, evaluated

| Candidate | Verdict |
|---|---|
| Re-assert the geometry request at the present completion | **Rejected.** By then the scene geometry is already landscape. The request asks for an orientation the scene already has, and the catch-up has nothing left to drive. At best it does nothing. It cannot undo the cut or the lag that already happened. |
| Hold the portrait composition, or a picture transform, from the toggle until the interface settles | **Rejected.** The swing is a motion of the whole interface. The app has no hook inside it (no coordinator) to counter-rotate against, and a surface held still in app space still swings with the window. It would hide nothing and add its own step. |
| Defer the landscape page's presentation to a settle signal | **Rejected as a fix on its own.** It changes what is drawn during the cut, not the cut or the swing. |
| **Remove the overlap: let the window turn first, and present at the landscape rest** | **Chosen.** It removes the cause instead of covering it. The enter becomes the mirror of the exit, a path the same device log shows working. |

## 3. Mechanism (MUPlayerChrome.swift)

1. **The decision block is untouched** (:1879, `lock(.landscape, rotateTo: .landscapeRight)`), and so is the parity queue. Only `syncMUFullscreenHosting` (:1904) changes, and only on enter. It calls `presentMUFullscreenAtLandscapeRest` (:1951) instead of presenting at once:
   - If the page is still portrait, or a turn is in flight, nothing is presented. `mu.hostingDeferred` is set and logged once: `mu fullscreen hosting: deferred to the landscape rest (toggle)`.
   - At a landscape rest it presents immediately, as before. With a portrait-only toggle origin, that case only comes from successor adoption.
2. **The turn.** The geometry request is now judged against the cover's host, which is the topmost controller, exactly as on every exit, with no presentation in flight. The window runs its ordinary coordinated turn:
   - the player receives `viewWillTransition` **directly**;
   - the shipped FIX-147 path runs unchanged: rigid flight, in-flight layout, the presentation held, `finishMURotation`;
   - the turn clock follows in-house's `muTurnSpeed`.
3. **The handover.** `layoutMUChrome` (:945) watches for a pass that is at rest (not in flight) on the landscape page while `hostingDeferred` is set. The enter's settle pass is one. It then schedules **one main-queue hop** to `serviceMUDeferredHosting` (:1973), never presenting from inside a layout pass.
   - That function re-checks everything: deferred, not torn, `isActive`, no presentation up, landscape, not in flight.
   - It then logs `landscape rest reached — presenting interface=…` and calls the unchanged `beginMUFullscreenPresentation` (:2018).
   - The window is already landscapeRight and the app mask is `.landscape`, so presenting a controller that supports `.all` (and in-house's landscape preference) changes no orientation.
   - The view moves into a container of the same size at the same place: from the host's full landscape slot into the fullscreen view's bounds.
4. **Cancellation.** All of these leave nothing to present:
   - an exit decided while the presentation is still owed (`syncMUFullscreenHosting` logs `deferred presentation cancelled (exit)`);
   - a queued toggle serviced at the settle;
   - a close or board after-exit issued at the settle;
   - a teardown (`isTorn`);
   - a route ending (`isActive = false`).
5. **Successor adoption** with no presentation up (:2228) goes through the same rule. A successor born mid-turn presents at its own landscape rest, not mid-turn.

**The expected enter on screen:** the same rigid glide the exits have, then an invisible handover. The owner's acceptable fallback, a clean cut with no swing, is not engineered here. A coordinated turn is what the exits already show working.

## 4. Constraint mapping

| Constraint | How it is met |
|---|---|
| Nothing flies the SwiftUI host's root view | Only `videoView` flies, in the shipped flight. During the turn the player view sits in its host slot and is resized by the system turn, exactly as on exits. |
| No custom transition animator | None. The presentation is the shipped unanimated one, just made later. |
| Fullscreen controller supports `.all` | Untouched. |
| Toggle decision block and parity queue untouched | Untouched. Only the hosting sync that runs **after** the decision changed. |
| Player and item never recreated | Untouched. |
| Exits, close machinery, `lockWhenSettled` | Exit branch unchanged (it also clears a still-owed enter presentation). After-exit and `lockWhenSettled` code unchanged. |
| Settled pages pixel-identical | The enter settles on the same landscape page through the same settle pass. The hosted rest is the same controller, view and frame as before. |
| No log lines removed | 0 removed. Added: `deferred to the landscape rest`, `landscape rest reached — presenting`, `deferred presentation cancelled (exit)`. |
| Settle path not redesigned | `finishMURotation` is untouched. The handover hangs off the settle's existing layout pass, one hop later. In-house's identity-aware settle and adoption handover therefore port on top unchanged. The one adoption line (:2228) is flagged for the leader. |

### Flagged behaviour changes and residual risks

1. **Hosting starts ~1 turn later on enter**, at the landscape rest instead of at the toggle. During the ~0.3 s turn, these happen against the unhosted surface:
   - **A sheet tapped mid-turn** is presented from the surface itself (`muSheetPresenter` falls back to `self`). The handover's `present` can then be refused, because the cover's host is already presenting. The shipped refusal rollback then runs (`present REFUSED — enter rolled back`), leaving an unhosted landscape page until the next toggle. This is an edge case, it is logged, and the page is not broken.
   - **Physical rotation:** the mask is `.landscape` and the topmost is the cover's host. A physical turn to landscapeLeft during that window could therefore be honoured before the handover. While hosted, the fullscreen controller's `shouldAutorotate = false` blocks it, as before.
2. **A rejected or absent enter turn** leaves the presentation owed, the page portrait, and `isActive = true`. Nothing is presented over a portrait window, which would recreate the overlap. The next toggle re-decides from the live layout. The log shows `deferred …` with no `landscape rest reached`.
3. **The host slot is now saved in landscape.** `hostOriginalFrame` is captured at landscape. The exit restore uses the host's current bounds for a view that filled its slot, which is the logged 1.49 case (`host slot 0,0 440x956`), so nothing changes there. A non-filling slot would restore a landscape-time frame, and no such slot exists in this app today.

## 5. Device checks that would prove it or kill it

**Prove:**
1. **The log, per enter, in order:**
   1. `mu fullscreen toggle … direction=enter`
   2. `orientation lock: mask=24 rotateTo=3`
   3. `mu fullscreen hosting: deferred to the landscape rest (toggle)`
   4. `mu rigid flight: armed` (no `forwarded`)
   5. in-flight layout lines, then `mu rigid flight: applied … anim=inherited`
   6. `mu rotation settled … size=956x440`
   7. `mu fullscreen hosting: landscape rest reached — presenting interface=3`
   8. `presenting (host slot …)`
   9. in-house's `mu fsVC: viewDidAppear bounds=956x440 interface=3`, followed by the present completion

   The decisive values are **`interface=3` at `viewDidAppear`** and a settle line on the enter. Both were absent in 1.49.
2. **The recording:**
   - no one-frame cut: the largest enter frame-diff is in the exit range;
   - **no swing**;
   - picture scale monotone with rotation progress, worst step ≤15%;
   - no visible change at the handover frame.

**Kill:**
- `deferred …` followed by **no** `rigid flight: armed` and no settle: the host-topmost turn is not coordinated on enter after all.
- `landscape rest reached` with `interface=1`, or a swing that remains: the presentation still re-evaluates orientation.
- A visible step at the handover.

If it is killed, my conclusion would be that nothing inside the standing constraints beats the system here. The remaining option is the owner's acceptable outcome: a clean cut with no swing. That would need the swing's driver identified by in-house's enter instrumentation first.

**Regression:**
- exits: no step, zero rejections;
- the full close matrix and the board retry;
- chip switch and failover while hosted (the adoption path);
- quality and audio sheets, both after the handover and tapped mid-turn (expect the logged refusal in the edge case above);
- PiP while hosted;
- pixel comparison of both settled pages.
