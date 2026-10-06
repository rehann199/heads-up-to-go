# Architecture

Conceptual view of the Android TV app. Names are descriptive, not production identifiers.

![System overview](../assets/diagrams/system-overview.svg)

## Layers

| Layer | Responsibility |
|---|---|
| Input | Per-screen key handlers translate D-pad, select and back into cursor moves or round actions |
| Screens | Splash, home, rules, settings, categories, player count, custom names, countdown, round, turn summary, winner, leaderboard |
| Round state | Shuffled deck, current player and word, per-player tallies, turn timer and pause |
| Data | Bundled prompt lists, settings, last game results, games-played counter |
| Services | Audio, analytics, crash reporting, ads, billing client in Dart |
| Native | Kotlin billing manager, store-link intent, activity host |

## Startup

```mermaid
sequenceDiagram
    participant App as main()
    participant FB as Firebase
    participant Sp as Splash
    participant Bill as Billing (Dart + Kotlin)
    participant Au as Audio
    App->>App: lock landscape
    App->>FB: init, enable crash collection (release/profile)
    App->>App: start analytics session, log app open
    App->>Sp: show splash
    Sp->>Sp: precache splash image
    Sp->>Bill: initialise (3 s cap), restore, refresh ownership
    Sp->>Au: initialise with saved settings
    Sp->>Sp: short delay, then go to home
```

## A turn

```mermaid
stateDiagram-v2
    [*] --> EntryAd: first turn only
    EntryAd --> Running: ad done, skipped, failed or timed out
    Running --> Feedback: right / left / down
    Feedback --> Running: after 1 s, next word
    Running --> Paused: select
    Paused --> Running: select
    Running --> ExitMenu: up
    ExitMenu --> Running: resume
    ExitMenu --> [*]: back
    Running --> TimesUp: timer reaches zero
    TimesUp --> Summary: after 1.5 s
    Summary --> Running: next player
    Summary --> Winner: last player
```

## Category selection and unlock

```mermaid
flowchart TD
    A[Select on a category card] --> B{Owned or free?}
    B -- yes --> G[Choose players, countdown, round]
    B -- no --> P[Purchase popup]
    P --> Buy[Buy / restore<br/>native billing]
    P --> Ad[Watch rewarded ad]
    Buy --> G
    Ad --> One[One play, held in memory] --> G
```

## Billing bridge

```mermaid
flowchart LR
    D[Dart subscription service] -- method channel --> K[Kotlin BillingManager]
    K -- event channel --> D
    K --> PB[Play Billing client]
    K --> C[(Cached ownership)]
```

Events carried to Dart include billing ready or disconnected, product list, ownership sync, purchase success, error and cancel.

## Focus and cursor ownership

- A screen with an explicit model (categories, player count, names, round) takes focus on a single node that receives all key events and updates its own cursor state.
- A screen with its own dialog hands focus to the dialog's scope and takes it back on close.
- The reusable carousel card widget handles focus visuals only: it animates scale, swaps layout and scrolls itself into view, and swallows horizontal arrows so framework traversal does not fight the cursor model.

## Build and release

Release builds enable minification and resource shrinking, keep Flutter and audio plugin classes through ProGuard rules, and package two ABIs. Signing material is supplied from outside the repository and falls back to debug signing when absent. Details of keys and identifiers are not published.
