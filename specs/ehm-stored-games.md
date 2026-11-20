# What EHM keeps from the games you play

*Version 1.0, 20 November 2026. Tested on EHM 1.5.0 (build 1669120) with a custom online database. How the game keeps games is the same in every database; how far back it reaches depends on how many leagues you load and how busy they are.*

This page describes how Eastside Hockey Manager keeps the details of games once they are played: what it stores, what it deletes, what it keeps elsewhere, and how a stored game's play-by-play is shown again.

## How to read this

Every statement carries one label.

| Label | Meaning |
|---|---|
| **Tested** | We checked it against what the game stored and showed on its own screens. |
| **From the game's code** | Read from the game's program logic. It has not been separately confirmed in game. |
| **Not known** | We could not settle it. Where we have a reading, it is given as a reading, not a fact. |

Where the game's developer has confirmed a point, the page says "confirmed by the developer".

## 1. The short version

- **The game keeps the detail of recent games only.** Box scores, game reports and play-by-play for a game are stored when it is played. Older ones are deleted as new games come in.
- **Your league's games are kept longer than other leagues'**, and the oldest are deleted first. **Tested.** The developer has also confirmed that your own club's games are kept longest of all.
- **The limit is storage space, not a number of games.** The "Games Stored" setting did not decide how much was kept in our tests.
- **Season totals are not affected.** Player and club season statistics are kept separately and stay complete when the game details are gone.
- **A stored game's play-by-play follows a fixed script.** Reopen it and the same events happen at the same times; only the wording of the lines changes.

## 2. Stored games are kept by priority

**Tested.**

The game keeps stored games in separate stores:

| Store | What it holds | How it is kept |
|---|---|---|
| Your league | the games in your own league, your club's included | kept longest of the league stores |
| Other loaded leagues | games in the other leagues you loaded | kept least |
| Tournaments and internationals | tournament and international games | kept a long time, in less detail |

- **Your own club's games are kept longest of all**, ahead of the rest of your league. That is confirmed by the developer; our count did not separate them from your league's other games.

- **When a store is full, its oldest games are deleted first.** We counted every stored game in one save. Each league's games ran unbroken back to a cut-off date, with nothing missing in between.
- **The order is predictable.** From what was stored, we predicted exactly which games would survive on a day that was only partly deleted, and the game's screen matched.

**How far back each store reached, in the save we measured:**

| Store | Reached back about |
|---|---|
| Your league | 200 days |
| Other loaded leagues | 115 days |
| Tournaments and internationals | years |

These depths are from one save. They will differ with how many leagues you load and how many games they play.

## 3. The limit is space, not a number of games

**Tested.**

- **The "Games Stored" setting did not limit what was kept** in our tests: the stores filled up long before the number it was set to. What the setting does when it is set lower is **Not known**.
- **What runs out is storage space in each store.**
- **Not known, our reading:** loading more leagues, or busier ones, would fill a shared store faster, so each league's games would reach back less far. We did not test this directly.

## 4. Season totals do not come from stored games

**Tested.**

- **Players' and clubs' season statistics are kept separately** from the stored game details.
- **An example from our test:** a player who had played every game of an 82-game season had only 63 of those games still stored, yet his season totals showed all 82 correctly.
- **So losing old game details does not change anyone's statistics.**

## 5. Games without play-by-play

**Tested.**

- **Some games are stored with less detail.** These are mostly games the game simulated quickly, outside the leagues you chose to load in detail. Only a small share of the games in your loaded leagues are like this.
- **For these games the game offers no play-by-play:** "There is no play-by-play available."
- **Not known:** exactly which leagues and games are stored in less detail beyond that pattern.

## 6. The play-by-play of a stored game

- **It follows a fixed script.** **Tested.** We opened the same stored game five times, three in one session and two more after restarting the game. All 32 timed events came back at the same times, in the same order, every time.
- **The wording changes.** About three lines in four were worded differently from one viewing to the next, while the events they described stayed the same.
- **It is re-created when you open the game.** The game re-creates a stored game's play-by-play by re-running the game when you open it. Confirmed by the developer, for games played on the current match engine.
- **Games stored in less detail have no play-by-play to show** (section 5).

## What I left out on purpose

- **Storage sizes and counts.** The page says how the limit behaves, not how big it is.
- **How any of this was measured,** and the names of the game's internal files.
- **Advice.** The page reports what the game does; it does not tell you what to do about it.
- **The clubs and players we tested on.**
