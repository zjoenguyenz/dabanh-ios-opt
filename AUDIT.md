# AUDIT.md: Đá Banh iOS + tvOS code audit (Phase 1, report only)

> **Scope correction (leader, 2026-11-23): the tvOS findings in this audit are superseded.**
> Production tvOS is a separate repository (`dabanh-tvos`, v1.7.6, green and owner-approved), which this audit did not see. The tvOS half below measured the legacy `DaBanhTV` target inside this monorepo instead. So:
> - **F1** (DaBanhTV does not compile) is not a shipping blocker. It stays on the list only as a low-priority monorepo cleanup and is skipped unless the leader says otherwise.
> - **Class G** (iOS ↔ tvOS drift) was measured against the wrong tree. Its findings G1–G7 do not describe production tvOS.
> - **The G3 pill question is withdrawn.**
> - **No tvOS finding is to be acted on**, including the tvOS halves of C1 and E5 and the tvOS rows of F.1. The tvOS rows of F.1 still describe the legacy target's build correctly.
>
> Phase 2 covers the iOS findings only: H1, then E1, then E2.

**Tree audited:** `zjoenguyenz/dabanh-ios-opt` `main` @ `b800b7b` (2026-11-23). Apart from the brief, this tree matches production `dabanh-ios` @ `6a8cb0d`: a `diff -rq` found only `highlights.json` different.
**In scope:** DaBanh (iOS 1.35/36), DaBanhTV (tvOS 1.3.5/17) + DaBanhTVTopShelf, the shared core, `tools/highlight_collector.py`, `.github/workflows/build-dabanh.yml`.
**Out of scope:** JoeTV, TV and JoeTVTopShelf, plus `build-joetv.yml`. I did not audit, build or modify these. The CI builds were dispatched as `target=ios` and `target=tvos` separately because `target=both` also builds the out-of-scope `TV` scheme.
**No code was changed.** The only repository writes are this file and two throwaway refs used to get class F evidence (see §F.0): `audit/tvos-last-green` (points at the existing commit `0859de6`, no new commit) and `audit/ipa-dissect` (one commit adding only the dissect workflow). Neither touches `main`. This session's git access was refused a delete (HTTP 403), so **both refs still exist and should be deleted by the owner**. Their run logs stay on GitHub either way.

### Evidence labels

- **CONFIRMED**: proven from the code, a CI log, a dissected binary, or a ported rule run on real data.
- **PLAUSIBLE**: the code path exists, but the trigger or impact needs device evidence.
- **UNMEASURED**: a cost claim with no timing behind it. A measurement method is given for each one.
- **BEHAVIOUR CHANGE**: the proposed fix changes observable behaviour (brief rule 1).
- **PROTECTED**: touches rule-3 machinery, so it is report-only and needs a written risk analysis before any PR.
- **SHARED-CORE**: compiles into both apps.

### Limits of this audit

- This host has no Swift toolchain. Parsing rules were ported line-for-line to Python (Appendix B) and run against the real producer files. Those results are evidence about the rules, not about the compiled Swift.
- ESPN, FotMob and the other score feeds are not reachable from this host: the proxy returned 403. So H covers the M3U, crest and highlights contracts against real files, but not the live score feeds.
- The proxy also blocks GitHub artifact downloads to this host, so the IPAs were opened on a GitHub macOS runner (§F.0).

---

## 0. Ranked summary (impact ÷ risk, highest first)

Impact and fix risk are scored 1–5 (5 = worst outage or riskiest fix). The rank is impact ÷ risk. Ties are broken by urgency.

| # | ID | Finding | Class | Impact | Fix risk | Labels |
|---|----|---------|-------|:-:|:-:|---|
| 1 | **F1** | **DaBanhTV does not compile at HEAD.** It has been red since `f7477ba` (2026-11-19): 16 errors in two shared files that reference iOS-only types. Production is the same tree, so no tvOS build can ship and Phase 2's "keep both targets green" rule cannot hold. | F/G | 5 | 2 | CONFIRMED · SHARED-CORE |
| 2 | **H1** | Kickoff and `Updated` stamp year-wrap. A match that kicks off at 23:00 on 31/12 (ICT) leaves the live lists at midnight and becomes "upcoming" a year later. New Year fixtures land a day early in leap-year Decembers. The mirror-freshness gate reads a stamp more than 30 days old as *fresh*. The next New Year's Eve is in 38 days. | H/E | 4 | 1 | CONFIRMED (ported) · PROTECTED · SHARED-CORE · BEHAVIOUR CHANGE |
| 3 | **E1** | A prewarm "dead" verdict never expires, which defeats FIX-130's 30-second TTL. A host that recovers stays hidden from the ladder and the board until the app relaunches. | E | 4 | 2 | CONFIRMED · PROTECTED · BEHAVIOUR CHANGE |
| 4 | **E2** | FotMob incidents (scorers, cards) are fetched once per match per app run. A live FotMob-only fixture's scorer list freezes while its score keeps moving. | E | 3 | 2 | CONFIRMED · PROTECTED · BEHAVIOUR CHANGE |
| 5 | **B1** | `invalid-urls.json` only grows. Signed URLs churn on every publish, and nothing prunes URLs that left the playlist. The file is decoded at every launch and rewritten after every validation pass. | B | 2 | 1 | CONFIRMED (growth) · UNMEASURED (size) · SHARED-CORE |
| 6 | **A1** | tvOS home recomputes the full `FixtureGrouping` about 8 times per body evaluation. The body re-evaluates on every focus move (every remote swipe), plus twice more in the focus `onChange`. | A | 3 | 2 | CONFIRMED (call count) · UNMEASURED (time) |
| 7 | **A2** | iOS `liveChannels`, `entries(for:)` and `sources(for:)` are recomputed on every access. The cost is O(channels × states) plus name normalisation, all on the main actor. Every state revision re-renders every store observer, up to about 5 times per poll cycle. | A | 3 | 2 | CONFIRMED (call graph) · UNMEASURED (time) |
| 8 | **E3** | The launch validation pass swallows the post-refresh pass. The freshly fetched list is not probed until the next refresh, about 15 minutes later. | E | 2 | 1 | CONFIRMED · PROTECTED · SHARED-CORE · BEHAVIOUR CHANGE |
| 9 | **C1** | AVPlayerItem KVO callbacks mutate non-Sendable coordinator state off the main thread: `HighlightMP4View.Coordinator` (2 Swift 6 warnings in the CI log) and `TVPlayerView.Coordinator.fired`. The iOS live player already hops correctly. | C | 2 | 1 | CONFIRMED |
| 10 | **B2** | `PlaylistStore.probeWave` creates a `URLSession` per wave and never invalidates it. Every other per-call session in scope invalidates; the only other one that doesn't is `TVImageCache.session`, an intentional app-lifetime singleton. | B | 1 | 1 | CONFIRMED · UNMEASURED (memory) · SHARED-CORE |
| 11 | **G3** | Shared `League` gained `.germany` and `.cups` since tvOS 1.3.5. Repairing F1 therefore adds two pills to the TV's already-full first tab row. The Cups pill is always empty on TV. | G | 3 | 3 | CONFIRMED · design-gate exposure |
| 12 | **F2** | CI `latest-stable` on `macos-latest` gives Xcode 26.6 / SDK 26.5, and will never reach Xcode 27 on that label. Both the runner and the Xcode version are effectively unpinned. | F | 2 | 1 | CONFIRMED |
| 13 | **G2** | The tvOS player has no resolve, pre-flight, stall watchdog or load timeout. A source that never reaches `.readyToPlay` or `.failed` leaves "Đang tải…" on screen forever. "FIX-128 tvOS tap-to-play port" changed no tvOS file. | G/E | 3 | 3 | CONFIRMED (code) · PLAUSIBLE (trigger) · PROTECTED |
| 14 | **H2** | The mirror's `Updated DD/MM HH:MM` pseudo-entry parses as a real channel with a no-signal stream. It lands in the Khác tab on both apps. | H | 2 | 2 | CONFIRMED (ported) · intent unclear |
| 15 | **H3** | The deployed collector is not the in-repo `tools/highlight_collector.py`. Production dates gained seconds at `6d31df2` with no tool change, and the in-repo tool still emits the shape that caused the 1.34 incident. | H | 2 | 1 | CONFIRMED |
| 16 | **G1** | tvOS has no match-state layer: chips, minutes and rails use the wall clock only, while iOS is state-aware. | G | 2 | 4 | CONFIRMED · BEHAVIOUR CHANGE if ported |
| 17 | **G4** | tvOS has no diagnostic log export, and the shared boot header says "iOS" on tvOS. This contradicts the brief's "both apps ship a log export". | G | 2 | 2 | CONFIRMED |
| 18 | **A3** | iOS crests and logos use bare `AsyncImage`, with no decoded-image cache. tvOS fixed exactly this in 1.3.5. | A/G | 2 | 2 | CONFIRMED · UNMEASURED |
| 19 | **A4** | `Channel.id = UUID()` per parse, and `Equatable` includes it. Every refresh re-identifies every row and fires tvOS `.onChange(of: store.channels)`. | A/D | 2 | 3 | CONFIRMED · PLAUSIBLE (focus effect) · SHARED-CORE |
| 20 | **A5** | `NSRegularExpression` is compiled per call in `MatchTitle.clean`, `Crests.teams` and `MatchTitle.fixtureKey`. These are on the A1/A2 hot path. | A | 2 | 1 | CONFIRMED · UNMEASURED · SHARED-CORE · PROTECTED (Crests) |
| 21 | **C2** | 7 Swift 6 isolation diagnostics in `PlayerView.swift` compile as warnings, not errors: main-actor members touched from `Timer` and `NotificationCenter` closures. | C | 1 | 1 | CONFIRMED · PROTECTED |
| 22 | **E4** | Both players pass headers through the string key `"AVURLAssetHTTPHeaderFieldsKey"`, which is not a documented AVFoundation option. | E | 2 | 3 | PLAUSIBLE · PROTECTED |
| 23 | **F3** | The TopShelf appex ships with version 1.0/1 inside DaBanhTV 1.3.5/17. | F | 1 | 1 | CONFIRMED (report only: version edits are out of scope) |
| 24 | D-x | Code-health items (§D). | D | 1–2 | 1–2 | CONFIRMED |

