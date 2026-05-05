# NHL Re-Rates Pipeline — Script Design Specifications

> **Version:** vNext (February 2026)  
> **Pipeline:** NHL Track (Gold Standard)  
> **Authors:** xECK29x + Claude  
> **Status:** Active — reference for all league pipeline adaptations  
> **Total:** 10 modeling phases, ~10,700 lines across 9 core scripts

---

## Overview

This document specifies the design, inputs, outputs, and key formulas for every modeling script in the NHL pipeline. The NHL pipeline is the **gold standard** — any league adaptation (AHL, European, CHL, NCAA) should replicate this architecture as closely as data availability allows.

### Pipeline Execution Order

```
Build Master → P2 (Physicality) → P3 (Defense) → P4 (Skating) → P5 (Offense/Cognition) 
→ P5b (Career Mentals) → P6 (Faceoffs) → P7 (Goalies) → P7b (Goalie Mentals) → P8 (CA/Roles)
```

### Universal Design Standards

These rules apply to **every** modeling script in the pipeline:

1. **Goalie detection:** `Goaltender == 20` identifies goalies (project-wide standard)
2. **Hard fail on critical missing data** — scripts die if required columns are absent
3. **No pre-filtering by TOI/GP** — reliability weighting handles small samples inside models
4. **Rate-based metrics preferred** — per-60 rates, not raw counting stats
5. **Position-group z-scoring** — normalize within F/D/G pools, never across them
6. **`_model` column naming** — every modeled attribute produces `{Attribute}_model`
7. **Column placement** — `_model` columns inserted immediately after their base EHM attribute column
8. **PA-scaled floors** — higher PA players get higher minimum ratings (franchise prospect ≠ fringe)
9. **Output encoding** — all CSVs written as `utf-8-sig` for Excel compatibility
10. **Interactive output naming** — scripts prompt for output filename with sensible defaults

### Shared Mathematical Foundations

| Tool | Formula | Used In |
|---|---|---|
| **Per-60 conversion** | `value / TOI_minutes × 60` | P2, P3, P5, P7 |
| **Position-group z-score** | `(value − pos_mean) / pos_std` | All phases |
| **Reliability weighting** | `(TOI / soft_threshold)^0.75`, capped at 1.0 | P2, P3, P4, P5 |
| **Normal-CDF mapping** | `Φ(z) → percentile → 1 + pct × 19` | P3, P5, P6, P7 |
| **Bayesian shrinkage** | `(goals + prior_rate × k) / (shots + k)` | P5, P7 |
| **Shrink-to-mid** | `reliability × data_score + (1 − reliability) × neutral` | P2, P3, P4, P5, P7 |
| **Elite tail boost** | PA-gated, role-gated boost above pivot for PA ≥ 180 | P5, P7 |
| **PA floor function** | PA 200→14, 180→13, 160→12, 140→11, 120→10 (varies by phase) | P3, P4, P5 |

### Reliability Thresholds (Skaters)

| Position | Soft Threshold | Full Trust |
|---|---|---|
| Forwards | ~200 min TOI | ~428 min TOI |
| Defensemen | ~300 min TOI | ~604 min TOI |

Below soft threshold, reliability is `(TOI / soft)^0.75` (never zero, never a hard cutoff). Between soft and full, rises from ~0.60 → 1.00. Above full = 1.0.

---

## Phase 2: Physicality

**Script:** `model_phase2_physicality_vNext.py` (1,002 lines)  
**Input:** `master.csv` (from build script)  
**Output:** `masterp2.csv`  
**Reads from:** Combined master only (no external files)

### Purpose

Models behavioral tendencies — how often and how willingly a player engages in physical on-ice behaviors. These are **frequency/likelihood** attributes, not performance multipliers. They should be stable, role-appropriate, and resistant to noise.

### Attributes Produced

| Attribute | `_model` Column | Description |
|---|---|---|
| WorkRate | `WorkRate_model` | Effort/engagement: hits, blocks, takeaways, DZ commitment |
| Hitting | `Hitting_model` | Hit frequency (Hits/60) + contact absorption |
| Fighting | `Fighting_model` | Fighting frequency, role-weighted for enforcers |
| Aggression | `Aggression_model` | Penalty-taking tendency (PIM/60, majors, misconducts) |
| Dirtiness | `Dirtiness_model` | Penalty severity profile (misconducts, match penalties) |
| Agitation | `Agitation_model` | Drawing penalties / getting under opponents' skin |
| Bravery | `Bravery_model` | Willingness to absorb contact, play through hits |
| Sportsmanship | `Sportsmanship_model` | Inverse of penalty rate; clean play signal |
| Temperament | `Temperament_model` | Emotional control under pressure |

