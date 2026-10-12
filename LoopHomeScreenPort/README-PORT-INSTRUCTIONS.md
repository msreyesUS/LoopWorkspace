# Port Marco's Loop home-screen redesign onto Loop 3.14.9

Created 2026-10-12 from the 3.14.2 workspace (`Loop-260809-1949`). This folder is
self-contained: it holds the exact patches of every custom change plus the design
spec, so the port can be done even if the old workspace is gone. It is also
committed to `https://github.com/msreyesUS/LoopWorkspace` (branch `main`, folder
`LoopHomeScreenPort/`).

## How to use this (Marco)

1. Build stock Loop 3.14.9 with the Loop build script (Build Loop, main branch) and
   run it on the phone first. Ideally run it stock through a pod change to confirm
   the fault 203 fix before customizing.
   - If the stock build crashes at launch on iOS 27 with "UIScene life cycle is
     required", that is expected with Xcode 27; the port below includes the fix.
2. Copy this whole `LoopHomeScreenPort` folder into the NEW workspace folder (or
   anywhere; just adjust the path below).
3. Open Claude Code inside the NEW workspace folder and paste:

   > Read `LoopHomeScreenPort/README-PORT-INSTRUCTIONS.md` and carry out the
   > "Instructions for Claude" section. Verify on my iPhone 18 Pro Max when done.

4. Keep the old workspace until the new build has been running happily for a few
   days.

## Instructions for Claude

You are in a fresh LoopWorkspace at Loop 3.14.9 (workspace tag v3.14.9; the Loop
submodule should be at commit c30b5c89). Re-apply the home-screen redesign.

### Sources, in order of preference

1. **Git forks (GitHub user msreyesUS)** — complete history:
   - `https://github.com/msreyesUS/Loop` branch `ui-redesign`, head `d6606dec`.
     Redesign base is `1f71ec4f` (LoopKit `dev` at 3.14.2).
   - `https://github.com/msreyesUS/G7SensorKit` branch `ui-redesign`, head `b3b19ae`,
     base `890e607`.
   - `https://github.com/msreyesUS/OmnipodKit` branch `ui-redesign`, head `9572ca3`,
     base `d68699c`.
2. **Patch files in this folder** — identical content, no network needed:
   - `loop-ui-redesign.patch` — `git diff 1f71ec4f d6606dec` of the Loop repo
     (33 files, +2733/-446). Apply in the `Loop` submodule with
     `git apply --3way --index loop-ui-redesign.patch`.
   - `g7sensorkit-sensor-pill.patch` — one file, `G7CGMManager+UI.swift`.
   - `omnipodkit-pod-lifecycle-bar.patch` — ten lines in `OmniPumpManager.swift`.
   - `loop-commit-history.txt` — the commit list, for context on intent.

### Expected merge behaviour