**Class F verdict, in one line:** neither app has the JTV defect. iOS: `DTSDKName iphoneos26.5` / `DTPlatformVersion 26.5` against `MinimumOSVersion 17.0`. tvOS: `appletvos26.5` / `26.5` against `17.0`. The SDK is newer than the minimum OS in every bundle. The real gap runs the other way: a 26.5 SDK running on 27.2 devices (§F.3–F.5).

---

## F. Build / CI / SDK alignment (MANDATORY)

### F.0 How the numbers were obtained

| Run | What | Result |
|---|---|---|
| [45292481632](https://github.com/zjoenguyenz/dabanh-ios-opt/actions/runs/45292481632) | `build-dabanh.yml`, `target=ios`, `main@b800b7b` | **success** · artifact `DaBanh-ipa-run1-20261123` |
| [45292484489](https://github.com/zjoenguyenz/dabanh-ios-opt/actions/runs/45292484489) | `build-dabanh.yml`, `target=tvos`, `main@b800b7b` | **FAILURE**: 16 compile errors (F1) |
| [45293267416](https://github.com/zjoenguyenz/dabanh-ios-opt/actions/runs/45293267416) | `build-dabanh.yml`, `target=tvos`, ref `audit/tvos-last-green` = `0859de6` (iOS 1.20, the newest commit whose shared core still compiles for tvOS) | **success** · artifact `DaBanhTV-ipa-run3-20261123` |
| [45293956534](https://github.com/zjoenguyenz/dabanh-ios-opt/actions/runs/45293956534) | Throwaway `audit-ipa-dissect.yml` on branch `audit/ipa-dissect`. It downloads the two artifacts above by run id, unzips them, and prints `PlistBuddy` keys and the Mach-O `LC_BUILD_VERSION` (`otool -l`). It does not build anything. | **success**: values below |

The tvOS IPA had to come from `0859de6` because HEAD cannot build. The DaBanhTV and DaBanhTVTopShelf target settings (deployment target, SDK selection, Info.plist pairs) are the same at `0859de6` and HEAD: `git diff 0859de6 origin/main -- project.yml` touches only the iOS target. The workflow is also the same (`latest-stable` on `macos-latest`). So the DT/Min values carry over to any HEAD tvOS build once F1 is fixed, as long as the runner keeps the same Xcode. The dissect workflow's source is in Appendix A. The two audit refs could not be deleted from this session (HTTP 403); GitHub keeps the run logs after they are deleted.

### F.1 Dissected values

| Bundle | `MinimumOSVersion` | `DTSDKName` | `DTPlatformVersion` | `DTPlatformBuild` / `DTSDKBuild` | `DTXcode` / `DTXcodeBuild` | `LC_BUILD_VERSION` (minos / sdk) | IPA sha256 (first 16) |
|---|---|---|---|---|---|---|---|
| **DaBanh.app** (iOS 1.35/36) | **17.0** | **iphoneos26.5** | **26.5** | 23F81a / 23F81a | 2660 / 17F113 | IOS 17.0 / 26.5 | `8412088dbb5ea1c0` |
| **DaBanhTV.app** (tvOS 1.3.5/17) | **17.0** | **appletvos26.5** | **26.5** | 23L470 / 23L470 | 2660 / 17F113 | TVOS 17.0 / 26.5 | `9841f04870fb9dee` |
| DaBanhTVTopShelf.appex | 17.0 | appletvos26.5 | 26.5 | 23L470 / 23L470 | 2660 / 17F113 | TVOS 17.0 / 26.5 | (inside the above) |

The runner reported `Xcode 26.6, Build version 17F113` and `Apple Swift version 6.3.3`. The compile lines confirm `-sdk …/iPhoneOS26.5.sdk -target arm64-apple-ios17.0 -swift-version 6` for the DaBanh target (run 45292481632 log). The tvOS log shows `AppleTVOS26.5.sdk`.

**The leader's pre-measurement is verified.** The production 1.35 IPA's `DTSDKName = iphoneos26.5`, `DTXcode = 2660`, `MinimumOSVersion = 17.0` match the fresh copy build exactly.

### F.2 The JTV defect (SDK older than the minimum OS)

- **DaBanh iOS: NOT PRESENT.** SDK 26.5 > MinimumOS 17.0.
- **DaBanhTV tvOS: NOT PRESENT.** SDK 26.5 > MinimumOS 17.0, and the same holds for the TopShelf appex.

JTV broke because its floor (27.0) was above the CI's SDK (26.5). Đá Banh's floor is 9 major versions below its SDK, so this defect cannot occur until someone raises the floor past what the CI's SDK provides.

### F.3 The inverse gap: 17.0 floor, 26.5 SDK, 27.2 devices

What it means, with evidence:

1. **No 27.x API can be called, even behind `if #available(iOS 27, *)`.** The symbols aren't in the 26.5 SDK. Two examples from the iOS & iPadOS 27 and tvOS 27 release notes:
   - SwiftUI's new `AsyncImage` initialisers that take a `URLRequest`.
   - `View.asyncImageURLSession(_:)`.
   These are exactly the knobs that would let iOS solve A3 without a hand-rolled cache. The notes also say `AsyncImage` now caches automatically using HTTP caching. I could **not** verify whether that automatic caching applies to binaries linked against SDK 26.x running on 27.2, so treat it as unverified. Don't rely on it.
2. **The app keeps the 26.x linked-on-or-after behaviour on 27.2 devices.** Items in the 27 notes worded "in apps built with the 27.0 SDKs" don't apply today. Two of them matter to the design gate:
   - **tvOS:** "Buttons on tvOS no longer automatically use the accent color from an app's asset catalog as a label tint color when building against the tvOS 27 SDK." Every TV button goes through `TVCardButtonStyle` (`DaBanhTV/TVMatchViews.swift:198-210`), and most labels set an explicit `.foregroundStyle`. Whether any label still inherits the accent tint can only be settled by a side-by-side on the TV, so this is **PLAUSIBLE design-gate exposure**.
   - **iOS:** the TabView selection rule ("might crash when its selection is set to a hidden or otherwise unavailable tab") and the launch-screen requirement. The iOS target already sets `UILaunchScreen` (`project.yml`), so the launch-screen requirement is met. I did not audit TabView usage against the selection rule.
3. **The 17.0 floor costs nothing today and buys nothing for 27.x.** The floor decides where the binary may install; the SDK decides which APIs exist. Raising the floor to 27.0 would not unlock any 27 API while the CI stays on SDK 26.5. It would create exactly the JTV defect. **Do not raise the floor before the SDK moves.**
4. **Moving to Xcode 27 is a compiler change as well as an SDK change.** Xcode 27 ships Swift 6.4 and turns `@State` into a macro, which stops compiling some initialiser patterns. A grep found 46 `@State` declarations and no `_x = State(…)` / `State(initialValue:)` assignments in scope, so the risk is low, but unverified until a build runs. The iOS target's 7 Swift 6 isolation warnings (C1, C2) may also be promoted to errors by a newer compiler (unverified).

### F.4 Is the unpinned `latest-stable` a risk? Yes, in both directions (F2, CONFIRMED)

- `build-dabanh.yml` uses `runs-on: macos-latest` plus `maxim-lobanov/setup-xcode@v1` with `xcode-version: latest-stable` in all three jobs.
- **Why the SDK is 26.5 and not 27.x:**
  - GitHub moved to one major Xcode per runner image. Xcode 27 lives on the separate `xcode-27` / `xcode-27-xlarge` labels, GA since October 2026.
  - `macos-latest` (macOS 26) defaults to Xcode 26.6 (runner-images issue #14344).
  - So `latest-stable` on `macos-latest` will **never** reach Xcode 27. The CI looks unpinned but is in practice frozen on the 26.x line, while the owner's devices run 27.2.
- **The other direction:** nothing in the workflow pins a specific Xcode. The 26.5→26.6 default change on 2026-07-21 already moved the toolchain under the project unannounced. A future `macos-latest` re-label would move the Swift compiler (6.3→6.4) and the SDK in one silent step, the same shape of surprise as JTV's.
- **Recommendation** (evidence-based, CI-only, no app code):
  - Pin explicitly, for example `xcode-version: '26.6'` on `macos-26`. That changes nothing about today's output, and a manifest diff can confirm the DT values above.
  - Adopt Xcode 27 as a **deliberate, separate step** on the copy: `runs-on: xcode-27` + `xcode-version: '27.x'`.
  - Then re-dissect, diff the DT/LC values, and run the design gate on device. Cover the tvOS button-tint item and every "built with the 27.0 SDK" note listed above.
  - Keep the 17.0 floor.

### F.5 F1: the tvOS target is red at HEAD (P1, CONFIRMED)

Run [45292484489](https://github.com/zjoenguyenz/dabanh-ios-opt/actions/runs/45292484489) fails with 16 errors and `** BUILD FAILED **`. Every error traces to a reference to an iOS-only type:

- `DaBanh/LeagueClassifier.swift:100`, `:238`, `:250`, `:255-259`: `MatchState`, `CompetitionID`, `.champions`, `.europa`, `.conference`.
- `DaBanh/PlaylistStore.swift:253`, `:257`, `:353`, `:361`, `:366-367`, `:379`, `:386`: `MatchStateStore`, `MatchState`, `FixtureEntry`.

These types live in `DaBanh/MatchStateStore.swift` and `DaBanh/MatchViews.swift`, which the DaBanhTV target does not compile (`project.yml`, DaBanhTV `sources`).

- **First bad commit:** `f7477ba` "DaBanh iOS 1.21/22: scoreboard redesign (MatchStateStore …)", 2026-11-19 15:39. This was found by checking every shared-core revision for references to iOS-only types; `0859de6` is the last clean one.
- The production tree is identical, so **no DaBanhTV build of the current production tree is possible.** Every core change since then (FIX-127 aliases, the state layer, DiagLog, the watchdog) has never reached the TV.
- `Crests.swift:86-93` shows the team knew about this constraint ("would have reddened the tvOS build (CB-2)"). `TeamNameNorm.swift` was even added to the tvOS sources. Nobody built tvOS afterwards: the copy had zero runs before this audit.
- **Cost today:** the TV is frozen at 1.3.5, Phase 2's "CI green on both targets" is impossible, and any tvOS fix is blocked.
- **Proposed fix options** (all SHARED-CORE; this is the first Phase-2 item):
  - (a) Move the iOS-only surface out of the shared files: `PlaylistStore.liveStates`, `state(for:)`, `effectiveBucket`, `entries(for:)`, `cupEntries`, `rotatedSources`, and `LeagueClassifier.bucket(of:now:state:)` / `competition(…)`. Put them in an iOS-only `extension PlaylistStore` file and give tvOS the state-less bucket. **Behaviour-neutral on iOS** if done as a pure move; the iOS IPA symbols and DT values should be unchanged.
  - (b) Compile `MatchStateStore.swift` (and `FixtureEntry`) into tvOS. **BEHAVIOUR CHANGE on tvOS:** the TV would start polling five score feeds and its live tab would become state-aware (G1).
  - (c) `#if os(iOS)` fences inside the shared files. This is the quickest, but it buries the platform split inside the protected files.
  - I recommend (a).
- **Risk:** (a) touches the protected `PlaylistStore` and the matching core, but only moves code. It also triggers G3, a design-gate exposure on TV.
- **Verify:**
  - iOS: same symbol table (`nm`) and same DT values; a playlist parse on a fixed `tttt.m3u` gives the same channel, live and hot counts.
  - tvOS: builds; the device shows the same rails as 1.3.5, apart from G3.

### F.6 F3: TopShelf appex version (report only)

The appex reports `CFBundleShortVersionString 1.0` / `CFBundleVersion 1` (dissected) inside DaBanhTV 1.3.5/17. That doesn't matter for an unsigned sideload. The App Store requires an extension's version to match its host app, so this needs fixing before any store submission. Version edits are out of scope for Phase 1 and 2, so it is reported only. Any fix must touch both version pairs per the brief.

---

## A. Performance

There is no device or profiler here, so every timing below is **UNMEASURED**. Call counts are CONFIRMED from the code.

**Proposed measurement method (all A items):**
- Wrap `FixtureGrouping.group`, `PlaylistStore.liveChannels`, `entries(for:)` and `M3UParser.parse` in `os_signpost` intervals.
- Run Instruments → Time Profiler + Points of Interest on the owner's iPhone 18 Pro Max (iOS 27.2) and the Apple TV 4K.
- Script: launch, then 20 remote swipes across a live rail (TV), and scroll the Trực tiếp list plus one state poll (iOS).
- Report the p50 and p95 interval and the main-thread busy % before and after.
- For a deterministic CPU baseline, add an XCTest `measure {}` running `FixtureGrouping.group` over a frozen copy of `dabanh-playlist/tttt.m3u` (465 channels on 2026-11-23).

### A1: tvOS home recomputes fixture grouping ~8× per body, on every focus move (CONFIRMED count, UNMEASURED cost)

- `entries` is a computed property: `TVFixtureEntry.group(tabChannels)` at `DaBanhTV/TVContentView.swift:71`, which calls `FixtureGrouping.group` (`Crests.swift:547`).
- One body evaluation reads `entries` in these places:
  - `countText` (`:182-187`)
  - `scrollBody` `entries.isEmpty` (`:297`)
  - `featuredEntries` for `featuredIDs` (`:312`, which calls `entries` at `:78`)
  - the three filters for `live`, `upcoming` and `rest` (`:313-320`)
  - `featuredEntries` twice more (`:328-329`)
  - That is about 8 groupings. `heroLookup` (`:128-134`) adds 2 more in the `focusedKey` `onChange`.
- On the Trực tiếp tab every `entries` call also recomputes `store.liveChannels` (`:62`), which buckets and sorts every channel.
- The body reads `focusedKey` (pills, hero and cards), so **every remote swipe** re-runs all of it.
- Each grouping normalises every channel's team names: `pairKey`, then `MatchTitle.clean`, `Crests.teams` and 2× `TeamNameNorm.groupKey`. That is 4 regex compilations per channel (A5) plus about 20 string passes.
- **Cost:** CPU and allocations on the main thread per swipe, scaling with the tab's channel count. UNMEASURED.
- **Proposed change:**
  - Compute the grouped `entries` once per data change, either as `@State` updated in the existing `onChange(of: store.channels / hotChannels / selectedLeague)` hooks (`:122-124`) or memoised in the store keyed by a publish revision.
  - `body`, `heroLookup` and `countText` read the cache.
- **Risk:** low. Grouping is a pure function of (channels, now). The one subtlety is `now`: the live-merge window (`Crests.swift:529-532`) and the bucket split use `Date()`, so the cache must also refresh on a minute tick or the live/upcoming split goes stale. That makes it BEHAVIOUR-sensitive and must be handled; otherwise it is NONE.
- **Verify:** snapshot the `entries` ids and rail membership for the same frozen playlist and a fixed clock before and after (identical); device: rails and hero follow focus exactly as before.

### A2: iOS board models recomputed per access; O(channels × states) on the main actor (CONFIRMED graph, UNMEASURED cost)

- `PlaylistStore.liveChannels` (`PlaylistStore.swift:336-348`) calls `effectiveBucket` (`:360-362`) for each of `channels + hotChannels`. That calls `MatchStateStore.state(for:)` (`MatchStateStore.swift:1261`): one `MatchMatching.pairKey` (a normalisation), a linear `states.filter` (`:1265`), and pool and staleness passes.
- Then `entries(for:)` (`PlaylistStore.swift:366-374`) groups and calls `state(for:)` again per entry.
- Read sites, each a full recompute:
  - `ContentView.swift:89` (`featuredEntries`; it is read again at `:243` and `:262` for preresolve, each calling `sources(for:)`, which groups **all** channels at `PlaylistStore.swift:611`)
  - `ContentView.swift:777`, `:973`, `:1058`, `:1374`
  - `MUHomeView.swift:198`, `:211`, `:216`
  - `competingChannels(for:)` at `ContentView.swift:100`, which groups the whole list per call
  - `MUBoardView.swift:26` and `MatchViews.swift:1056` (`sources(for:)` per board or row, read twice in the `MUBoardView` body, `:51` and `:53`)
- **Re-render storm:** `PlaylistStore` forwards every `MatchStateStore.objectWillChange` (`PlaylistStore.swift:257-259`). `revision` is bumped once per cycle (`MatchStateStore.swift:1208`) plus once per FotMob details fetch, up to 4 more (`:1252`, `:1452`). The cycle runs every 60 s while anything is live. So up to about 5 whole-tree invalidations per minute, each paying the recompute above.
- **Proposed change:**
  - Memoise `liveChannels`, `entries(for:)` results and the grouping per (publish revision, state revision, minute tick).
  - Coalesce the details bumps into one `revision += 1` per cycle.
- **Risk:** low–medium. PlaylistStore and MatchStateStore are **PROTECTED**, and coalescing delays a scorer appearing by at most the rest of the same cycle (a few seconds, **BEHAVIOUR CHANGE (timing)**).
- **Verify:** signpost counts per poll cycle; board snapshots identical on a recorded state set.

### A3: iOS crest/logo images have no decoded-image cache (CONFIRMED, UNMEASURED)

- `CrestImg` (`DaBanh/MatchViews.swift:499-506`) and the channel logo (`:430`) use bare `AsyncImage`, as do the MU views (`MUHomeView.swift:253`, `:362`, `:879`; `MUMatchDetailView.swift:266`).
- tvOS 1.3.5 found exactly this (`TVMatchViews.swift:240-252` explains the flicker and re-download) and added `TVImageCache`.
- **Cost:** a re-fetch and re-decode per re-render or recycle, and placeholder flicker. UNMEASURED.
- **Proposed change:** share one image cache across both apps (a new shared file, **SHARED-CORE**). Rendering is unchanged.
- **Risk:** low. Design-gate check: the placeholder must look identical.
- **Verify:** Charles/URLSession metrics showing request count per crest per session before and after; side-by-side screenshots.
- Note F.3-1: the SDK-27 `AsyncImage` cache knobs are not available on SDK 26.5.

### A4: `Channel.id` is a fresh UUID per parse and is part of `Equatable` (CONFIRMED; effect PLAUSIBLE)

- `Channel.swift:5-6` (`struct Channel: Identifiable, Equatable { let id = UUID() … }`).
- Every refresh re-parses, so every `ForEach(…)` keyed by `Channel.id` re-identifies all rows: `MatchViews.swift:1014`, `TVContentView.swift:386`.
- `onChange(of: store.channels)` (`TVContentView.swift:123-124`) fires on every refresh even when the list content is identical.
- On tvOS a re-identified focused view **may** lose focus mid-refresh: PLAUSIBLE, not observed.
- **Proposed change:** derive a stable id, such as the URL (already the de-dup key everywhere, e.g. `PlaylistStore.swift:343`).
- **Risk:** medium. **SHARED-CORE**. Duplicate URLs within one list would collide (the lists are de-duplicated by URL on most paths, but not `rawChannels`). Treat it as **BEHAVIOUR CHANGE (identity)** until proven neutral.
- **Verify:** on TV, focus a rail card, force a refresh, and check focus stays.

### A5: regex compilation inside hot functions (CONFIRMED, UNMEASURED)

- `MatchTitle.clean` compiles a regex per call (`Crests.swift:382`).
- `Crests.teams(ofCleanTitle:)` compiles 3 per call (`:340-346`).
- `MatchTitle.fixtureKey` compiles 2 (`:399`, `:403`).
- These run per channel inside every A1 and A2 grouping. The parser files already use static precompiled regexes (`M3UParser.swift:12-13`, `:86`, `:138`), so the house pattern exists.
- **Proposed change:** hoist them into `static let`.
- **Risk:** minimal (same patterns, same options). The file is **PROTECTED** (Crests) and **SHARED-CORE**.
- **Verify:** unit-equality of `clean`, `teams` and `fixtureKey` over all 465 + 238 titles from both mirror playlists; XCTest `measure`.

### A6: main-actor parse and validation work (CONFIRMED, UNMEASURED)

- `PlaylistStore` is `@MainActor`. `fetchPlaylist` parses on it after the network await (`PlaylistStore.swift:520-526`), and `loadCache` parses two files at launch (`:424-441`).
- `M3UParser.parse` runs 2 regexes and ~120 `contains` keyword checks per entry (`LeagueClassifier.swift:35-43`).
- With 465 + 238 entries this is probably small. Measure it before acting (A method above).

---

## B. Memory and lifecycle

### B1: the persisted invalid-URL set never shrinks (CONFIRMED growth; size UNMEASURED) · SHARED-CORE · PROTECTED

- `probeWave` does `invalidURLs.subtract(validNow); invalidURLs.formUnion(invalidNow)` (`PlaylistStore.swift:748-750`).
- Only URLs in the *current* pool can leave the set. A dead signed URL that drops out of the playlist stays forever. The only reset is the manual "Kiểm tra link" button (`:696-700`).
- The set is decoded at every launch (`:308-312`) and re-encoded whenever it changes (`:314-319`). Both apps run this.
- **Proposed change:** after each pass, intersect `invalidURLs` with the URLs of `rawChannels + rawHotChannels`. **BEHAVIOUR: NONE** for any URL still in the list.
- **Verify:** file size across a day; the hidden-link count equals before for the current list.

### B2: un-invalidated URLSession per validation wave (CONFIRMED; impact UNMEASURED) · SHARED-CORE · PROTECTED

- `PlaylistStore.swift:723-725` creates an ephemeral session per wave (2 per pass, a pass after every load and refresh) and never calls `finishTasksAndInvalidate()`.
- Every other per-call session in scope invalidates (`PlayerView.swift:2107`, `:2178`, `:2649`; `MatchStateStore.swift:591`, `:744`, `:879`, `:946`, `:1040`; `NuvioAddon.swift:435`, `:471`).
- **Fix:** `defer { session.finishTasksAndInvalidate() }`. BEHAVIOUR: NONE.
- **Verify:** Instruments Allocations: `__NSURLSessionLocal` count stays flat across 5 refreshes.

### B3: per-app-run caches without bounds (CONFIRMED, small)

- `MatchStateStore.detailCache` (`:1125`, keyed by FotMob id) and the learned-logo registry (`Crests.swift:124`) are unbounded per run. Each is bounded in practice by the day's fixtures.
- `SourcePrewarmStore.states` (`PlayerView.swift:2567`) is unbounded too, and that is the E1 defect.
- No action needed beyond E1 and E2.

### B4: playback teardown (no defect found)

- tvOS `dismantleUIViewController` invalidates KVO, removes the observer, pauses, and nils the player (`TVPlayerView.swift:109-113`, `:155-162`).
- iOS `HighlightMP4View.Coordinator.teardown` does the same (`HighlightViews.swift:1030-1037`).
- `MatchStateStore.start()` captures `self` strongly for the loop's life (`:1128-1130`: `[weak self]`, then an immediate `guard let self`). The store lives for the app's lifetime, so this is harmless; noted for accuracy.

---

## C. Concurrency correctness

**Mode check:** the DaBanh target compiles with `-swift-version 6` (CI log). The DaBanhTV target inherits the project base `SWIFT_VERSION 5.9` (`project.yml:9`), so it gets no strict checking.

### C1: KVO callbacks touching coordinator state off the main thread (CONFIRMED)

- **iOS, Highlights:** `HighlightMP4View.Coordinator.watch` calls `self?.report()` straight from the `item.observe(\.status)` handler (`HighlightViews.swift:1014-1016`).
  - `report()` mutates `reported` and calls `onFail()` (`:1025-1028`), which mutates SwiftUI state.
  - The KVO handler's thread is not guaranteed to be main, and CI flags it: `HighlightViews.swift:1015:45` and `:1020:17`, "capture of 'self' with non-Sendable type … in a '@Sendable' closure".
  - The file's own comment ("AVKit callbacks arrive on the main thread", `:1001-1002`) is an assumption, not a guarantee.
  - The iOS live player shows the correct pattern: read the value, then `DispatchQueue.main.async` (`PlayerView.swift:1004-1011`).
- **tvOS:** `TVPlayerView.Coordinator.fireOnce` reads and writes `fired` from the KVO thread (`TVPlayerView.swift:131-138`) and from the main-queue notification (`:142-146`), so the at-most-once guarantee (`:117`) is racy. This compiles silently in 5.9 mode.
- **Proposed change:** hop to main before touching coordinator state, mirroring `PlayerView.swift:1004-1011`. BEHAVIOUR: NONE (same callbacks, same once-only).
- **Verify:** Thread Sanitizer run of a failing MP4 source and a failing TV source; the CI warning count drops by 2.

### C2: Swift 6 isolation diagnostics accepted as warnings (CONFIRMED) · PROTECTED

- CI run 45292481632 reports 7 isolation warnings in `PlayerView.swift`: `:1041` (`noteFailure` from a `NotificationCenter` block), `:1161` (`stallWatchdogFired` from a `Timer` block), and `:1370-1372` (`player`, `controlsVisible`, `animate`, `chromeFadeDuration`, `alpha` from a `Timer` block).
- The blocks do run on main: `queue: .main` / a run-loop timer. So this is a typing hole, not a live race.
- Rule 6 forbids weakening checking. The non-weakening fix is to state the isolation, for example by wrapping the bodies in `MainActor.assumeIsolated { }`. I believe that API is available on iOS 17, but I have **not verified it against the 26.5 SDK headers here.**
- BEHAVIOUR: NONE.
- **Verify:** the CI warning list is empty for these lines.

### C3: other notes (no defect found)

- `DiagLog`'s watchdog state is lock-guarded (`PlaylistStore.swift:80-86`).
- `Crests.MapBox` (`Crests.swift:45-68`), `MUWallpapers.ListBox` and `NuvioStreamHeaders.Table` (`NuvioAddon.swift:549-584`) are honest `@unchecked Sendable` lock boxes.
- I found no `withChecked*Continuation` in scope.
- Cancellation: `probeWave` task groups ignore cancellation, but they are never cancelled either; no hole found.

---

## D. Code health

- **D1, oversized protected files:**
  - `PlayerView.swift` is 3,119 lines and holds 11 types, including `LivePlayerViewController` (~700 lines), `MatchPlaySession`, `SourcePrewarmStore` and `ResolveHub`.
  - `ContentView.swift` is 1,536 lines and `MatchStateStore.swift` 1,462.
  - Change risk is high because unrelated types share one review surface. Proposal: split files without code changes (a pure move, behaviour NONE), but only after F1 is fixed, so CI can prove each move.
- **D2, duplicated constants and logic:**
  - `TVContentView.enTabValue = -2` (`:40`) duplicates `LeagueClassifier.enTabValue` (`LeagueClassifier.swift:276`).
  - `TVEnglish.groupName` / `isEnglish` (`TVMatchViews.swift:108-111`) duplicates `LeagueClassifier.enGroupName` / `isEN` (`:271`, `:284-286`).
  - The year-wrap rule appears twice (`M3UParser.swift:150-163` and `PlaylistStore.swift:818-832`), and both copies carry H1.
  - `DiagLog` lives inside `PlaylistStore.swift` (`:40-213`).
- **D3, stale comments that mislead reviewers:**
  - `TVMatchViews.swift:15-17` says the shared engine files are "untouched"; they changed heavily (G).
  - `TVMatchViews.swift:246-247` says "tvOS 26/27 changed nothing in AsyncImage"; the tvOS 27 notes list automatic `AsyncImage` caching (F.3-1).
  - `PlaylistStore.swift:236-237` says "The iOS UI does not render this tab yet"; iOS does (`ContentView.swift:777`).
- **D4, dead code:** a spot-check of 9 candidates found no unreferenced functions. `DaBanh/LinkProbe.swift` is compiled only by the out-of-scope TV and JoeTV targets (`project.yml:142`, `:170`); it is noted, not audited.

---

## E. Playback-path and matching-path risks (report only, rule 3)

### E1: prewarm "dead" verdicts never expire (CONFIRMED) · PROTECTED · BEHAVIOUR CHANGE

- FIX-130 (P-4) made hub verdicts expire after 30 s (`PlayerView.swift:2790-2806`).
- But every reader ORs in `SourcePrewarmStore.isDead` (`:2574`), whose `.dead` state is written at `:2677` and never cleared, except by a cancelled in-flight warm (`:2654`, `:2709`).
- The readers are `isKnownDead` (`:2038-2041`, which filters the ladder head), the MU detail (`MUMatchDetailView.swift:57-60`), `MUBoardView.swift:110` and `MatchViews.swift:1060`.
- **Cost:** a source that failed one prewarm stays filtered for the whole app run. That is exactly what P-4 says it removed.
- **Proposed change:** give `SourcePrewarmStore` the same 30 s TTL (stamp `.dead(at:)`) or route its verdicts into the hub's map.
- **Risk:** a retestable dead host costs one extra pre-flight per tap, which is what the web standard already does.
- **Verify:** DiagLog on device: a source prewarmed dead becomes eligible again after 30 s (a "prewarm: source proven dead" line, then a later head selection of the same host).

### E2: FotMob incidents frozen for live fixtures (CONFIRMED) · PROTECTED · BEHAVIOUR CHANGE

- `fetchFotMob` builds states with `goals: [], cards: []` (`MatchStateStore.swift:742` ff.). Details are fetched only when `state.goals.isEmpty && state.cards.isEmpty && detailCache[id] == nil` (`:1236-1238`, `:1436-1438`).
- After the first fetch, each cycle copies the cached incidents back in (`:1182-1188`), so neither gate ever passes again for that id.
- **Cost:** for a live fixture whose winner is FotMob (no ESPN coverage), the score updates every 60 s but the scorer and card list stays at its first snapshot.
- **Proposed change:** re-fetch details for **live** FotMob states when the score changed since the cached fetch (or every N minutes), still capped at 4 per pass.
- **Risk:** more FotMob calls (bounded).
- **Verify:** a recorded live FotMob fixture with 2 goals 20 min apart: the board shows both scorers.

### E3: the launch validation pass swallows the post-refresh pass (CONFIRMED) · PROTECTED · SHARED-CORE · BEHAVIOUR CHANGE

- `start()` schedules validation on the cached list (`PlaylistStore.swift:398`), then `await refresh()`. That ends with `scheduleValidation()` (`:506`), which returns immediately while the first pass is still `validating` (`:689`).
- The first pass's pool was taken from the *old* `rawChannels` (`:704`). A pass of about 470 probes at 6–8 s timeouts easily outlasts a 20 s fetch, though that is not measured.
- So the fresh list's new URLs go unprobed until the next 15-minute refresh.
- **Proposed change:** if a pass is running when a new list lands, mark it dirty and re-run once at the end.
- **Verify:** DiagLog the pool size and revision per pass on device.

### E4: header injection through an undocumented AVURLAsset option key (PLAUSIBLE) · PROTECTED

- Both players pass `["AVURLAssetHTTPHeaderFieldsKey": headers]` (`PlayerView.swift:712`, `TVPlayerView.swift:104`).
- That string is not a documented AVFoundation option. I cannot read the 26.5 SDK headers from this host, so I **cannot verify** that it appears in them.
- It is device-proven today, but Apple can drop an undocumented key in any OS without notice, and the Referer-dependent CDNs would then 403.
- I believe a public `AVURLAssetHTTPUserAgentKey` exists (iOS/tvOS 16+) for the User-Agent half only. That is **unverified here**, so check the SDK before proposing it.
- **No change proposed.** Flag this for the owner as a tracked platform risk, and add a DiagLog witness when a Referer-bearing source fails with 403.

### E5: tvOS player has no load timeout (CONFIRMED code; PLAUSIBLE trigger)

- `TVPlayerView` advances only on `.failed` or `FailedToPlayToEndTime` (`TVPlayerView.swift:130-147`).
- A stream that stalls before `.readyToPlay` (a TCP-open but silent CDN) leaves "Đang tải…" up forever (`:27-43`). iOS has a 6 s/10 s stall watchdog (`PlayerView.swift:1155-1162`).
- This is part of G2. Porting the watchdog is a **BEHAVIOUR CHANGE** on tvOS.

---

## G. iOS ↔ tvOS drift

**Ground truth.** The brief says tvOS implements its own fixture-matching and state logic inside `TVContentView` / `TVMatchViews`. **It does not.** tvOS has no state layer at all, and it groups through the shared `FixtureGrouping` (`TVMatchViews.swift:94-98`). The drift is between iOS HEAD and a TV app frozen on an older shared core (F1).

- Shared-core change since tvOS 1.3.5 (`b9bfe60`): `git diff --stat b9bfe60 origin/main` over the shared files reports **1,844 insertions / 35 deletions**. That covers `Channel`, `Crests`, `LeagueClassifier`, `PlaylistStore`, `crests.json` (+236 lines), and the whole new `TeamNameNorm` (908 lines).
- None of it has shipped to the TV.

| # | Divergence (provable from code) | iOS (HEAD) | tvOS (1.3.5 / last-green) | Newer / correct |
|---|---|---|---|---|
| G1 | Live classification | State-aware `effectiveBucket` (`PlaylistStore.swift:360`, `LeagueClassifier.swift:100-112`): FT leaves live at the next poll, ET and pens stay live | Wall clock only (`TVContentView.swift:313-320`, `:543`; `TVMatchViews.swift:145`): an FT match shows LIVE until +125 min, and a late or ET match drops out at +125 | iOS |
| G1b | Live minute | Feed minute (state) | `now − kickoff` (`TVMatchViews.swift:43-45`): counts through half-time, so by full time it reads ~15′ ahead | iOS |
| G2 | Tap-to-play | ResolveHub, pre-flight, race ordering, prewarm, stall watchdog (`PlayerView.swift`) | Plain sequential ladder, advance on `.failed` only, no timeout (`TVPlayerView.swift:52-59`). Commit `1a2d8f9` "FIX-128/128B: tvOS tap-to-play port" changed only `DaBanh/ContentView.swift`, `DaBanh/PlayerView.swift` and `project.yml` (`git show --stat`) | iOS. The "port" never happened |
| G3 | League tabs | `League` = NHA, TBN, Ý, Pháp, **Đức**, **Cúp châu Âu**, Khác (`Channel.swift:43-56`) | 1.3.5 had 5 cases (`b9bfe60:Channel.swift`); TV renders `ForEach(League.allCases)` in row one (`TVContentView.swift:202`), which `:191-194` already calls full at ~1,800pt | On repair, the TV gains 2 pills (Đức, Cúp châu Âu). No channel has rank 6, so the Cups pill always shows 0 trận on TV. **Design gate:** decide before F1 lands |
| G3b | Team-name grouping | FIX-127 shared aliases (`TeamNameNorm.swift`) | 1.3.5: the pre-FIX-127 reduction, so the duplicate-fixture disease the alias table cures is still live on TV | iOS |
| G4 | Diagnostics | DiagLog file + export (`ContentView.swift:1506-1517`) | No DiagLog at 1.3.5 or 0859de6 (0 references). At HEAD, tvOS would compile DiagLog, but there is no export UI and the boot header reads "launch Da Banh iOS" on TV (`PlaylistStore.swift:70`) | iOS (and fix the header) |
| G5 | Image loading | Bare `AsyncImage` (A3) | `TVImageCache` NSCache + 32/128 MB URLCache (`TVMatchViews.swift:253-268`) | tvOS |
| G6 | EN block | `LeagueClassifier.isEN` / `enTabValue` | `TVEnglish.isEnglish` / own `enTabValue` (D2). Same values today | Equal; duplicated |
| G7 | Initial tab, playlist order, EN merge, link validation | Shared `PlaylistStore` | Same code, but the TV runs the 2026-11-18 copy, without the state-aware live list | iOS |

**Recommendation:** repair F1 with option (a), keeping tvOS state-less, so the TV gets the shared fixes (G3b, G7, DiagLog) without G1/G2 behaviour changes. Then approve G1, G2 and G3 one at a time as explicit BEHAVIOUR CHANGE / design-gate items.

---

## H. Data contracts

### Method

- The app's parsing rules were ported to Python (Appendix B) and run against the real producer files.
- Inputs:
  - `dabanh-playlist@b03118d` (2026-11-23 17:50 CST): `tttt.m3u` (465 entries), `582049.m3u` (238), `crests.json` (1,589 keys)
  - this repo's `highlights.json` (20 matches)
  - the `highlights.json` history (47 revisions)

### H1: kickoff and `Updated` stamp year-wrap (CONFIRMED by the ported rule) · PROTECTED · SHARED-CORE · BEHAVIOUR CHANGE

**The rule.** Both parsers use the current ICT year, then add 365 days if the result is more than 30 days in the past: `M3UParser.swift:150-163` and `PlaylistStore.swift:818-832`.

**Ported runs** (`now` given in ICT):

| Input | now | Parsed | Bucket / effect |
|---|---|---|---|
| `23:00 31/12 ⚽ A vs B` | 2027-01-01 00:30 | **2027-12-31 23:00** | 1 (upcoming). The match is live, but it leaves Trực tiếp at midnight and the board loses it |
| `20:00 02/01 …` | 2028-12-30 12:00 (leap year) | **2029-01-01 20:00** | One day early, so the wrong stamp date key and no state binding (`MatchStateStore.swift:1264`) |
| `20:00 02/01 …` | 2026-12-30 12:00 | 2027-01-02 20:00 | Correct |
| `Updated 14/10 06:36` (40 days old) | 2026-11-23 12:00 | **2027-10-14** | Age −467,676 min, so it reads *fresh*. A dead mirror never yields to the origin |
| `Updated 31/12 23:50` | 2027-01-01 00:10 | **2027-12-31 23:50** | Reads fresh. Harmless here, but the same inversion |

- **Cost:** the next trigger is New Year's Eve 2026, 38 days away. Premier League fixtures traditionally fall on 30/12–1/1.
- **Proposed change:** pick the year in {y−1, y, y+1} whose date is nearest to `now`, in both parsers via one shared helper (closing D2).
- **Verify:** add the 5 rows above as unit cases; re-run the full playlist corpus and expect identical kickoffs for every current title. They are all 2026 today: 448 of 448 stamped titles in `tttt.m3u`.

### H2: the `Updated` pseudo-entry becomes a channel (CONFIRMED by the ported parser)

- `tttt.m3u` line 2 is `#EXTINF:-1 group-title="Updated 24/11 06:36", Updated 24/11 06:36`, followed by `https://freem3u.xyz/static/no-signal/low.m3u8`.
- The parser accepts it as a channel (title "Updated 24/11 06:36", no kickoff, rank 9). The iOS Khác tab filters only EN (`ContentView.swift:958-961`), and tvOS filters only EN (`TVContentView.swift:68`). So a "channel" that plays a no-signal slate is listed in Khác on both apps, and the link validator probes it every pass.
- **Contract gap:** nothing tells the app this entry is metadata. It could be intentional (an Android-style freshness row), so ask the owner before changing it.
- **If unintended:** skip entries whose group-title matches the `UpdatedStamp` regex. That is a **BEHAVIOUR CHANGE** (one row disappears).

### H3: the deployed collector is not the in-repo collector (CONFIRMED)

- `highlights.json` revisions before `6d31df2` (2026-11-22 17:13 CST) carry 17-character `matchDate` (`2026-11-22T20:00Z`). From `6d31df2` on they carry 20-character dates with seconds (`…T20:00:00Z`).
- `tools/highlight_collector.py` last changed at `08e69e6` (14:31 the same day). It still writes `ev["date"].replace("+00:00", "Z")` (`:148`), which is the seconds-less ESPN shape behind the 1.34 empty-tab incident.
- So the producer of record runs code that is not in this repo.
- The app has tolerated both shapes since 1.35: the `HighlightDates.parse` fallback cascade (`Highlights.swift:205-225`), with `yyyy-MM-dd'T'HH:mmXXX` for the short form. So there is no outage today.
- **Risk:** re-running the in-repo tool reintroduces the short shape on any app older than 1.35.
- **Proposed change:** commit the deployed collector and document the contract (`matchDate`/`addedAt`: RFC 3339 UTC, seconds required).

### H4: contracts that hold (CONFIRMED by the ported checks)

- **`crests.json`:** a `[String: String]` object, 1,589 entries (≥100 required, `Crests.swift:22-27`). 0 non-string values, 0 keys not already in `Crests.norm` form, 0 non-https values. The bundled copy (695) is a strict subset (894 remote-only keys; 7 shared keys with updated URLs).
- **M3U:**
  - Every stamped title (448 of 465 in `tttt.m3u`; 122 of 238 in `582049.m3u`) yields an A-vs-B pair.
  - No status emoji survive `MatchTitle.clean`.
  - The EN block (16) and the 🔥 Hot group (2) are present, as `isEN` and `setHot` expect.
  - `#EXTVLCOPT` lines follow their `#EXTINF`, as the parser requires (`M3UParser.swift:31-36`).
  - 66 and 103 `.flv` entries respectively are hidden by design (`PlaylistStore.swift:584-587`).
  - `582049.m3u` has no `Updated` stamp, so `UpdatedStamp.parse` returns nil there, which reads as "unknown" (only the mirror file is stamp-gated).
- **`highlights.json`:** `version 1`; 20 matches; all `matchDate`/`addedAt` are 20-character UTC, so they parse on the first `ISO8601DateFormatter` path. Competitions are a subset of the decoder's enum. Kinds are `youtube` and `espn-mp4` only. `espnEventId` is always a string. 2 matches have no sources and are hidden by design (`Highlights.swift:310`).
- **Unverified:** whether ESPN's scoreboard `date` still omits seconds. `site.api.espn.com` is blocked from this host. The score feeds (ESPN, FotMob, Flashscore, LiveScore) were not contract-tested for the same reason.

---

## Rule 5: secrets

- **No P1 found.** I grepped for key, token, secret, password, bearer and authorization patterns and listed every hard-coded host in the in-scope Swift.
- The one fixed header value is Flashscore's `x-fsign: SW9D1eZo` (`MatchStateStore.swift:949`). It is the public constant their own web client sends, not an account credential (the code comment at `:934` says "no account/cookies"). I'm reporting it so the owner can decide; it is not P1.
- The DiagLog redaction rules hold at the sites I checked: `diagSourceTag` logs title and host only (`PlayerView.swift:620-627`).

## Rule 6: conflicts found (report only)

- tvOS draws its own focus chrome: `.focusEffectDisabled()` plus a hand-drawn cyan ring (`TVMatchViews.swift:174-236`, `TVContentView.swift:464-472`). This contradicts rule 6 ("do not hand-draw focus chrome"), but it is owner-approved under the design gate (`TVMatchViews.swift:185-197`).
- Changing it would be a UI change, so nothing is proposed. The owner should rule on which law wins.
- Deprecated API: `AVAsset.mediaSelectionGroup(forMediaCharacteristic:)` (`MUPlayerChrome.swift:1100`, `:1113`; CI warning "deprecated in iOS 16.0: Use loadMediaSelectionGroup(for:)"). It is fixable without a UI change (async load before building the sheet). It's a candidate PR; the risk is the sheet appearing a few ms later (BEHAVIOUR: timing).

---

## Proposed Phase 2 order (subject to approval)

1. **F1** (option a), plus a decision on G3 (whether the pills appear on TV). This unblocks everything.
2. **F2**: pin `macos-26` + Xcode 26.6 in the copy's workflow. The DT values must be identical.
3. **H1** (before 31/12), as an approved BEHAVIOUR CHANGE.
4. **B2, B1, C1, A5**: small, NONE-behaviour diffs.
5. **E1, E3, E2**: each needs a risk analysis and an approved BEHAVIOUR CHANGE.
6. **A1, A2**: memoisation, with the before/after measurement method above.
7. **A3, A4, C2, H3, deprecated API**.
8. Separately, an **Xcode 27 / SDK 27 adoption trial** on the copy, with a device design-gate review (F.3–F.4).
9. **G1, G2** (TV state layer, TV player port) only as explicitly approved behaviour changes.

---

## Appendix A: the throwaway dissection workflow (branch `audit/ipa-dissect`; owner to delete)

```yaml
name: AUDIT — dissect DaBanh IPAs (throwaway, Phase 1 evidence)
on:
  push:
    branches: [audit/ipa-dissect]
permissions:
  contents: read
  actions: read
jobs:
  dissect:
    runs-on: macos-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - { label: "DaBanh iOS (main 1.35/36)", run_id: "45292481632", app: DaBanh.app, exe: DaBanh }
          - { label: "DaBanhTV tvOS (last-green 0859de6, 1.3.5/17)", run_id: "45293267416", app: DaBanhTV.app, exe: DaBanhTV }
    steps:
      - uses: actions/download-artifact@v4
        with: { run-id: "${{ matrix.run_id }}", github-token: "${{ secrets.GITHUB_TOKEN }}", path: art }
      - name: Dissect
        run: |
          IPA="$(find art -name '*.ipa' | head -1)"; shasum -a 256 "$IPA"
          unzip -q "$IPA" -d x
          for P in "x/Payload/${{ matrix.app }}/Info.plist" x/Payload/${{ matrix.app }}/PlugIns/*.appex/Info.plist; do
            for K in CFBundleIdentifier CFBundleShortVersionString CFBundleVersion MinimumOSVersion DTSDKName \
                     DTPlatformName DTPlatformVersion DTPlatformBuild DTSDKBuild DTXcode DTXcodeBuild; do
              echo "$K = $(/usr/libexec/PlistBuddy -c "Print :$K" "$P")"; done; done
          # main binary + appex binaries
          otool -l "x/Payload/${{ matrix.app }}/${{ matrix.exe }}" | grep -A6 LC_BUILD_VERSION
```

(This is abridged. The committed version also loops over every appex Mach-O and prints `xcodebuild -version`. The full output is in run 45293956534.)

## Appendix B: the Python port used for §H

<details><summary>Port of the M3UParser, KickoffParser, UpdatedStamp, LeagueClassifier.bucket, MatchTitle.clean, Crests.teams and Crests.norm rules</summary>

```python
import re, unicodedata
from datetime import datetime, timedelta, timezone
ICT = timezone(timedelta(hours=7))
LOGO = re.compile(r'tvg-logo="([^"]*)"'); GROUP = re.compile(r'group-title="([^"]*)"')
def title_of(extinf):                      # M3UParser.title(of:)  M3UParser.swift:65-76
    q = False
    for i, ch in enumerate(extinf):
        if ch == '"': q = not q
        elif ch == ',' and not q: return extinf[i+1:].strip(' ')
    return "Kênh"
def ict_rule(day, month, hour, minute, now):   # M3UParser.swift:150-163 == PlaylistStore.swift:818-832
    d = datetime(now.astimezone(ICT).year, month, day, hour, minute, tzinfo=ICT)
    if (d - now).total_seconds() < -30*24*3600: d += timedelta(days=365)
    return d
KO = re.compile(r'(\d{1,2}):(\d{2})\s+(\d{1,2})/(\d{1,2})')
def kickoff(title, now):
    m = KO.search(title)
    if not m: return None
    h, mi, d, mo = map(int, m.groups()); return ict_rule(d, mo, h, mi, now)
UPD = re.compile(r'Updated\s+(\d{1,2})/(\d{1,2})\s+(\d{1,2}):(\d{2})')
def updated_stamp(text, now):
    m = UPD.search(text[:512])
    if not m: return None
    d, mo, h, mi = map(int, m.groups()); return ict_rule(d, mo, h, mi, now)
def parse_m3u(text, now):                  # M3UParser.parse  M3UParser.swift:15-52
    out, pend, logo, group = [], None, "", ""
    for raw in text.split("\n"):
        line = raw.strip()
        if line.startswith("#EXTINF"):
            m = LOGO.search(line); logo = m.group(1) if m else ""
            m = GROUP.search(line); group = (m.group(1) if m else "").strip(' ')
            pend = title_of(line)
        elif line.startswith("http") and pend is not None:
            out.append(dict(title=pend, logo=logo, group=group or "Kênh", url=line, kickoff=kickoff(pend, now)))
            pend = None
    return out
def bucket(ch, now, rank=9):               # LeagueClassifier.bucket(of:now:)  :47-53
    k = ch["kickoff"]
    if k is None: return 3
    s = (now - k).total_seconds()
    if -600 <= s <= 125*60: return 0
    return (1 if rank < 9 else 2) if s < -600 else 3
TIME = re.compile(r'\d{1,2}:\d{2}\s+\d{1,2}/\d{1,2}')
def clean(t):                              # MatchTitle.clean  Crests.swift:380-392
    m = TIME.search(t)
    if m and len(t[:m.start()]) < 12: t = t[m.end():]
    for e in ("⚽", "🟢", "🔴"): t = t.replace(e, "")
    return t.strip(' ')
VS = re.compile(r'\s+vs\.?\s+', re.I)
def teams(t):                              # Crests.teams(ofCleanTitle:)  Crests.swift:338-361
    s = re.sub(r'\([^)]*\)', ' ', re.sub(r'\[[^\]]*\]', ' ', t)); m = VS.search(s)
    if not m: return None
    side = lambda x: (x[:x.index("•")] if "•" in x else x).strip(' ')
    a, b = side(s[:m.start()]), side(s[m.end():]); return (a, b) if a and b else None
def norm(s):                               # Crests.norm  Crests.swift:186-202
    t = "".join(c for c in unicodedata.normalize("NFD", s.lower()) if unicodedata.category(c) != "Mn").replace("đ", "d")
    out, sp = [], True
    for c in t:
        if "a" <= c <= "z" or "0" <= c <= "9": out.append(c); sp = False
        elif not sp: out.append(" "); sp = True
    return "".join(out).strip()
```

Known port deviations:
- Swift's `Calendar.date(from:)` rolls invalid dates (31/02) forward, whereas the port drops them. No such title exists in the corpus.
- Swift counts `Character`s where Python counts code points in the `< 12` prefix check. The corpus prefixes are ASCII plus one emoji, so the result is the same.

</details>
