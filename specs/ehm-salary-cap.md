# How EHM counts the salary cap

*Version 1.0, 20 November 2026. Tested on EHM 1.5.0 (build 1669120) with a test copy of the Pivot database, re-checked on a custom online database. Every setting named here is a database value; your database may set it differently.*

This page explains which players count against a club's salary cap in Eastside Hockey Manager, how the game works out the money on the Finances page, and why you should keep your own cap sheet if your club carries buried contracts.

## How to read this

Every statement carries one label.

| Label | Meaning |
|---|---|
| **Tested** | We checked it on a running game against the game's own screens and figures. |
| **From the game's code** | Read from the game's program logic. It has not been separately confirmed in game. |
| **Not known** | We could not settle it. Where we have a reading, it is given as a reading, not a fact. |
| **Comparison** | Taken from outside the game (the real NHL CBA), for comparison only. |

**Two kinds of fact appear here:**
- **How the game counts:** its own behaviour, the same in every database.
- **The numbers it counts with:** set by the database you play. They appear below as named settings, never as values to copy.

## 1. The short version

- **The Salary Cap Chart and the Finances page do not agree.**
  - The chart's Counts column counts a buried contract's excess over the farm limit.
  - The Finances totals ("Salaries left to pay", "Averaged Club Salary") leave buried players out entirely.
  - The contract offer screen quotes the Averaged Club Salary when it tells you how much your board will allow.
  - **Tested.**
- **If your club carries buried contracts, the game's money total understates what the chart says counts.** Keep your own sheet. The calculator on the web version of this page does it for you.
- **Injured players on IR stop counting.** The AI places injured players on IR itself; a human manager must do it.
- **In the offseason window only the top 23 players by current weekly wage count.**

## 2. Where the settings are

All of these are in the Editor, on the league's transfer rules or its extra rules.

| Setting (Editor name) | What it does |
|---|---|
| Salary cap | The cap. The chart and the Finances page compare against it. |
| Salary floor | The floor. |
| Salary cap farmed wage max yearly limit (extra rule 121) | The **farm limit**: how much of a buried one-way contract is forgiven (section 4). |
| Salary cap offseason start day / end day (extra rules 30 and 31) | The **offseason window**, when only the top 23 count (section 3). |
| Long term IR min days | How long an injury must be before the AI places the player on IR. A league that leaves it at 0 uses 24 days. The Editor also shows Short term and Medium term IR min days; this test does not use them. |
| Contract buyout bracket 1 / 2 age limit, … percentage cap hit, … apply salary adjustment on cap hit, … cap hit years multiplier (extra rules 149 to 154) | Feed the buyout-shaped cap penalty (section 6). Each is a value the database sets; how they combine is not known. |
| Player budget (each club) | The club's wage budget. The contract offer screen shows it beside the cap. |

## 3. Who counts against the cap

The Salary Cap Chart's **Counts** column shows, for each contracted player, whether he counts and why.

| Label | Means | Counts? |
|---|---|---|
| **RP** | Roster Player | yes |
| **BC** | Buried Contract: in the minors on a one-way deal above the farm limit | only the part above the limit, on the chart (section 4) |
| **N/R** | Not on roster | no |
| **IR** | Injured Reserve | no (section 5) |
| **OFF** | outside the top 23 in the offseason window | no |
| **TC / LC** | trial or loan contract | no |
| **WVR** | on waivers | **Not known** |

**Outside the offseason window, the game decides in this order.** **Tested** on every row of seven clubs' charts.
1. A contract with no cap hit doesn't count. Trial and loan contracts land here.
2. A player **on IR** doesn't count.
3. A player **playing for the club that holds his contract** counts (RP).
4. A player **in the minors on a one-way contract** whose cap hit is above the farm limit counts as **BC**.
5. Anyone else doesn't count (N/R).

**Inside the offseason window,** only the **top 23 players by current weekly wage** count. **Tested** on every row of three clubs' charts.
- **The sort:**
  - the club's players by current weekly wage, highest first;
  - ties by full name, A to Z, then by the player's place in the database;
  - the order is fixed, not random.
- **The count:** the first 23 are RP; the rest are **OFF**.
- **Not cap hit:** the list is sorted by weekly wage, so on the chart an OFF player can sit above an RP player.
- **Goalies:** there is no separate count for skaters and goalies.

