# FIX-148: the rigid glide on ENTER

- **Base:** branch head `9984916` (FIX-147 R2.1).
- **Branch:** `claude/mu-rigid-turn`. The code commit is `97c9edb`, and its iOS CI build is green with no new warnings.
- **Device status:** none of this has been run on a device.
- **Out of scope:** E-1 (the turn clock speed). `muTurnSpeed` and the turn-clock machinery are untouched.

---

## Audit

### A1. Why the hosted surface never sees `viewWillTransition` on enter: CONFIRMED from the code

The enter sequence (`muToggleFullscreen`, MUPlayerChrome.swift:1823) runs in one main-thread turn:

1. The unchanged FIX-139 decision calls `OrientationHelper.lock(.landscape, rotateTo: .landscapeRight)` (:1864). The geometry request is answered by the system later, not inside this call.
2. `beginMUFullscreenPresentation` moves the **view** of `LivePlayerViewController` into a new `MUFullscreenViewController` (:1943) and presents that controller `.fullScreen`, unanimated, from the cover's host (:1964).

Only the view changes parent. The **controller** stays a child of the cover's SwiftUI host, the `UIViewControllerRepresentable` parent chain. A `.fullScreen` presentation removes the presenter's view from the window. When the turn arrives, UIKit delivers `viewWillTransition` down the window's controller hierarchy:

- the root;
- then the presented chain, whose topmost member is now `MUFullscreenViewController`.

The hosted surface's controller is not in that chain, so it gets nothing. `MUFullscreenViewController` had no `viewWillTransition` override, so the transition stopped there. This matches the device log: 0 of 5 enters show flight lines.

Exits are different. The dismissal completes before the turn back starts, so the view is home and the host is topmost again. The surface then receives the turn directly. This matches the exits' full flight cycles.

### A2. What drives the one-frame cut: CONFIRMED mechanism; the share of each part is unmeasured

With no delivery, `mu.transitionInFlight` stays false for the whole enter. The rotation still resizes `MUFullscreenViewController.view` inside the system's animation, and autoresizing resizes the hosted view. The surface's `viewDidLayoutSubviews` → `layoutMUChrome` then runs **as if at rest**. Three things follow:

- **The picture snaps.** `placeMUVideo` (:1157) has no flight armed, so it writes `videoView.frame` plainly inside the system animation. That is the AVPlayerLayer model-bounds snap convicted in FIX-147: the picture lands on its final mapping at the first commit.
- **The chrome swaps page instantly.** `applyMUPresentation` (:938-940) runs on the first pass. FIX-143 holds the landscape page's fonts, chips and visibility back until the settle, but only while `transitionInFlight` is set. So the whole page presentation switches in one frame.
- **There is no settle.** `finishMURotation` never runs on an enter: no settle line, no settle pass, and the FIX-142 queue never engages (see B1).

Together these make the enter's first frame the largest frame-to-frame difference in the recording, as measured (frame-diff 36.5).

A third candidate, **PLAUSIBLE and not convicted**: if a frame is ever committed between moving the view out of its host slot and installing `MUFullscreenViewController`'s view in the window, that frame shows an empty slot. By the code, both happen in one turn on the main thread with an unanimated presentation, so they should land in one commit. In-house's enter-path instrumentation should settle this. The test is whether the fullscreen view is in the window when `present` returns.

### A3. The ~80° swing toward portrait and back: PLAUSIBLE; not owned here

The app makes exactly one orientation request on enter (:1864, landscapeRight). The swing therefore comes from the system re-evaluating orientation for the new topmost full-screen controller. `MUFullscreenViewController` declares supported `.all`, `shouldAutorotate = false`, and no `preferredInterfaceOrientationForPresentation`. A full-screen presentation without that preference falls back to a system-chosen orientation, which here was portrait, the presenter's. That re-evaluation then meets the `.landscape` app mask and is pulled back. This is the target of the in-house override, and I have not duplicated it.

FIX-148 makes the swing **visible in the log** for the first time. If it is an interface-level transition, it now reaches the surface as a forwarded transition, and each one logs:

- `mu transition: forwarded … size=… from=…`;
- `mu rigid flight: armed`.

An enter with exactly one forwarded pair (portrait → landscape) and a visible swing means the swing is not an interface transition.

Either way, the flight rides whatever transitions the system runs, rigidly. It does not suppress a swing. Suppressing it is the in-house measure's job.

### A4. Which mechanism can own the picture's motion during an enter

| Option | Verdict |
|---|---|
| Re-parent the surface's controller under `MUFullscreenViewController` | **Rejected.** It moves a controller that SwiftUI owns out of its representable parent, and it risks the hierarchy-inconsistency crash class. The adoption, teardown-release and refusal-rollback logic all assume the controller stays put. |
| A custom presentation / transition animator | **Forbidden** (the standing constraint). |
| An explicit animation started from the toggle, outside the system turn | **Rejected.** It would not share the system's clock or curve, so it could not be a rigid body with the rotation. That is the same desync class as E-1. |
| **`MUFullscreenViewController` forwards the window's transition to the hosted surface** | **Chosen.** The surface runs its existing, shipped MU transition path on the **same** system coordinator. The flight's end values go into the coordinator's own alongside animation, exactly as on exits. |

