# How finances work in Eastside Hockey Manager

Draft v0.1, 19 November 2026. For database editors and players who want to know what the finance settings
actually do. Written from tests on EHM 1.5.0 (build 1669120) with the Pivot database (pre-November release version).

## How to read this

Every statement carries one of three labels.

| Label | Meaning |
|---|---|
| **Tested** | We changed a value or ran a simulation and read the result from the game's saves and screens. |
| **From the game's code** | Read from the game's own program logic. It has not been separately confirmed by a simulation. |
| **Not known** | We looked and could not find the rule. |

All money is yearly unless it says weekly. The game stores contract wages weekly, so a 1,500 salary becomes
28 a week and shows as 1,456 in some places.

---

## 1. The short version

- A club's **Professional status** decides which finance rules apply to it at all. An Amateur club has a wage
  budget of zero and its board will not approve a paid contract.
- **Financial Info** rows hold the wage curves and the budget range. Each club uses one row.
- Wages come from one formula. The **spread factor** is the exponent in it: a high spread means only
  high-reputation clubs and people get much above the minimum.
- The **minimum full-time salary** is the starting point of every wage, for players and staff alike.
- A league's **minimum and maximum player wage** live in its transfer rules, not in Financial Info.
- **TV money** is set per league and paid to every club in full. It is the cleanest way to fix a league
  whose clubs lose money.
- A league that is not loaded as playable does not run a real economy. Do not judge a league's finances
  from a save where it was in the background.

---

## 2. Where the settings live

| Setting | Where in the Editor | What it controls |
|---|---|---|
| Professional status | Club, Basic Data | Which budget and wage rules the club gets |
| Player budget, Cash | Club, Finances | Starting wage budget and bank balance |
| Reputation, attendance, market size | Club | Feed the wage and budget formulas and gate income |
| Financial Info rows | Financial Info | Wage curves, minimum wages, budget range, junior allowance |
| Min / max player wage, contract length, roster limits | League transfer rules | Hard limits on player contracts in that league |
| Total TV money, Total prize money | Competition, Comp Rules, League Rules | Yearly league payouts |
| Salary per person | Staff / player record | The wage a person starts the game on |
| Disable Starting Wage Adjustments | New-game screen | Whether the game may change database salaries |
| Disable Starting Financial Adjustments | New-game screen | Whether the game may change starting balances and budgets |

---

## 3. Professional status

This is the first thing to check when a club cannot sign anyone.

| Status | Wage budget | Staff contracts | What we saw |
|---|---|---|---|
| Professional | From the Financial Info budget curve | Full-time | Normal behaviour |
| Semi-Professional | From the same curve at three-quarters strength | Part-time, with the part-time minimum as the floor | Normal behaviour |
| Junior | **Junior salary allowance × 25**, whatever the database says | Full-time | 1,500 allowance gives 37,500 |
| Amateur | **Zero** | None can be paid | Board allows $0 |

The Junior and Amateur rows of that table are tested. The Professional and Semi-Professional budget rules
are from the game's code.

- **Tested: Amateur clubs cannot pay anyone.** In one world, every one of 3,160 Amateur clubs had a budget of 0,
  and across ten weeks not one of them negotiated a signing. A human manager at an Amateur club saw "Your
  board will allow a salary up to $0" on a staff offer.
- **Tested: changing the status fixes it.** Setting the 62 NCAA clubs from Amateur to Junior gave each a budget
  of 37,500 on day one, with the database budget left at 0. The board then allowed staff offers, and over a
  full season every club signed staff and made money.
- **Tested: the Junior budget does not hold AI clubs back.** Major junior clubs carry payrolls of 150,000 to
  300,000 on a 37,500 budget and sign players every year. The finance screen compares the Player Budget with
  *player* salaries only, and junior players earn very little.
- **From the game's code:** an Amateur club's wage offer is computed as 0 before any Financial Info value is
  looked at. So no Financial Info setting can rescue an Amateur club.