## 4. Buried contracts

**On the chart (Tested).**
- **The rule:** while the league's season is under way, a player in the minors on a **one-way** contract counts only the part of his cap hit above the farm limit:
  - counted per week = cap hit per week − (farm limit ÷ 52, rounded down), never below 0.
- **The two figures:** the chart shows the reduced amount first, the full one in brackets. For example, a $6,000,000 contract with a farm limit of $1,200,000 shows "$4,800,000 ($6,000,000)".
- **Two-way contracts:** a player in the minors on a two-way contract shows $0.
- **Below the limit:** so does a one-way player whose cap hit is below the farm limit.

**On the Finances page (Tested).**
- **No buried amount at all:** "Salaries left to pay" counts only the players playing for the club. "Averaged Club Salary" is built from it, so it carries no buried amount either.
- **So the game shows two cap positions** whenever a club carries buried contracts:
  - the chart's, with the buried excess;
  - the Finances page's, without it.

**What the game enforces: Not known.**
- When you make an offer, the offer screen quotes the Averaged Club Salary, the total without buried players (section 8).
- Whether the board's limit itself takes buried contracts into account is not known.

**After the season:** whether a farmed one-way player counts in full once the season is over is **Not known**. On a date during the playoffs he still showed the reduced figure.

## 5. Injured players: IR

**Tested.**
- **A player on IR stops counting.** His cap hit stays on the chart, marked IR, but the club's figures leave him out, this season and next.
- **What puts a player on IR:**
  - **An AI-run club:** the player is on IR if his current injury is at least the league's "Long term IR min days", or he is on the IR list. The AI places long-injured players on IR itself.
  - **A human-run club:** only the IR list counts, so the manager must place him.
  - We saw the same injured player show IR under the AI and BC the moment a human manager took the club over.
- **No long-term tier:** EHM's relief comes from IR itself. There is no separate long-term status and no "exceed the cap" step.

## 6. Cap penalties

- **The Finances page shows each penalty as its weekly amount × 52,** rounded like every other figure. It counts in every season up to its last one. **Tested** on four clubs.
- **The game can create a penalty three ways.** **From the game's code.**
  - **one is shaped like a buyout,** using the contract's remaining wages and years and the contract buyout rules (section 2);
  - **one records half a season's cap hit** until the contract ends;
  - **one takes its amount from elsewhere.**
- **What triggers each, and the buyout formula: Not known.**

## 7. The money on the Finances page

**Tested.**

| Line | What it is |
|---|---|
| **Salaries paid out** | what the club has paid this season so far |
| **Salaries left to pay** | the cap hits of the players playing for the club (IR left out) × 52, scaled by the share of the season left. Next season's column uses the full season |
| **Cap penalties** | section 6 |
| **Salary Cap (p/a)** | the league's cap |
| **Averaged Club Salary (p/a)** | paid out + left to pay + penalties. Shown as "Est." during the offseason window |

**Rounding.** Figures are rounded to the **nearest** step (halves up), and the step depends on the size:

| Amount up to | Rounds to the nearest |
|---:|---:|
| $10,000,000 | $1,000 |
| $100,000,000 | $5,000 |
| above | $10,000 |

- **An example:** $108,507,256 shows as $108,510,000.
- **Small amounts use smaller steps.**

**Why two screens can disagree.**
- **The page works out its own figures** from the current roster each time it is shown.
- **The game also keeps a stored payroll,** and opening the page does not reliably update it. We saw the stored figure out of date while the page was right.
- **If a number elsewhere disagrees with the Finances page, trust the page,** and remember that the page leaves buried players out.

Club wage budgets are covered in [How finances work in Eastside Hockey Manager](ehm-finances.html).

## 8. What your board lets you offer

The contract offer screen says, for example:

> Your board will allow a salary up to $9,000,000 (p/a) for this contract.
> The current Averaged Club Salary is $87.5M (p/a), while the player budget is $109M (p/a) and the cap is at $104M (p/a).

- **The cap total it quotes is the Averaged Club Salary,** rounded down: the total that leaves buried players out. **Tested.**
- **The board's limit is not simply your cap room.** It depends on the role you offer (a Core role allowed far more than a Fringe one) and on the club. **Tested** on two clubs.
  - For the same three players and roles, the club with less cap room and a different reputation was allowed less at every role.
  - The finance guide traces the board's limit to the club's reputation. How cap position enters is **Not known**.