### Key Data Signals

- `nhlmisc_Hits/60`, `nhlmisc_BkS/60`, `nhlmisc_TkA/60`, `nhlmisc_GvA/60` — NHL.com miscellaneous rates
- `nhlpen_PIM/60`, `nhlpen_Minor/60`, `nhlpen_Major/60`, `nhlpen_Misconduct` — penalty data
- LB Hockey: Play Through Contact, Net-Front Play, Hits Absorbed/60, D-Zone Pressure Mitigation
- Evolving Hockey RAPM net impact (for WorkRate engagement)
- MoneyPuck on-ice xG differential (context)
- BMI z-scores from `HeightCm` / `WeightKg` (for implied physical presence)

### NHL Lift Constants

All Phase 2 attributes receive a small positive lift reflecting NHL-level baseline:

| Constant | Value | Rationale |
|---|---|---|
| `NHL_LIFT_WORKRATE` | +1.5 | NHL players have baseline effort requirements |
| `NHL_LIFT_HITTING` | +1.0 | Physical game expected at NHL level |
| `NHL_LIFT_BRAVERY` | +1.0 | NHL-level courage to compete |
| `NHL_LIFT_FIGHT` | +0.5 | Smaller — avoids creating fake enforcers |
| `NHL_LIFT_SPORT` | +0.5 | Already stable, minimal lift needed |

### Trait Floors

Realism floors (not PA clamps) — ensure NHL players don't drop unrealistically low:

| Attribute | Floor |
|---|---|
| WorkRate | 7 |
| Hitting | 3 |
| Bravery | 6 |
| Fighting | 1 |
| Aggression | 3 |

### Role Integration

Phase 2 uses role context more than most phases:
- **Enforcers:** Fighting and Aggression as KEY (floor 14), Hitting and Bravery as ESSENTIAL (floor 13)
- **Grinders:** WorkRate and Hitting as KEY
- **Playmakers/Snipers:** No physicality KEY attributes — follow standard distribution
- Role-aware fighting weight: enforcers get 2.0× weight on fighting signals, grinders 1.5×, standard 1.0×

### Adaptation Notes (Other Leagues)

- **Required signals:** Hit rates, penalty data, TOI. Any league with play-by-play can supply these.
- **LB Hockey signals** are NHL-only — other leagues substitute with available contact/physicality proxies or omit with penalty on Bravery precision.
- **NHL lift constants** must be calibrated per league level (SHL ~0.8×, AHL ~0.7×, CHL ~0.5×).
- **BMI data** is universally available from EliteProspects.

---

## Phase 3: Defensive Skills

**Script:** `model_phase3_defense_vNext.py` (1,160 lines)  
**Input:** `masterp2.csv` (Phase 2 output)  
**Output:** `masterp3.csv`  
**Reads from:** Combined master only

### Purpose