- **From the game's code:** the statuses None, National (Senior) and National (U20) are not valid for a club
  that plays in a league.

College leagues are a special case inside the game. A college player's current season shows as "N/A" on the
contract screen whatever the club's status, and college clubs receive a yearly intake of new players in late
August. **Tested:** both still happen with the clubs set to Junior. The intake was 17 to 20 years old, and
rosters still ran up to age 24.

---

## 4. Which Financial Info row a club uses

**From the game's code.** The game walks the Financial Info table from the top and takes the first row that
fits:

1. a row whose league is the club's league;
2. otherwise a nation-only row for the nation of the club's league;
3. otherwise a nation-only row for the club's own nation;
4. otherwise the first row, which has no nation or league and acts as the default.

Things that follow from that:

- **A league row only wins if it sits above the nation rows in the table.** A league row added below its
  nation's row is never reached.
- A nation with only one row uses it for every league in that nation, top flight and third tier alike.
  Clubs then differ only through their own reputation.
- **Tested:** the NCAA has its own row in the Pivot database and does not fall back to the United States row.

---

## 5. The wage formula

Almost everything in Financial Info feeds this one formula.

> **wage = minimum full-time salary + (maximum − minimum full-time salary) × (reputation ÷ 10,000) ^ spread factor**

- *maximum* is the Max salary for the job or position.
- *reputation* runs from 0 to 10,000.
- *spread factor* is the Player spread factor for players and the Non-player spread factor for staff.

Whose reputation is used depends on the question:

| Question | Reputation used |
|---|---|
| What will this club pay? (board limit, and staff wages when a game is created) | The **club's** reputation |
| What does this person ask for? | The **person's** own reputation, blended with ability for players |

So a club can afford a person when the club's standing is at least as good as the person's. The two numbers
come off the same curve, which is why changing a row moves both the board's limit and the candidates' asks.

**Tested:** when a game is created, staff wages follow this formula using the club's reputation. For physios
and other minor staff it matched to the dollar at every reputation level we checked. For coaches and head
coaches it matched at lower reputations and came in *under* the formula at higher ones, so treat the formula
as an upper limit there. What trims the top end is not known.

### What the spread factor does

The spread factor decides how much of the gap between minimum and maximum a given reputation earns.

| Reputation | Spread 2 | Spread 3 | Spread 4 | Spread 5 |
|---|---|---|---|---|
| 2,000 | 4.0% | 0.8% | 0.16% | 0.03% |
| 3,500 | 12.3% | 4.3% | 1.5% | 0.5% |
| 5,000 | 25.0% | 12.5% | 6.3% | 3.1% |
| 7,500 | 56.3% | 42.2% | 31.6% | 23.7% |
| 10,000 | 100% | 100% | 100% | 100% |

- **Low spread: wages rise early.** Mid-reputation clubs pay well above the minimum.
- **High spread: wages stay on the floor** until reputation is high, then climb steeply.
- A spread of 4 or more puts every club below about 3,000 reputation on the minimum, however large the
  maximum is.

**Tested example, Great Britain row** (head coach maximum 625,000, minimum 10,000):

| Club | Reputation | Board limit at spread 4.0 | Board limit at spread 3.0 |
|---|---|---|---|
| Second-tier club | 2,000 | 10,000 | 13,500 |

At spread 4.0 a head coach asking 11,000 could not be signed. At 3.0 the same club's limit rose to 13,500 and
the offer went through. The candidate's ask rose as well, from 10,000 to 12,000, because asks use the same
row.

**Tested example, NCAA row.** With a head coach maximum of 4,000,000 and spread 2.25, college head coaches
started on 106,000 to 250,000 and club payrolls were around 420,000. With the maximum at 750,000 and spread
3.5 they started on 21,000 to 56,000. The maximum and the spread together set the level, so change them as a
pair.

---

## 6. Financial Info, field by field