- **Loop:** upstream made no content changes to `StatusTableViewController.swift`,
  `BolusEntryView.swift`, or `CarbEntryView.swift` between `1f71ec4f` and
  3.14.9, so the patch should land nearly clean. Expect small conflicts only in
  `Loop.xcodeproj/project.pbxproj` (SceneDelegate.swift must be added to the Loop
  target), `Loop/Info.plist`, and the two `Localizable.xcstrings` files (keep both
  sides' strings). `FavoriteFoodDetailView.swift`: take upstream's version.
- **G7SensorKit:** upstream also edited the same file (11 commits since base).
  Resolve preserving Marco's display behaviour (see spec).
- **OmnipodKit:** upstream rewrote `OmniPumpManager.swift` for connect-on-demand.
  Do NOT merge; re-implement the ten-line lifecycle-bar behaviour by hand (spec
  below). Do not touch any pod communication code. This is the whole reason for
  the upgrade.
- **UIScene lifecycle:** keep Marco's `SceneDelegate.swift` + Info.plist
  `UIApplicationSceneManifest` + AppDelegate deferral. Upstream 3.14.9 has not
  adopted scenes and the iOS 27 SDK kills legacy-lifecycle apps at launch.

### Design spec (what the result must look like)

Status screen, top to bottom, on a soft blue-to-green vertical gradient with
translucent rounded cards (62% system background, hairline border):

1. **Top status card** (74pt tall, one shared card): sensor pill on the left
   (icon + state text such as sensor time remaining), **48pt loop ring vertically
   centered** in the middle, pump pill on the right showing **pod time remaining
   (days/hrs/min) instead of the basal dial**, with the pod lifecycle bar under it.
   Pill text is 13pt semibold in `secondaryLabel` (same tone as "Eventually").
2. **"Glucose Now" hero card:** title top-left (bold), display unit in
   `secondaryLabel` under it, an info button top-right that **opens the Insulin
   Delivery history** (`InsulinDeliveryTableViewController` segue). Centered large
   glucose "coin": grey gradient ring with soft shadow, value 64pt; white in range,
   flooded yellow when high, flooded red with white text when low; Dexcom-style
   trend arrow cut into a bulge on the ring. "Eventually <value>" caption in
   `secondaryLabel` below.
3. **"Glucose – Last 3 Hours" chart card:** history-only line (no prediction),
   fixed scale with dashed gridlines, green target band with "Target Range" label
   and the band bounds on the right edge; **gradient shading under the line only
   where the line is above the target band** (fill floor is the band's top edge).
4. **Stats row:** three small cards, Active Insulin (caption: last bolus amount
   and time), Active Carbs (caption: last carb entry time), Basal (caption: when
   the current net basal took effect).
5. **Status slot:** a **permanent 58pt row, invisible when idle**, that shows the
   bolus progress card, Starting/Canceling Bolus, Insulin Suspended, No Recent
   Glucose, or an active override. Cards **fade in and out in place** (snapshot
   fade-out, alpha fade-in); every card is rendered at the fixed 58pt height so
   the action buttons below never move. A second row appears for an active
   override when the primary slot is busy.
6. **Action buttons:** Bolus (slate `primaryButton1`), Carbs (green), More
   (blue `infoAccent`), Settings (light slate `primaryButton3`). **Flat
   `.filled()` configuration, never Liquid Glass or gradients.** Content is a
   custom centered stack (icon above 15pt semibold label) pinned to the button's
   exact center; minimum height 64pt; table has `delaysContentTouches = false`.
   Medium haptic on tap. "More" presents a sheet with full-width Pre-Meal,
   Suspend, and Preset buttons (leading icon, 17pt), with a pressed-in "active"
   style (tinted border, light fill) when a preset is on.

Bolus modal: titled "Cancel" bar button (not the system X), "Bolus Summary"
card first with the forecast chart below it, list top margin 4pt and 10pt
between cards (iOS 17+ `contentMargins`/`listSectionSpacing`). Add Carb Entry:
no trailing toolbar Continue button; only the blue in-content one.

Color assets (in `Loop/DerivedAssetsBase.xcassets`): critical, criticalBackground,
fresh, infoAccent, primaryButton1/2/3, purpleAccent, secondaryAction,
statusBackgroundTop/Bottom, warning, warningBackground — all in the Loop patch.

G7SensorKit: during normal operation the sensor pill highlight shows **time
remaining as days/hrs/min**; the lifecycle bar length represents time remaining
throughout sensor life; "Sensor Issue" is critical (red), not warning.

OmnipodKit: pod lifecycle bar shows progress throughout normal pod life; it
depletes (remaining fraction) and switches to the warning state only in the
final four hours; otherwise it stays the normal pump color near end of life.

### Known pitfalls from the original work

- Setting `titleAlignment = .center` on a `UIButton.Configuration` with an
  attributed title does nothing; the paragraph style wins. The redesign bypasses
  UIButton's own image/title layout entirely for the top-icon buttons.
- Nested `tableView.beginUpdates()` batches defer row changes to the outermost
  (animated) `endUpdates()`; the status-row update must run outside any other
  batch or the fade breaks and the buttons bounce.
- Xcode occasionally reverts an edit made while it has the file open. After
  each edit, confirm it on disk (grep) before building.
- The bolus progress nib is 44pt tall; without the fixed 58pt row height the
  buttons shift 14pt when it appears.

### Finish

Build, run on the iPhone 18 Pro Max, verify every item in the spec, enact a
small bolus and a suspend to confirm the cards fade without moving the buttons.
Then push each repo to a new branch `ui-redesign-3.14.9` on the msreyesUS forks,
point `.gitmodules` at those branches, commit the submodule pointers, and push the
workspace to `msreyesUS/LoopWorkspace`.
