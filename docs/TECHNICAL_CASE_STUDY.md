# Technical Case Study

This is an engineering write-up of Heads Up To Go, an Android TV party game. Statements describe behaviour observed in the production code, manifests and build files during a read-only review. Nothing here is a performance measurement; no metrics are claimed. Diagram labels are conceptual and do not match production identifiers.

## Shape of the project

| Area | Observation |
|---|---|
| Dart | 52 files, about 21k lines; about 13k lines of logic and UI, about 8k lines of prompt lists |
| Largest screens | The category catalogue is roughly 3,000 lines; settings is about 1,800 and the round screen about 1,500 |
| Kotlin | A `FlutterActivity` host, a small `Application` class and a billing manager, about 570 hand-written lines |
| Tests | 23 unit tests in 3 files: the analytics sanitiser (16), ad placement (3) and category analytics labels (4) |
| Release | Minification and resource shrinking enabled, ProGuard rules for the Flutter and audio plugins, ABIs limited to arm64-v8a and armeabi-v7a |
| Rendering | The Impeller backend is disabled in the manifest, with a comment that it crashed on the Android TV emulator |
| Fonts | Bundled in the app. A code comment records that fetching fonts at runtime had been a leading crash cause on low-end TV boxes with unreliable TLS or DNS |

## 1. The TV platform contract

The manifest declares the Leanback feature as required and the touchscreen feature as not required, registers a Leanback launcher category on the main activity, and provides a banner drawable for the TV launcher. The app is locked to landscape and runs in immersive full-screen mode after splash. Navigation is written around the remote's keys rather than touch.

Minimum SDK is taken from the Flutter default; target and compile SDK are 36.

## 2. Input model: arrows and select

Input is central to the design because the remote is the only controller. The project does not rely on a single mechanism:

| Screen | Mechanism |
|---|---|
| Home, categories, player count, custom names, round | A screen-level key listener plus explicit cursor state (section and index, a cursor index, or row and column). Movement rules are written as code. |
| Category carousel cards | A reusable focus-aware card widget that animates scale on focus, swaps a small and expanded layout, scrolls itself into the safe area, and swallows Left and Right so framework traversal does not move away from the explicit model. |
| Round pause menu | A dialog with its own focus scope and key listener, selecting between Resume and Back with the arrows, with Back and Escape treated as resume. |
| Purchase popup | A dedicated focus node handling Select for buy, restore and watch-ad actions. |

Most screens match on key labels such as "Arrow Up" and "Select". Select, Enter and, in the round and pause menu, the Back key are handled; the category screen uses the older raw key event API.

Details of the round controls are in [Product overview](PRODUCT_OVERVIEW.md).

## 3. Game state

Round state lives in the screen state object. Players hold correct, wrong and passed word lists; the score is the correct count. A turn has a once-per-second timer and a separate animation controller that drives the circular timer, which can be paused and resumed from the stored animation value. A feedback phase disables scoring input, shows a one-second result screen (a second and a half for the end of the turn), then advances the deck. After the last player's summary the winner screen is shown.

The deck is a copy of the category's list, shuffled once when the game screen is created. Bundles build their deck by combining and shuffling words from several lists.

## 4. Persistence

Persistence is local only:

- Settings (turn length, sound, music track) in `shared_preferences`
- The last game's results as JSON, read back for the leaderboard
- A total-games counter used for category unlock progress
- On the Kotlin side, cached billing ownership in `SharedPreferences`
- Temporary rewarded-ad access to a category is deliberately kept in memory and cleared on every cold start

There is no backend of the app's own, no account system and no cloud save.

## 5. Monetisation

- **Unity Ads** through `unity_ads_plugin`: interstitials before the first turn and when entering the leaderboard, and a rewarded ad that grants one play of a locked category. Interstitials respect a 90-second cooldown, preload and load timeouts guard against SDK stalls, and every show path has a safety timeout. If the no-ads entitlement is owned, SDK initialisation is skipped.
- **Billing**: a Kotlin `BillingManager` wraps the Play Billing client. It exposes initialise, product query, purchase, restore, ownership and destroy over a method channel, and pushes events back over an event channel. It reconnects after a service disconnect, enables pending one-time purchases, acknowledges purchases that are not yet acknowledged and caches ownership locally. The catalogue contains one subscription product (the no-ads entitlement) and a list of one-time products.

Product identifiers, ad unit identifiers and prices are intentionally not published.

## 6. Telemetry and diagnostics

- **Firebase Analytics** via a single service. All names and parameters pass through a sanitiser that renames reserved and prefixed events, truncates to GA4 limits and caps parameter counts. Screen visits are logged explicitly with a stable name and an exit outcome, and the automatic route observer is deliberately not used, with a code comment recording that it had double-fired screen views.
- **Lifecycle events** are debounced, and are suppressed while a full-screen ad is on screen.
- **Crashlytics** is enabled for release and profile builds only, with a custom wrapper that keeps startup and navigation breadcrumbs and a lifecycle state key. Flutter and platform errors are routed to it.

## 7. Audio and lifecycle

A single audio service plays selectable looping music with `just_audio`. It observes app lifecycle and pauses when the app becomes inactive or paused, resuming on return. The splash screen initialises it with the saved settings.

## 8. Resource handling

- Splash artwork is precached before the first navigation.
- The main screens (round, categories, players, names, settings) dispose their focus nodes, controllers and animation controllers in `dispose`, and the round screen unfocuses before navigating away.
- Ads and billing start off the critical path; billing initialisation during splash is capped at three seconds.
- Image assets are bundled PNGs; there is no remote content loading.

## 9. What is also in the repository

A static-routing configuration for Vercel, pointing at a Flutter web build output folder, and a Vercel project link folder exist in the production repository. They are not part of the Android TV app, and nothing in the repository documents what the web deployment was used for, so this case study makes no claim about it.

## 10. Limitations noted in review

- Game rules, the key handlers and focus movement are in large screen state classes and are not covered by tests.
- The category screen keeps its own selection state rather than using framework focus traversal, which gives exact control at the cost of a very large file.
- Some key handlers write debug output to the console on every key press.
- The raw key event API in the category screen is the older API.
- Winner selection takes the highest score and the code does not describe tie handling.
- The system Back key has no custom handling during a round; the pause and exit menu is reached with the Up arrow.