---

## Mechanism (the diff)

1. **`MUFullscreenViewController.viewWillTransition`** (MUPlayerChrome.swift:360). It calls `super`, then forwards to `MUFullscreenState.hostedController` through `muHostedWillTransition`. It forwards only while the hosted view is actually a subview of its view.
   - It adds no animation and no animator.
   - `supportedInterfaceOrientations` is untouched (`.all`, :368).
2. **The surface's single MU transition path** (PlayerView.swift).
   - `viewWillTransition` (:987) now calls `super`, then `muWillTransition(forwarded: false)`.
   - The forward (:1002) calls the same `muWillTransition(forwarded: true)` (:1013).
   - The body is the shipped FIX-142/143/147 code, byte for byte:
     - set the in-flight state;
     - arm the rigid flight with the picture's settled portrait frame;
     - start the turn clock (a no-op at `muTurnSpeed = 1`);
     - in the alongside block, run `layoutIfNeeded`, whose first pass applies the flight;
     - in the completion, run `finishMURotation`.
   - On an enter, the size passed is the fullscreen view's new size, which the hosted view fills.
3. **One transition, handled once.** `mu.activeTransitionCoordinator` is a weak reference to the coordinator being handled. A second delivery of the same coordinator is ignored and logged: `mu transition: second delivery of the same transition (…) — ignored`. The case it covers is an exit whose turn begins while the view is still hosted, so both the forward and the direct delivery fire.

What the enter now gets:
- It gets the same glide as the exit: final box once, a uniform scale on the coordinator's curve and clock, and the picture and chrome as one body.
- The chrome's page presentation is held until the settle, as on exits.
- It has a settle, which completes the settle line, the settle pass and the parity queue.

## Constraint mapping

| Constraint | How it is met |
|---|---|
| Nothing flies the SwiftUI host's root view | Only `videoView`, a plain subview, flies, as in FIX-147. The hosted view is resized by the system's own autoresizing inside the system turn, exactly as on exits. |
| No custom transition animator | None. The forward only calls into the surface, which uses `coordinator.animate(alongsideTransition:)`. |
| Fullscreen controller supports `.all` | Untouched. |
| Toggle decision block and parity queue untouched | Untouched. The forward makes the existing queue reachable on enters (see B1). |
| Player and item never recreated | Untouched. |
| Settled pages pixel-identical | The settle runs the same `finishMURotation` pass, and the flight ends at identity transform on the plain final frame. |
| Exit flight, close machinery, `lockWhenSettled` | Unchanged code. On exits the guard in point 3 above ensures at most one handling per transition. |
| In-house's three changes | Not touched. The new override sits between `viewDidLoad` and `supportedInterfaceOrientations` in `MUFullscreenViewController`. In-house's `preferredInterfaceOrientationForPresentation` override goes in the same class, so expect a merge at port time that sits next to it, not a semantic conflict. At `muTurnSpeed = 1` there are no `mu turn clock` lines by design. |

### B1. Flagged behaviour changes (all on the enter path)

1. **A toggle tapped during an enter turn is now queued** (FIX-142 parity) and runs at the enter's settle. Before, the enter was never "in flight", so the tap ran the decision immediately mid-turn. The page flag still read portrait, so the tap re-issued the enter and was effectively lost.
2. **Enters now produce a settle.** The `mu rotation settled` line, the settle pass and `serviceMUAfterExit` now run on enters. The after-exit service is a no-op with nothing waiting.
3. **The landscape page's presentation now appears at the settle** instead of on the first in-flight pass, the same as exits since FIX-143.

## On-device checks (enter)

1. **Per enter:** `mu transition: forwarded …` → `mu rigid flight: armed` → `applied … anim=inherited` → in-flight layout lines → `mu rotation settled`.
   - Exactly one forwarded pair means one interface transition.
   - Report `second delivery … ignored` lines on exits; they are expected and harmless.
2. **High-speed capture:**
   - no one-frame cut;
   - scale monotone with rotation progress;
   - worst single-frame step ≤15%;
   - no post-landing swing (with the in-house override; without it, the forwarded lines show whether the swing is an interface transition).
3. **Regression:**
   - exits: still no step, and zero rejections;
   - the close matrix and the board retry (FIX-147 R2.1);
   - chip switch while hosted, failover, sheets, PiP;
   - pixel comparison of both settled pages.
4. **The 0.5 s exit freeze:** every flight, clock and transition line is kept, and the new forwarded and second-delivery lines are added, so a recurrence shows which delivery path it took.