| Field | What it does | Label |
|---|---|---|
| Nation, League | Decide which clubs use the row (section 4) | From the game's code |
| Max forward / defenceman / goalie salary | Top of the player wage curve for that position | From the game's code |
| Player spread factor | Exponent of the player wage curve | From the game's code |
| Max GM/head coach salary | Top of the staff curve for GMs and head coaches. One field covers both jobs | Tested |
| Max coach salary | Top of the curve for coaches and assistant GMs | Tested |
| Max scout salary | Top of the curve for scouts | Tested |
| Max other non-player salary | Top of the curve for physios and remaining staff | Tested |
| Non-player spread factor | Exponent of the staff curve | Tested |
| Min full-time salary | Starting point of every wage, players and staff. Floor for a Professional club. Also sets a floor under the club budget (20 × this) | Tested for staff; rest from the game's code |
| Min part-time salary | Wage floor at a Semi-Professional club, and 20 × this is its budget floor | From the game's code |
| Junior salary allowance | × 25 = a Junior club's wage budget | Tested |
| Foreign player salary boost | Multiplies a foreign player's wage ask | From the game's code |
| Min salary budget, Max salary budget | The range of club wage budgets for Professional and Semi-Professional clubs | From the game's code |
| Salary budget spread factor | Exponent of the budget curve | From the game's code |
| The "Median" columns | Shown by the Editor for guidance. They are worked out from the other fields and change when you change a spread factor | Tested |

Notes:

- **The minimum full-time salary is not a league minimum wage.** It is where the curve starts. A league's
  real minimum for players is in its transfer rules (section 8).
- **The salary budget fields do nothing for Junior or Amateur clubs.** A Junior club's budget is the
  allowance × 25 and an Amateur club's is zero.