Models defensive skill execution — not effort or willingness (that's Phase 2), but how effectively a player defends. Can this player close passing lanes? Suppress chances? Position correctly?

### Attributes Produced

| Attribute | `_model` Column | Description |
|---|---|---|
| Positioning | `Positioning_model` | Defensive positioning quality — most important per research notes |
| Checking | `Checking_model` | Body-checking effectiveness (goalies forced to 1) |
| Pokecheck | `Pokecheck_model` | Stick-on-puck defensive skill |
| DefensiveRole | `DefensiveRole_model` | Willingness to commit to defensive play (goalies forced to 20) |

### Key Data Signals

| Signal | Column | Feeds |
|---|---|---|
| HockeyStatCards D Rating | `hscOD_D Rating` | Positioning, Checking |
| MoneyPuck on-ice xGA (derived) | `_mp_xga60` | Positioning, Pokecheck |
| Evolving Hockey Def GAR/60 | `ehgar_Def_GAR/60` | Positioning |
| Evolving Hockey xDef GAR/60 | `ehxgar_xDef_GAR/60` | Positioning |
| Evolving Hockey EV Defense Rate | `ehgGAR_EVD_GAR/100FA` | Positioning, Checking |
| RAPM xGA/60 | `_rapm_xga60` | Positioning, Pokecheck |
| LB Chance Suppression | `_lb_suppress` | Positioning |
| LB Retrievals | `_lb_retrievals` | Pokecheck |
| LB D-Zone Pressure Mitigation | `_lb_dz_mitigate` | Positioning |
| EH relative xGA/60 | `_eh_rel_xga60` | Positioning |
| EH Giveaway/Takeaway rates | `_eh_give60` / `_eh_take60` | Positioning |
| EDGE zone start/time percentiles | `edgeZone_*` | Checking, DefensiveRole |
| NHL.com Blocked Shots/60 | various | Checking, Pokecheck |
| Deployment: PK TOI, SH share | derived | DefensiveRole |

### Global Defensive Lift

Phase 3 applies a **+2 global lift** to skater defensive skill attributes. This reflects that NHL-level players should generally have respectable defensive fundamentals — the floor of NHL defensive ability is above the EHM scale's midpoint.

### PA Floor Table (Skills Only, Not DefensiveRole)

| PA Range | Floor |
|---|---|
| ≥ 200 | 14 |
| 180–199 | 13 |
| 160–179 | 12 |
| 140–159 | 11 |
| 120–139 | 10 |
| < 120 | 8 |

### PA Ceiling (Prevents Over-Inflation)

Skills are also **capped** by PA to prevent low-PA players from rolling unrealistically high:
- PA < 100: cap at 14
- PA 100–139: cap at 16
- PA 140–179: cap at 18
- PA ≥ 180: cap at 20

### Index Construction Pattern

Each attribute builds a composite index from weighted z-scored signals, then maps through normal-CDF to 1–20. Example for Positioning:

```
positioning_index = Σ(weight_i × z_score_i) for all defensive signals
→ percentile via pct_rank()
→ map_index_to_1_20() via normal-CDF shaping
→ apply PA floor/ceiling
→ apply role-based adjustments
→ clip to [1, 20]
```

### Soft Tether

A `soft_tether()` function prevents Positioning and Checking from diverging too far (max ~4 point spread), reflecting that these skills are correlated in real hockey — a player with elite positioning rarely has terrible checking.

### Adaptation Notes (Other Leagues)

- **Minimum required:** Some form of defensive impact metric (xGA, Corsi Against, or defensive zone suppression). Without this, Positioning loses its primary signal.
- **LB/RAPM signals** are NHL-only premium signals. Leagues without these rely more heavily on xGA and shot suppression.
- **EDGE zone data** provides deployment context. Substitute with TOI splits or zone-start data.
- **+2 global lift** should be league-calibrated (e.g., +1 for SHL, +0 for AHL/CHL).

---

## Phase 4: Skating & Conditioning

**Script:** `model_phase4_skating_conditioning_vNext.py` (881 lines)  
**Input:** `masterp3.csv` (Phase 3 output)  
**Output:** `masterp4.csv`  
**Reads from:** Combined master only

### Purpose

Models movement quality and physical sustainability. This is where NHL EDGE tracking data has the most direct impact — actual measured skating speeds and distances, not proxy estimates.

### Attributes Produced

| Attribute | `_model` Column | Primary Driver |
|---|---|---|
| Acceleration | `Acceleration_model` | EDGE burst counts ≥22mph + top speed percentile |
| Pace | `Pace_model` | EDGE top speed percentile + average speed |
| Agility | `Agility_model` | BMI z-score (inverse: lighter/shorter = more agile) |
| Balance | `Balance_model` | BMI z-score (direct: heavier/larger = more balanced) |
| NaturalFitness | `NaturalFitness_model` | EDGE distance per shift + workload sustainability |
| Stamina | `Stamina_model` | EDGE total distance + TOI-based workload metrics |

### EDGE Tracking Columns Used

| Column | Signal | Feeds |
|---|---|---|
| `edgeSpeed_top_speed_pct` | Top speed percentile | Pace, Acceleration |
| `edgeSpeed_num_22plus_mph` | Burst count ≥22mph | Acceleration |
| `edgeDist_es_per60_pct` | ES distance per-60 percentile | Stamina, NaturalFitness |
| `edgeDist_es_max_game_pct` | Max single-game distance | Stamina |
| `edgeDist_es_max_period_pct` | Max single-period distance | NaturalFitness |
| `edgeDist_all_total_pct` | Total distance percentile | Stamina |
| `edgeZone_es_def_zone_pctile` | Defensive zone time % | Context (low weight) |
| `edgeZone_es_off_zone_pctile` | Offensive zone time % | Context (low weight) |

### EDGE Fallback Logic

When EDGE data is missing (player not tracked, mid-season callup), Phase 4 falls back to:
- TOI-based workload proxies (TOI/GP, shift counts)
- Physicality signals from Phase 2 (Hitting_model, Bravery_model, Aggression_model)
- BMI-only for Agility/Balance

Fallback ratings receive a reliability penalty — the model explicitly trusts EDGE data more than proxy estimates.

### Balance/Agility — BMI-Based Model

**Hard fail** if `HeightCm` or `WeightKg` missing — these are the only Phase 4 attributes with a mandatory physical measurement requirement.

```
BMI = WeightKg / (HeightCm / 100)²
BMI_z = z-score within position group

Balance: higher BMI → higher Balance (heavier players are harder to move)
Agility: lower BMI → higher Agility (lighter/smaller players are more nimble)
```

### Goalie Handling

Per EHM research notes: "Skating attributes do not affect goalie performance significantly." Goalies receive conservative, narrow-band ratings with Agility having a special floor (~12) and tail allowance (18–20) for athletically elite netminders.

### 20-Rating Governance

`govern_20s_by_pct()` and `govern_19s_by_pct()` functions ensure only players in the top percentile of EDGE data can achieve 19–20 ratings. A 20 in Pace requires being in the 99th percentile of EDGE top speed — anything less is capped at 19.

### Adaptation Notes (Other Leagues)

- **EDGE data is NHL-only.** This is the biggest adaptation challenge.
- **DEL** has NHL EDGE-equivalent tracking data (skating metrics, shot velocity, xG, puck possession) — DEL Phase 4 can closely mirror the NHL version.
- **SHL/Liiga** lack tracking — Phase 4 must rely entirely on BMI + TOI proxies. Skating ratings will be less precise.
- **CHL** has no tracking data — use prospect scouting overlay data (EP grades, Scott Wheeler notes) for skating assessment.

---

## Phase 5: Cognitive & Offensive Skills

**Script:** `model_phase5_cognitive_offensive_unified_vNext.py` (1,936 lines)  
**Input:** `masterp4.csv` (Phase 4 output)  
**Output:** `masterp5.csv`  
**Reads from:** Combined master only

### Purpose

The largest and most analytically complex phase. Models how players create offense and make plays — playmaking instincts, attacking awareness, shooting technique, and vision.

### Attributes Produced

| Attribute | `_model` Column | Category |
|---|---|---|
| Anticipation | `Anticipation_model` | Cognitive |
| Decisions | `Decisions_model` | Cognitive |
| Creativity | `Creativity_model` | Cognitive |
| Movement | `Movement_model` | Cognitive/Skating hybrid |
| Passing | `Passing_model` | Technical |
| Stickhandling | `Stickhandling_model` | Technical |
| Wristshot | `Wristshot_model` | Technical |
| Slapshot | `Slapshot_model` | Technical |
| Deking | `Deking_model` | Technical |
| Deflections | `Deflections_model` | Technical |
| Flair | `Flair_model` | Offensive-only, PA ≥ 180 |
| OffensiveRole | `OffensiveRole_model` | Deployment tendency |
| PassTendency | `PassTendency_model` | Tactical tendency |

### Key/Essential Floor System

| Floor Type | Rating | Applies When |
|---|---|---|
| Technical KEY | 13 | Attribute is KEY for player's role |
| Technical ESSENTIAL | 12 | Attribute is ESSENTIAL for player's role |
| Mental KEY | 14 | Cognitive attribute is KEY for role |
| Mental ESSENTIAL | 13 | Cognitive attribute is ESSENTIAL for role |

### Role-Based Offensive/Defensive Clamps

Defensive archetypes receive soft caps on pure offensive attributes:
- Defensive roles: Passing capped at 16, Creativity capped at 15
- Offensive roles: No offensive caps, but defensive soft floor ensures minimum hockey sense
- All-around roles: Balanced — no aggressive caps in either direction

### Goalie Handling

Goalies receive conservative narrow bands for Anticipation, Decisions, Passing, Stickhandling only. All other Phase 5 attributes forced to 1 for goalies. PassTendency hard-set to 15 (neutral/pass-first outlet).

### Key vNext Improvements

1. **Z-score mapping** replaced percentile-rank mapping (prevents uniform ~38-per-bin distributions)
2. **Bayesian shrinkage** for shot-type percentages: `bayes_shpct(goals, shots, prior, k)` prevents 1-shot/1-goal inflation
3. **Elite tail boost** for PA ≥ 180, offensive-lean roles, already above average
4. **`compress_19_20_tail()`** — ensures 19–20 ratings are genuinely rare

### Adaptation Notes (Other Leagues)

- **Minimum required for cognitive attrs:** xGF differential or Corsi differential + some deployment context.
- **Minimum required for shooting attrs:** Shot-type breakdown with goals + attempts per type. Without this, Wristshot/Slapshot lose their primary signal.
- **LB Hockey signals** (Transition, Entry Volume, Puck Management, OZ Pressure) are premium NHL-only. Leagues without these lose precision on Passing and Creativity.
- **Flair** is tied to PA ≥ 180 regardless of league — franchise-level talent marker.

---

## Phase 5b: Career-Informed Mental Attributes

**Script:** `model_phase5b_career_mentals_vNext.py` (1,971 lines)  
**Input:** `masterp5.csv` (Phase 5 output)  
**Output:** `masterp5b.csv`  
**External reads:** `career_signals_v2.csv`, `draft_history.csv` (optional), `staff_award_history.csv` (optional)

### Purpose

Uses multi-year career history to model character/mental attributes that single-season analytics cannot capture. Runs AFTER Phase 5 to layer career-informed mentals on top of analytics-driven attributes.

### Attributes Created (New `_model` Columns)

| Attribute | Primary Signal | Default (Insufficient Data) |
|---|---|---|
| Consistency | Detrended PPG CV (skaters) / SV% CV (goalies) | 12 |
| ImportantMatches | Playoff PPG delta vs regular season (tier-weighted) | 12 |
| Pressure | Mirrors IM for skaters + Cup/award bonuses | 12 |
| Determination | 45% IM + 35% Elite Longevity + 20% career path + awards | 12 |
| Leadership | Captaincy history (C/A seasons × tier weights) + awards | 12 |
| Loyalty | Tenure ratio + team changes (NHL-only scope) | 10 |
| Ambition | International breadth + trajectory + awards + draft pedigree | 10 |
| Adaptability | Geographic diversity (regions played) | 10 |
| Professionalism | Career PIM rate + longevity + Sportsmanship correlation | 12 |

### Attributes Adjusted (Existing `_model` Columns, ±3 Max)

| Attribute | Signal | Max Adjustment |
|---|---|---|
| Anticipation | Consistency signal | ±3 |
| Decisions | Consistency signal | ±3 |
| WorkRate | Consistency + IM signal | ±3 |
| Bravery | IM signal | ±5 (wider for bravery) |
| Teamwork | Consistency + Leadership signal | ±2 |

### Playoff Delta — Tier Weighting

| Tier | Weight | Example Leagues |
|---|---|---|
| 1 (NHL) | 1.00 | NHL |
| 2 (KHL/SHL/NL) | 0.70 | KHL, SHL, National League |
| 3 (AHL/Liiga) | 0.50 | AHL, Liiga |
| 4 (CHL/NCAA) | 0.35 | OHL, WHL, QMJHL, NCAA |
| 5–6 | 0.20 | USHL, European lower tiers |
| 7+ | 0.10 | Junior A, lower leagues |

### Stanley Cup Bonus (Scaled)

| Cups Won | Determination Bonus | Pressure Bonus |
|---|---|---|
| 1 | +2 | +2 |
| 2 | +3 | +3 |
| 3+ | +4 | +4 |

### Award Bonus System

Awards accumulate raw points per attribute. Per-type diminishing returns (first 3 wins full value, additional at 50%). Raw points → bonus via:

| Raw Points | Bonus |
|---|---|
| 0 | 0 |
| 1–4 | +1 |
| 5–9 | +2 |
| 10–15 | +3 |
| 16–22 | +4 |
| 23+ | +5 |

### Player Matching

Career signals use `pkey` (First|Last|DOB) crosswalk to match against master's `playerId`. Accent stripping (`strip_accents()`) applied for fuzzy matching.

### Adaptation Notes (Other Leagues)

- **Career signals data** comes from EP2EHM export — works for any player with EliteProspects career history.
- **Award mapping** covers NHL, AHL, ECHL. European awards would need addition to `AWARD_ATTR_MAP`.
- **Captaincy data** comes from EP Player_stats.csv — league-universal.
- **Phase 5b is league-portable** with minimal adaptation since it reads from career history, not current-season analytics.

---

## Phase 6: Faceoffs

**Script:** `model_phase6_faceoff_vNext.py` (366 lines)  
**Input:** `masterp5b.csv` (Phase 5b output)  
**Output:** `masterp6.csv`  
**External reads:** `faceoff-elo.csv` (Faceoff ELO ratings, from hockeyeloratings.com)

### Purpose

Models faceoff ability as a standalone mechanical skill. Intentionally isolated from offensive/defensive skill modeling — a great faceoff man can be a terrible skater.

### Attributes Produced

| Attribute | `_model` Column | Description |
|---|---|---|
| Faceoffs | `Faceoffs_model` | Faceoff ability (1–20 scale) |

### Position-Specific Ranges

| Position | Range | Baseline | Logic |
|---|---|---|---|
| Centers | 9–20 | ELO-driven | Logistic curve from Faceoff ELO; real 18–20 tail |
| Wingers | 4–15 | 7.2 baseline | Softened shrinkage; not a hard pile-up at baseline |
| Defensemen | 2–7 | ~3 | Tight range; D rarely take faceoffs |
| Goalies | 1 | Fixed | Goalies don't take faceoffs |

### Position Detection

Does NOT rely on `nhlfo_Pos` (often NaN). Uses:
1. Role prefix (Centre/Winger/Defenceman/Goalie) as primary
2. PosGroup (F/D/G) as fallback
3. `nhlfo_Pos` only if it contains real values

### Small Modifiers

WorkRate and Stamina z-scores from earlier phases provide ±1 adjustments — faceoff consistency requires sustained effort.

### Adaptation Notes (Other Leagues)

- **Faceoff ELO** is NHL-only. Other leagues need raw FO% with Bayesian shrinkage: `bayes_pct(wins, attempts, prior=0.50, k=200)`
- **NHL.com faceoff data** (`nhlfo_*` columns) provides GP, FO, FOW, FOL, FOW%.
- For leagues without faceoff data, use DB defaults or set Centers to 10–12 (league-average) and non-centers to position-appropriate ranges.

---

## Phase 7: Goaltending

**Script:** `model_phase7_goalies_vNext.py` (727 lines)  
**Input:** `masterp6.csv` (Phase 6 output)  
**Output:** `masterp7.csv`  
**Reads from:** Combined master only (all goalie data pre-merged in build)

### Purpose

Models goalie-specific attributes using five independent data ecosystems. Skaters pass through unchanged.

### Attributes Produced (Goalies Only)

| Attribute | `_model` Column | Primary Signal |
|---|---|---|
| Reflexes | `Reflexes_model` | High-danger SV%, GSAx |
| Positioning | `Positioning_model` | EDGE shot-location SV%, GSAx, overall positioning |
| Glove | `Glove_model` | EDGE left-side SV% (Bayesian smoothed) |
| Blocker | `Blocker_model` | EDGE right-side SV% (Bayesian smoothed) |
| Recovery | `Recovery_model` | Rebound control metrics, 2nd-chance SV% |
| Rebounds | `Rebounds_model` | Rebound generation rate, uncontrolled rebounds |
| OneOnOnes | `OneOnOnes_model` | Shootout SV% |
| Agility | `Agility_model` | Athletic metrics + floor ~12, tail 18–20 |
| Pressure | `Pressure_model` | High-stakes performance (analytics-driven, refined by P7b) |
| Stamina | `Stamina_model` | Rest splits (back-to-back performance) |
| WorkRate | `WorkRate_model` | Goalie-specific workload (starts, SA/60, GP pace) |

### Five Data Ecosystems

1. **NHL.com Advanced** — Sv%, GAA, SA/60, quality starts, rest splits, saves by strength, shootout stats
2. **MoneyPuck Goalies** — GSAx, high-danger Sv%, expected saves model
3. **Evolving Hockey GAR** — Goalie-specific Goals Above Replacement
4. **EDGE Goalie Tracking** — Save % by shot location, lateral movement
5. **LB Hockey Goalie WAR** — Independent WAR evaluation

### Four-Leg Reliability

```
reliability = blend(GP_leg, GS_leg, TOI_leg, SA_leg)
```
Each leg scaled 0–1, then blended. Full trust at ~50 starts + 1,500 SA.

### Bayesian Save% Smoothing

```
smoothed_sv = (saves + prior_sv × prior_shots) / (shots_against + prior_shots)
```
Where `prior_sv` = goalie's own overall SV%, `prior_shots` = 200–300 depending on segment. Prevents small-sample segments from spiking.

### "Shrink to Average+" Target

Core goalie skills shrink toward **0.55** (not 0.50) when evidence is weak. NHL goalies as a pool should not cluster as "below average" — the midpoint for an NHL goalie is already world-class.

### Adaptation Notes (Other Leagues)

- **MoneyPuck GSAx** is the single most important goalie metric. Leagues with xG models (DEL, SHL potentially) can provide an equivalent.
- **EDGE goalie tracking** is NHL-only. Without it, Glove/Blocker lose L/R precision — fall back to overall SV%.
- **AHL goalies** have only basic Sv%/GAA/GP from AHLTracker. Use wider confidence intervals and more aggressive shrinkage.

---

## Phase 7b: Goalie Career Mentals

**Script:** `model_phase7b_goalie_mentals_vNext.py` (795 lines)  
**Input:** `masterp7.csv` (Phase 7 output)  
**Output:** `masterp7b.csv`  
**External reads:** `career_signals_v2.csv`, `draft_history.csv` (optional)

### Purpose

Mirrors Phase 5b for goalies only. Career-informed mental attributes layer on top of Phase 7's analytics-driven goalie attributes.

### Attributes Produced (Goalies Only)

| Attribute | Primary Signal | Note |
|---|---|---|
| Consistency | SV% coefficient of variation | League-filtered (reliable leagues only) |
| ImportantMatches | Playoff SV% delta | Positive = clutch |
| Pressure | **60% P7 analytics + 40% career** | Unique blend — P7 already has analytics-based Pressure |
| Determination | IM + longevity + path + draft grit + cups | Same formula as skaters |
| Leadership | Captaincy history | **Capped at 14** unless actual C/A history (rare for goalies) |
| Loyalty | Tenure ratio + team changes | |
| Ambition | International + trajectory + awards | |
| Professionalism | PIM rate + longevity | |
| Adaptability | Geographic diversity | |

### Goalie-Specific Consistency (SV% CV)

Uses SV% coefficient of variation instead of PPG CV. Requires careful league filtering — many leagues don't report saves/GA reliably. Detection rule: if `Saves == 0 AND GA == 0 AND GP > 0`, stats are not reported for that league/season — exclude entirely.

### Pressure Blend (Unique to 7b)

```
goalie_pressure = 0.6 × P7_analytics_pressure + 0.4 × career_pressure_signal
```
This reflects that goalie pressure is partly captured by current-season shot volume/situation data (P7) and partly by career playoff history.

### Adaptation Notes (Other Leagues)

- Identical adaptation profile to Phase 5b — reads from career history, not current-season analytics.
- Goalie career filtering for SV% CV is critical — apply `confirmed_reliable_leagues` filter.

---

## Phase 8: CA Suggestion & Role Engine

**Script:** `model_phase8_ca_role_suggestion_vNext.py` (2,866 lines)  
**Input:** `masterp7b.csv` (Phase 7b output)  
**Output:** `masterp8_master.csv`, `masterp8_roles.csv`, `masterp8_FLAGGED.csv`, `masterp8_CA_summary.csv`, `masterp8.html`  
**External reads:** `player_styles.csv` / `ep_styles.csv` (EP style tags), `role_exceptions.csv` (optional manual overrides)

### Purpose

The "quality inspector" — evaluates each player's full modeled attribute profile against their database CA, suggests CA adjustments with tier labels, and scores every player against 29 archetype fingerprints for role suggestion.

### CA Suggestion Pipeline

```
Composite Z-scores → Percentile (per position pool) → Tier Assignment →
Spread Within Tier → GP Confidence Blend → Box Score Production Floor (F only) →
Career Floor (all positions) → Star Power Floor (named list) →
Tier Label Re-derivation → CA Delta → Write to Master
```

### CA Tier Tables

**Forwards:** 30 tiers from High NHL Franchise (200) to Low ECHL Bottom 6 (60)  
**Defense:** 30 tiers from High NHL Franchise (200) to Low ECHL Bottom Pair (60)  
**Goalies:** 30 tiers from High NHL Franchise (200) to Low ECHL Backup (60)

### GP Confidence Blending

| GP | Blend |
|---|---|
| ≥ 30 (MIN_GP_FULL) | 100% suggested CA |
| 8–29 | Linear blend between suggested and existing CA |
| < 8 (MIN_GP_PARTIAL) | 100% existing CA (insufficient data) |

### Role Archetype Engine — 29 Archetypes

**Forward (14):** Sniper, Sniper (Finesse), Sniper (Physical), Playmaker, Playmaker (Finesse), Playmaker (Physical), Defensive, Defensive (Finesse), Defensive (Physical), Power Forward, All Around, Grinder, Enforcer, and additional sub-variants.

**Defense (14+):** Offensive, Offensive (Finesse), Offensive (Physical), Pointman, Pointman (Finesse), Pointman (Physical), Playmaker, Playmaker (Finesse), Defensive, Defensive (Finesse), Standard, Standard (Physical), and additional sub-variants.

Each archetype is a `dataclass` with `signal_expectations` (dict of signal → direction + weight) and metadata.

### EP Style Tag Integration

- Tags parsed from `ep_styles.csv` (EP2EHM export)
- Normalized via accent stripping + lowercase matching
- **Direct role mapping:** Tags naming a specific archetype (e.g., "Sniper", "Power Forward") override analytics if score is within `EP_OVERRIDE_MAX_GAP` (12 points) of the analytics best
- **Soft score boosts:** 5–15 point bonuses to matching archetypes
- **Confidence scaling:** 1 tag = 60%, 2 tags = 85%, 3+ tags = 100%
- **Conflict resolution:** `_resolve_ep_conflicts()` uses goalshare as tiebreaker

### Deployment Compatibility Adjustment

Post-scoring adjustment ensures role suggestions don't break EHM AI coaching:
- PP specialist tagged "defensive" → would make OffRole irrelevant → no PP time
- PK specialist tagged "sniper" → would make DefRole irrelevant → no PK time
- Scores adjusted to maintain deployment compatibility

### Goalie Role Suggestion (3 Styles)

Goalies scored for Butterfly, Mixed, Hybrid using analytics signatures + EP goalie tags:
- **Athletic Goalie** → Mixed (+15 bonus)
- **Butterfly Goalie** → Butterfly (+15 bonus)
- **Positional Goalie** → Butterfly modifier (Positioning +1, Anticipation +1)
- **Hybrid Goalie** → Hybrid (+15 bonus)

### HTML QA Output

Generates a self-contained HTML file with:
- CA suggestion leaderboard (sorted by delta)
- Role suggestion table with archetype scores
- Attribute diff viewer (DB vs modeled values)
- EP tag audit panel
- Filterable by team, role, position

### Adaptation Notes (Other Leagues)

- **CA composite signals** must be calibrated per league — NHL uses 20+ analytical pillars, AHL uses ~7, European varies.
- **Tier tables** need league-appropriate CA ranges (AHL floor ~60, European floor varies).
- **Role archetypes** are league-universal — same 29 archetypes work for any hockey league.
- **EP tags** are available for any player with EliteProspects coverage.
- **Deployment compatibility** logic is EHM-universal.

---

## Cross-Pipeline Adaptation Matrix

For adapting the NHL gold standard to other league pipelines:

| Phase | NHL (Full) | DEL (Rich) | SHL/Liiga | AHL | CHL/NCAA |
|---|---|---|---|---|---|
| P2 Physicality | Full (26+ signals) | Near-full (tracking available) | Partial (no LB/RAPM) | Partial (AHLTracker only) | Minimal (EP + league stats) |
| P3 Defense | Full (LB + RAPM + xGA) | Near-full (xG available) | Partial (basic xGA) | Basic (AHLTracker) | Scouting overlay only |
| P4 Skating | EDGE tracking (direct) | DEL tracking (direct) | BMI + TOI proxy | BMI + TOI proxy | Scouting overlay |
| P5 Offense | Full (67+ EH signals) | Moderate | Moderate | Basic | Scouting overlay |
| P5b Mentals | Full (career signals) | Full (EP career) | Full (EP career) | Full (EP career) | Partial (young players) |
| P6 Faceoffs | ELO + FO% | FO% + Bayesian | FO% + Bayesian | FO% (basic) | FO% (basic) |
| P7 Goalies | 5 ecosystems | 2–3 sources | 1–2 sources | Basic (SV%/GAA) | Minimal |
| P7b G-Mentals | Full (career) | Full (career) | Full (career) | Full (career) | Partial |
| P8 CA/Roles | Full composite | Moderate | Moderate | Limited composite | Scouting-driven |

---

*This spec should be updated whenever a modeling script is modified. Last updated: March 2026.*