## 9. Keep your own cap sheet

Online leagues often keep a cap sheet outside the game. These findings show why: a club with buried contracts sees two different cap positions, and the game's money figures quote the lower one.

A sheet that matches the chart needs, per player:
- the cap hit;
- where he plays: with the club, in the minors on a one-way deal, in the minors on a two-way deal, or on IR.

Then:
- **the chart's count** = roster cap hits + (each buried one-way cap hit above the farm limit − the farm limit) + penalties;
- **the game's money total** = roster cap hits (IR out) + penalties, over a full season;
- **cap room** = the cap − whichever total your league uses.

The web version of this page has a calculator that does this.

## 10. "Is the cap bugged?"

**The verdicts follow one rule:**
- **GAME BUG:** the game contradicts its own rules or figures.
- **WORKS AS DESIGNED:** consistent with itself, even where it differs from the real NHL.
- **UNPROVEN:** not settled, including where two of the game's own screens disagree.

| Claim | Verdict | Why |
|---|---|---|
| "Buried players count their full salary" | **UNPROVEN** | The chart counts only the part above the farm limit; the Finances totals count none of it (section 4) |
| "The cap screen shows my real cap position" | **UNPROVEN** | With buried contracts, the chart and the Finances page give different totals; the offer screen quotes the Finances one |
| "Injured players still count against my cap" | **WORKS AS DESIGNED** | IR players leave the club's figures; the chart still shows their cap hit (section 5) |
| "My injured player isn't on IR" | **WORKS AS DESIGNED** | On a human-run club you must place him; the AI does it only for its own clubs |
| "In the offseason the game ignores some of my players" | **WORKS AS DESIGNED** | Only the top 23 by weekly wage count (section 3) |
| "The 23rd player in the offseason is picked at random" | **WORKS AS DESIGNED** | Ties break by full name, then database order |
| "Cap figures are rounded oddly" | **WORKS AS DESIGNED** | Size-based rounding (section 7) |
| "The Finances numbers changed when I opened the page" | **WORKS AS DESIGNED** | The page works out its own figures from the current roster |
| "A two-way player sent down counts nothing" | **WORKS AS DESIGNED** | That is the game's rule during the season |
| "Buyouts don't affect the cap" | **UNPROVEN** | A buyout-shaped penalty exists in the code; its trigger is not known |
| "You can't retain salary in a trade" | **WORKS AS DESIGNED** | The Editor has a "Retain salary in trades" setting, but the game never implemented it (confirmed by the developer) |

## 11. EHM compared with the real NHL

This is a comparison, not a list of bugs. The real-NHL column describes the current NHL–NHLPA Collective Bargaining Agreement in broad terms, from outside the game; it is not quoted and gives no figures. Check the agreement's text before quoting it.

| Topic | Real NHL (CBA) | EHM | Can EHM do it? |
|---|---|---|---|
| **Buried contracts** | A player in the minors frees a set amount of his cap hit; the rest counts | The chart counts the part above the farm limit; the Finances totals count none of it | **Partly:** the chart follows the idea; the money figures do not |
| **Injured players** | Regular IR gives no cap relief. LTIR lets a team exceed the cap by the injured player's cap hit | Any IR placement takes the player out of the club's figures | **No LTIR:** IR itself is the relief; there is no "exceed the cap" step |
| **Offseason** | All contracts count; teams may exceed the cap by a set margin | Only the top 23 by weekly wage count during the window | **No overage margin** |
| **Buyouts** | A share of the remaining salary, spread over twice the remaining years | A buyout-shaped penalty on the contract buyout rules; its formula is not known | **Not known** |
| **Retained salary** | A team may retain part of a traded player's salary, within limits | The Editor setting exists but was never implemented (confirmed by the developer) | **Not possible** |
| **Cap hit** | The contract's average annual value | The contract's cap figure | yes |

## What I left out on purpose

- **Values from the databases we tested.** Every setting is named, not given; your database sets its own.
- **Club wage budgets in leagues without a cap.** That is finance; the finance guide covers budgets.
- **Save-file details** that only matter to outside tools.
- **How any of this was measured.** The page says what the game does, not how we read it.
- **The players and clubs we tested on.** The examples are rounded.
- **A formula for the board's offer limit.** Six data points are not enough to fit one honestly.