- The game refuses values outside these ranges: spread factors 1 to 10; foreign boost 1 to 5; minimum
  salaries and junior allowance at least 250; minimum budget at least 5,000; maximum budget no more than
  100,000,000. (From the game's code.)

---

## 7. The club wage budget

**From the game's code.** For a Professional or Semi-Professional club:

> **budget = min salary budget + (max − min salary budget) × (club strength ÷ 200) ^ salary budget spread factor**

- *club strength* starts from the club's reputation ÷ 50 and is scaled by the chairman's resources, the
  attraction of the club's city, its market size and its financial state.
- The result is rounded to 10,000 and kept above 20 × the minimum full-time salary (20 × the part-time
  minimum for a Semi-Professional club).
- In a league with a salary cap, the budget is also tied to the cap.

**Tested:**

- **What the finance screen compares.** "Player Budget" is set against "Total Player Salaries". Staff wages
  are not counted.
- **Day one.** With Starting Financial Adjustments left on, the game changes the database budgets
  straight away. One league authored at 920,000 for every club started at 700,000 to 920,000; another
  authored at 690,000 started at 410,000 to 530,000.
- **Every season rollover the budget is set again, and it follows the club's money.** In one season we saw
  rises of 11% to 35% at profitable clubs, a 10% cut at clubs that finished in the red, and every Junior
  club go from 37,500 to 43,125.
- **Budgets do not strictly limit AI clubs.** Clubs commonly run payrolls somewhat over budget.

---

## 8. Player wages

### Starting wages

| Situation | What the game does | Label |
|---|---|---|
| Player has a database salary, **Disable Starting Wage Adjustments ticked** | Keeps it. All 1,450 NHL salaries survived | Tested |
| Player has a database salary, option **not** ticked | May change it. 37 NHL players changed: young players on expiring deals jumped to 4.6M–9.4M and some veterans were cut to the league minimum | Tested |
| Player has **no** database salary | The game sets it, mostly at the league minimum, whichever way the option is set | Tested |

- Tick "Disable Starting Wage Adjustments" if you want the database's salaries respected.
- The option does nothing for players without a salary. If a league's players have none, most of them start
  on the minimum.
- Even with the option ticked, a small number of salaries in a few leagues were still lowered by roughly
  10%. The cause is not known.

### League limits

These are in the league's transfer rules, not in Financial Info.

| Rule | Effect | Label |
|---|---|---|
| Contract player min yearly wage | No player in the league is paid less. A lower database salary is lifted to it. Applies to players only, never staff | Tested for the level; rest from the game's code |
| Contract player max yearly wage | Ceiling on any player contract. This is the figure the board quotes for a player offer | Tested |
| Contract max years | Longest contract | From the game's code |

- **Tested:** the board allowed a player offer up to exactly the league maximum (40,000 in one league,
  30,000 in another).
- **A high minimum wage is expensive.** In one second-tier league 71% of players sat on a 35,000 minimum. With
  rosters growing from 23 to 31 during the season, the minimum alone set most of the wage bill.
- A league with the minimum and maximum set to the same number has exactly one legal wage. That works: junior
  and college leagues are set up this way.

### Asks

**From the game's code.** A player's ask comes from the wage formula using a blend of his reputations and
current ability, then:

- is reduced for very high asks and for players aged 34 and over;
- is multiplied by the foreign player boost if he is foreign;
- is capped at the league maximum and floored at the league minimum.

---

## 9. Staff wages

- **Tested:** staff normally have no salary in the database, so the game sets every staff wage when the
  game is created, from the club's reputation and its Financial Info row (section 5).
- **Tested:** the board's limit for a staff offer comes from the same calculation. At a low-reputation club
  with a steep spread factor, the limit is the minimum full-time salary and nothing more.
- **From the game's code:** league minimum wage rules do not apply to staff. The floor for staff is the
  minimum full-time salary, or the part-time minimum at a Semi-Professional club.
- A Semi-Professional club offers staff part-time contracts.

If users report that they cannot hire staff, check in this order: the club's Professional status, then the
row's Non-player spread factor, then the staff maximums.

---

## 10. Where a club's money comes from and goes

The finance screen's lines are stored per club for this month, last month, this season and last season.
**Tested:** the stored lines match the screen exactly, and month to month the balance changes by income minus
expenditure.

### Income

| Line | What we found |
|---|---|
| TV Revenue | Section 11. Often the largest line outside the top leagues |
| Gate Receipts, Season Tickets | Follow attendance and ticket price. Small for clubs with low prices |
| Merchandising | Steady through the season |
| Other | Large for many clubs. What drives it is not known |
| Prize Money, Interest, Players Shipped Out, Investments | Small or occasional in our tests |

### Expenditure

| Line | What we found |
|---|---|
| Wages | Players and staff together |
| Ice Rink/Arena Maintenance | Can be nearly as large as wages at a small club |
| Other | A steady cost through the season |
| Players Acquired, Signing On Fees, Bonuses | Occasional |
| A year-end charge | See below |

### The year-end charge

**Tested.** At the end of each season the game takes a large one-off amount from clubs with a healthy bank
balance. It tracks the balance, not that season's profit:

- about half the balance in most leagues;
- about 80% in one top European league and about 90% in the NHL.

It behaves like owners taking out accumulated cash. A club can show a "loss" for the year because of it
while being profitable on its operations. The exact rule, and the threshold above which it applies, are not
known.

### What healthy and unhealthy look like

One season, median club, all leagues loaded as playable:

| League type | Income | Wages | Result for the season |
|---|---|---|---|
| NHL | about 155M | about 115M | about +9M before the year-end charge |
| Top European league | 4M–10M | 1.9M–5.2M | +0.5M to +1.2M |
| College (as Junior) | 1.1M | 0.15M | +0.8M |
| Second-tier league, healthy | 1.7M–2.4M | 0.9M–1.4M | around break-even to +0.3M |
| Second-tier league with no TV money and a high minimum wage | 0.6M–1.8M | 0.7M–1.6M | −0.3M to −0.6M |

A rough guide from those seasons: leagues where wages took about 60% of income sat at break-even, with
around half their clubs losing money. Every league above 80% lost money. Healthy leagues were at 50% or less.

---

## 11. TV money and prize money

Set per league under Comp Rules, League Rules.

- **Tested: "Total TV money" is paid to every club in full.** It is not a pot that is shared out. A league
  set to 4,000,000 pays each of its clubs 4,000,000.
- **Tested:** it is paid on the first day of the season and again at each season rollover.
- **Tested:** changing it changes each club's income by exactly that amount in the first season. Six leagues
  whose clubs lost money were moved to break-even or a small profit by this alone.
- North American leagues store 0 here and the game works out its own figure per club. The rule is not
  known.
- "Total prize money" is a separate pot divided by the prize money type. We have not tested it.

**A caution.** Extra income raises the club's budget at the next rollover (section 7), and clubs then spend
more. In one league the clubs spent about a quarter of a large TV increase on wages within the first season.
Judge a change over two seasons, not one.

---

## 12. Playable and background leagues

**Tested.** The game runs a full economy only for leagues loaded as playable.

| League state | What the game runs |
|---|---|
| Playable | Every income and expenditure line |
| Background | No wage lines and almost no income, but some costs are still charged |
| Not simulated | A small charge only |

So in a long save, the clubs of background leagues drift deep into the red. That is how the game handles
leagues it is not simulating in detail. Two leagues that looked ruinous as background leagues were
comfortably profitable when loaded as playable.

**Only judge a league's finances from a save in which it was playable.**

---

## 13. The two new-game options

| Option | Ticked | Not ticked (default) |
|---|---|---|
| Disable Starting Wage Adjustments | Database salaries are used as entered | The game may raise or cut them |
| Disable Starting Financial Adjustments | Database balances and budgets are used as entered | The game sets starting balances and adjusts budgets |

- **Tested:** the wage option works as described (section 8).
- The financial option was left at its default in all our tests. The Pivot database stores a cash balance of 0
  for every club, so ticking it would most likely start every club with nothing.

---

## 14. Fixing common problems

| Symptom | Likely cause | Where to look |
|---|---|---|
| Board allows $0 for any contract | Club is Amateur | Club, Professional status |
| Players sign but staff cannot be hired | Staff limit sits on the minimum salary | Financial Info: Non-player spread factor, then staff maximums |
| Staff are paid far too much | Staff maximums too high for the spread factor | Same two settings, lowered together |
| A league's clubs lose money every year | Wages exceed what the league earns | League TV money; league minimum wage; then budgets |
| Most players in a league earn the same low wage | They have no database salary, or the league minimum equals most asks | Player salaries; league minimum wage |
| Stars on expiring deals start on huge wages | Starting wage adjustments are on | Tick Disable Starting Wage Adjustments |
| A new league row has no effect | It sits below the nation row | Financial Info table order |
| A league looks bankrupt in a long save | It was a background league | Reload with the league playable before changing anything |

When you change something, change one kind of setting at a time, check day one, then simulate at least one
full season with the league playable. Two seasons if you changed income.

---

## 15. What we still do not know

- What trims coach and head coach wages below the formula at higher-reputation clubs.
- The rule and threshold for the year-end charge.
- How the game sets TV revenue for North American leagues.
- What drives the "Other" income line.
- Why a handful of NHL clubs take a very large arena charge late in the season.
- Why a few database salaries are still lowered with wage adjustments disabled.
- How far into the red a club must be before its board stops a human manager signing people.
- Whether the Starting Financial Adjustments option keeps an authored budget through later seasons.

---

## 16. How this was tested

- Several new games were started from the same database with one setting changed each time, and the saves
  were read on day one, monthly, and after each season rollover.
- Club balances, budgets, contracts and finance lines were read directly from the save files and checked
  against the in-game finance and contract screens.
- One ten-year save and one two-season save with many leagues playable supplied the long-run figures.
- Statements labelled "From the game's code" come from reading the game's program logic for build 1669120.
  Other builds may differ.
