# Product Overview

Heads Up To Go is a party game for Android TV. The TV shows a prompt, a countdown timer and the players' scores. One person acts out or describes the prompt, the others guess, and whoever holds the remote marks the answer. Everything is operated with the remote's D-pad and select button.

No Android TV screenshots are included in this repository; the images used are artwork (see [Privacy and source](PRIVACY_AND_SOURCE.md)).

## Platform

- Android TV only. The manifest requires the Leanback feature, declares that a touchscreen is not required, registers a Leanback launcher activity and provides a TV banner.
- Landscape orientation is forced at startup.
- Flutter and Dart for the UI, with a small amount of Kotlin for billing and for opening the store listing.

## Flow of a game

1. **Splash.** Precaches the splash image, warms the billing ownership cache (with a 3-second cap on initialisation), enters immersive full-screen mode, starts audio and then moves to the home screen. Ads and billing are also initialised in the background.
2. **Home.** Navigated with Up and Down; entries lead to play, categories, rules and settings.
3. **Categories.** A catalogue in four sections (featured, free, paid, bundles). Left and Right move within a section, Up and Down move between sections.
4. **Players.** Choose 2 to 6 players with number circles, and optionally enter names on an on-screen keyboard.
5. **Countdown.** 3, 2, 1, GO.
6. **Round.** Each player has one timed turn. The deck is shuffled once per game.
7. **Turn summary.** Lists the correct, wrong and passed words for the player who just finished.
8. **Winner and leaderboard.** The player with the highest score is shown, and the last game's standings are stored locally and shown ranked.

## Round controls

| Key | During a turn |
|---|---|
| Right | Mark correct |
| Left | Mark wrong |
| Down | Pass |
| Up | Open the pause menu (Resume or Back) |
| Select / Enter | Pause or resume |

Each action shows a one-second feedback screen (correct, wrong, pass); the end of the timer shows a "Time's up" screen for a second and a half before the summary. While feedback is showing, scoring input is ignored.

## Content

| Item | Count |
|---|---|
| Featured carousel cards | 5 |
| Free categories | 9 |
| Paid categories | 18 |
| Bundles (each combines several word lists) | 5 |
| Bundled word lists | 25 (9 free, 16 paid) |
| Prompts | About 7,000 in total (approximate, counted by line) |

Some paid categories are present in the catalogue without a wired word list. Prompt text is not published here.

## Settings

- Turn length from 60 to 180 seconds in 20-second steps (default 120)
- Sound on or off
- Background music track (eight tracks)

## Ways a category is unlocked

- A one-time purchase through Google Play Billing
- A no-ads subscription
- A rewarded ad that grants a single play of one category (held in memory only)
- A games-played counter shown on some categories

## What is not in the product

- No touch interface
- No accounts, sign-in or cloud leaderboard; the leaderboard shows the last game only
- No localisation setup; the interface is English
