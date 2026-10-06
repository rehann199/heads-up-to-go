# Engineering Challenges

Six problems that stand out in the Android TV app. Each is drawn from behaviour visible in the code. Outcomes describe what the code does; no metrics are claimed.

---

## 1. A game that is played with four arrows and a select key

**Problem**
A remote has no pointer. Menus, a carousel catalogue, a number picker, a typing keyboard and a live round all have to be driven by the same few keys, and the meaning of an arrow changes between a menu and a round (move versus score).

**Context**
Flutter offers geometric focus traversal, which works for simple lists but is hard to predict in a layout of expanding cards, overlapping pills and a letter grid.

**Approach**
- Each screen has one key listener and its own small position state: a section and index for the catalogue, a cursor index for the player screen, a row and column (clamped when moving between rows of different length) for the name keyboard.
- Moves between regions are written explicitly: the catalogue moves Up and Down between four sections and stops at the end of a row; Up from the first section reaches a back control.
- In a round the same arrows are remapped: Right is correct, Left is wrong, Down is pass, Up is the pause and exit menu, and Select toggles pause.
- A flag disables scoring input while a feedback screen is showing, so a double press cannot score twice.

**Trade-offs**
The movement rules are code, so adding a control means editing the cursor logic. The catalogue screen is about 3,000 lines, and there are no tests around the movement.

**Outcome**
Every screen has a deterministic, readable movement model, and the round maps one key to one action.

---

## 2. Keeping focus after navigation, dialogs and ads

**Problem**
When a route is pushed or popped, or a dialog closes, a TV user can end up with no focused widget and a remote that does nothing.

**Context**
The round pushes a turn summary, can show an exit dialog and may be preceded by a full-screen ad. Each hands focus elsewhere.

**Approach**
- The round screen unfocuses its key node and its scope before navigating, and sets a "navigated away" flag so that a later dependency change does not pull focus back.
- When the screen is current it requests focus after a short delay, once the previous owner has released it.
- The exit dialog has its own focus scope and key listener, and the round screen does not accept focus while it is open.
- Returning from the summary resets the flags and restores focus for the next player.

**Trade-offs**
The fixed delays are timing assumptions, and the flags must be kept in step with every navigation path.

**Outcome**
Focus is explicitly released and re-requested at each hand-off instead of relying on a default.

---

## 3. A catalogue card that grows when focused

**Problem**
Focusing a card should make it larger and show richer content, without the grid reflowing, without the card leaving the visible area and without a horizontal arrow moving focus somewhere unexpected.

**Approach**
A reusable focus card widget animates a scale on focus change, swaps a small child for an expanded child through an animated switcher, scrolls itself into the safe area of the carousel when it is close to an edge, and consumes Left and Right so that the screen's explicit cursor model remains the only thing that moves selection.

**Trade-offs**
The safe-area padding is a constant that must match the list padding, and the widget depends on the screen to move selection.

**Outcome**
A focused card expands and stays on screen; the visual state follows focus while movement stays under the screen's control.

---

## 4. Ads that must never trap the player

**Problem**
Interstitial and rewarded ads depend on an SDK and a network. A callback that never arrives would leave a TV on a spinner with no way out.

**Context**
Ads appear before the first turn, when entering the leaderboard, and as an optional rewarded unlock.

**Approach**
- SDK initialisation and preloading happen in the background with a 10-second init cap and 15-second load timeouts, and a show call waits at most 16 seconds for a load.
- A 90-second cooldown limits interstitial frequency, and initialisation is skipped when the no-ads entitlement is owned.
- Every show path has a 60-second safety timer, and every failure, skip and timeout path calls the same continuation that starts the game or leaves the screen.
- A rewarded ad grants one play, held only in memory, and consumed when the round starts; it is cleared on each cold start.
- A flag marks a full-screen ad as visible so that the app being paused by the ad is not logged as the player leaving.

**Trade-offs**
Timeouts are fixed constants, and a slow ad can still delay a turn start for part of the cap.

**Outcome**
Each ad outcome (completed, skipped, failed, timed out) leads to the same next step.

---

## 5. Billing without blocking the UI

**Problem**
Play Billing is callback-based and can disconnect; ownership affects which cards are locked and which artwork is shown.

**Approach**
A Kotlin manager owns the billing client. It reconnects two seconds after a disconnect, ignores duplicate connection attempts, queries product details and purchases after setup, acknowledges unacknowledged purchases, and caches ownership in local preferences. Dart gets one method channel for commands and one event channel for results, and resolves ownership during splash with a three-second cap on initialisation, so catalogue artwork is chosen after ownership is known.

**Trade-offs**
A cached ownership value can be stale until the next query completes. Product identifiers live in a native constant list that has to match the store catalogue.

**Outcome**
The Dart side deals in simple events, and the catalogue can decide card artwork from known ownership.

---

## 6. Analytics that cannot silently break

**Problem**
GA4 rejects or drops events that break its rules, and Flutter's automatic route observer and lifecycle callbacks can produce duplicate events.

**Approach**
One sanitiser sits in front of every event: reserved names get a suffix, reserved prefixes are prefixed, names and values are truncated, empty values are dropped and parameters are capped. Screen views are logged by hand with stable names and a visit sequence so exits carry an outcome. Leave events are debounced within 500 ms. The sanitiser is plain Dart with 16 unit tests.

**Trade-offs**
Renaming hides a mistake at the call site rather than failing loudly, and a hand-logged screen event is easy to forget on a new screen.

**Outcome**
Event names and parameters are always within the platform's limits, and duplicated lifecycle events are collapsed.
