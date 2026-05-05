# Tier 2 Pipeline Design Spec — AHL & European Adaptations

> **Version:** vNext (March 2026)  
> **Reference:** NHL Pipeline Design Spec (Gold Standard)  
> **Authors:** xECK29x + Claude  
> **Status:** Active  
> **Covers:** AHL (3 scripts, ~4,915 lines), European (2 scripts, ~5,124 lines)

---

## Overview

The NHL pipeline is the **gold standard** — 20+ independent analytical sources, 10 modeling phases, and 10,700 lines of modeling code. The AHL and European pipelines must produce the same quality of EHM-compatible ratings with dramatically less data. This spec documents how each pipeline adapts the NHL architecture, where it fills gaps synthetically, and the specific techniques used to maintain rating credibility when signal is weak.

### Core Adaptation Principle

**The pipeline should never fabricate signal that doesn't exist.** When data is thin, the correct response is:
1. Widen confidence intervals (more regression toward neutral)
2. Lean more heavily on PA as a ceiling/floor anchor
3. Use role/archetype templates to shape distributions
4. Accept less precision (narrower rating range, more players clustered around 10–12)

What the pipeline must **never** do is treat absence of data as evidence of absence of skill. A player with no RAPM data isn't a bad defender — we just can't tell.

---

## Signal Availability Comparison

This table drives every adaptation decision. Each cell shows whether the signal exists and at what quality.

| Signal Category | NHL | AHL | SHL/HA | ELH/MAXA | Liiga/Mestis | DEL |
|---|---|---|---|---|---|---|
| **xG models (xGF, xGA, GSAx)** | ✅ Full (MP, EH) | ❌ None | ❌ None | ❌ None | ❌ None | ✅ Full |
| **RAPM (isolated impact)** | ✅ Full (EH) | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| **GAR/WAR decomposition** | ✅ Full (EH, LB) | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| **Puck tracking (EDGE)** | ✅ 6 categories | ❌ None | ❌ None | ❌ None | ❌ None | ✅ Full |
| **Corsi/Fenwick/Possession** | ✅ Full | ❌ None | ✅ CF%/FF%/CCF% | ✅ CF%/SCF% Rel | ✅ CF% (Liiga) | ✅ Full |
| **Primary assists (A1/A2)** | ✅ Full (EH) | ❌ None | ✅ A1/A2 | ❌ None | ✅ A1 | ✅ Full |
| **Shot-type breakdown** | ✅ Full (NHL.com) | ❌ None | ❌ None | ❌ None | ❌ None | ✅ Partial |
| **Faceoff data** | ✅ Full + ELO | ❌ None | ✅ FO% + OFO/DFO | ✅ FO% | ✅ FO% | ✅ Full |
| **Hits/Blocks per game** | ✅ Full (NHL.com) | ✅ Basic | ✅ H/BKS per GP | ✅ H/BKS | ✅ H/BKS | ✅ Full |
| **PIM detail (EFF/PERS)** | ✅ Full | ✅ Basic | ✅ EFF/PERS split | ✅ Basic | ✅ Basic | ✅ Full |
| **TOI with PP/SH/EV splits** | ✅ Full | ❌ None | ✅ TOI PP/SH/EV | ✅ TOI splits | ✅ TOI splits | ✅ Full |
| **Game-state goals (Trail/Tied)** | ❌ None | ❌ None | ✅ Trail/Tied/Lead | ❌ None | ❌ None | ❌ None |
| **Goalie save location** | ✅ EDGE zones | ❌ None | ❌ None | ❌ None | ❌ None | ✅ Zones |
| **Goalie GSAx** | ✅ MoneyPuck | ❌ None | ❌ None | ❌ None | ❌ None | ✅ xG model |
| **LB Hockey grades** | ✅ Full | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| **HockeyStatCards** | ✅ Full | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| **EP style tags** | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| **Career history (EP)** | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full |

---

## AHL Pipeline

### Architecture

```
Build AHL Master → Phase 1: Skaters → Phase 2: Goalies → Phase 3: CA/Role
```

The AHL compresses NHL Phases 2–6 into a single skater script because the data doesn't support the fine-grained signal separation that justifies separate phases. When you have 20+ analytical sources, it makes sense to model defense separately from physicality separately from skating. With one source (AHLTracker), the signals overlap too much for separate phases to add value.

### Script: `model_ahl_skaters_vNext.py` (2,278 lines)

**Single-source reality:** All performance data comes from AHLTracker.com — scoring (5v5, PP), on-ice metrics (5v5, PP, SH), advanced stats, and penalties. No xG, no RAPM, no tracking, no WAR.

#### Signal Tier System

The AHL model explicitly classifies every attribute by the quality of available signal:

| Tier | Quality | Attributes | Technique |
|---|---|---|---|
| **Tier 1** (strong signal) | Direct AHLTracker data supports modeling | OffensiveRole, DefensiveRole, Wristshot, Slapshot, Passing, Fighting, Aggression, Dirtiness, Agitation | Standard composite scoring from rate stats |
| **Tier 2** (moderate signal) | Partial data, supplemented by PA/role inference | Creativity, Deflections, Stickhandling, Deking, Checking, Pokecheck, Positioning | PA-weighted composites with role modifiers |
| **Tier 3** (biometric/synthetic) | No direct data — inferred from proxies | Faceoffs, Hitting | PIM as proxy for Hitting; Faceoffs synthetic from position + PA |
| **Discipline** | PIM data available | Sportsmanship, Temperament | PIM/60 rates, similar to NHL |
| **Physical** | No tracking data — biometric proxy | Acceleration, Pace, Agility, Balance, Stamina, NaturalFitness | BMI z-scores + TOI proxy + PA shaping |
| **Mental** | Career history (same as NHL) | Anticipation, Bravery, Decisions, WorkRate, Teamwork, Determination | Career signals from EP2EHM + PA-scaled base |
| **Character** | Career history (same as NHL) | Consistency, ImportantMatches, Pressure, Adaptability, Professionalism | Identical to NHL Phase 5b |

#### Gap-Filling Technique 1: PA-Shaped Score Bands

The most important AHL-specific technique. Instead of mapping a 0–1 score directly to 1–20, the rating range is **shaped by PA**:

```python
def pa_band(pa, family):
    """PA-shaped lo/hi band for score_to_rating mapping.
    
    PA 100 (low AHL): range ~3-15, median performer → 9
    PA 120 (mid AHL): range ~3-16, solid → 10
    PA 140 (high AHL): range ~3-16, strong → 10-11
    PA 160+ (prospect): range ~3-16, strengths → 11-12
    """
```

A PA-120 player scoring in the 50th percentile of AHL performance gets ~10. A PA-160 prospect scoring identically gets ~11–12 because their ceiling is higher and the data is consistent with a player who hasn't peaked yet.

**Why this matters:** Without PA banding, AHL ratings cluster too tightly around 9–11 because the data can't differentiate well. PA provides external information (scouting consensus on a player's ceiling) that helps spread ratings more realistically.

#### Gap-Filling Technique 2: TOI Proxy

AHL has no TOI data. The model synthesizes a TOI proxy from on-ice event rates:

```python
toi_proxy = 0.5 + (rank01(events_per_game) - 0.5) * gp_reliability
```

Players with more on-ice events per game are assumed to play more. This is imperfect but better than ignoring deployment entirely.

#### Gap-Filling Technique 3: Synthetic Faceoffs

AHL has **zero** faceoff data. The model generates synthetic faceoff ratings from:

```
faceoff_score = position_base + PA_offset + role_modifier + noise
  Centers: base 0.55 (maps to ~10-12)
  Wingers: base 0.30 (maps to ~5-8)
  Defense: base 0.15 (maps to ~3-5)
```

This is explicitly synthetic — no actual faceoff performance data exists. The ratings are defensible because they reflect positional expectations scaled by PA.

#### Gap-Filling Technique 4: PIM as Hitting Proxy

AHL has no hit tracking. Hitting is modeled from:

```python
hit_score = 0.18 + 0.16*pa01 + 0.10*gf_rel + 0.32*physical_role_flags
            + 0.22*size01 - 0.28*finesse - 0.18*sniper
```

PIM correlates with physical engagement. Body size and role archetype provide additional differentiation. This won't identify a player who throws 15 clean hits per game, but it correctly separates power forwards from playmakers.

#### Gap-Filling Technique 5: Mental Attributes from Career History

**This is the AHL pipeline's strongest gap-filler.** Career signals from EP2EHM (`career_signals_v2.csv`) provide the same multi-year, multi-league career data used in NHL Phase 5b. Since career history is player-level (not league-level), AHL players with EP coverage get the same quality of mental attribute modeling as NHL players.

Mental base values (Anticipation, Decisions, WorkRate, Bravery) are generated from PA + role + impact proxy, then **nudged ±3** by career Consistency/IM signals — identical to NHL Phase 5b's adjustment mechanism.

#### Six-Layer Guardrail System

1. **Reliability shrink** (GP-based, sqrt scaling)
2. **Role contracts** (Key/Essential/Irrelevant per archetype)
3. **PA banding** (AHL-calibrated lo/hi ranges)
4. **AHL technical ceiling enforcement** — family-specific caps: `skill: 17, def: 17, skate: 18, phys: 18`
5. **League ceiling** — ensures AHL ratings stay below NHL equivalents at the same percentile
6. **GP-proportional weighting** — low-GP players get wider confidence intervals

#### AHL-Specific PA Normalization

```python
def pa01(pa):
    """AHL population: median PA ~120, range 86-200.
    PA 120 → 0.40, PA 100 → 0.15, PA 140 → 0.60, PA 180 → 0.85"""
    return ((pa - 80) / 130).clip(0, 1)
```

This is calibrated differently from NHL (where the population skews higher) and European (where each league has its own center).

### Script: `model_ahl_goalies_vNext.py` (722 lines)

**Even thinner data than skaters.** Primary inputs: basic Sv%, GAA, GP, and limited shootout data from AHLTracker.

#### Goalie Attributes Modeled (14 core + 6 synthetic)

| Category | Attributes | Data Source |
|---|---|---|
| **Core goalie skills** | Positioning, Reflexes, Blocker, Glove, Recovery, Rebounds, OneOnOnes, Agility, Pressure, Stamina, WorkRate, Pokecheck, Anticipation, Decisions | Sv%, GAA, SA/game, shootout stats, GP workload |
| **Synthetic deterministic** | Acceleration, Pace, Balance, Fighting, Passing, Stickhandling | BMI proxy, PA-scaled defaults |
| **Hard-set to 1** | Checking, Deflections, Deking, Faceoffs, Creativity, Hitting, Movement, Flair, Slapshot, Wristshot | N/A — goalies don't use these |

#### Key Differences from NHL Phase 7

| Aspect | NHL Phase 7 | AHL Goalies |
|---|---|---|
| Data ecosystems | 5 independent sources | 1 (AHLTracker summary) |
| GSAx available | ✅ (primary quality signal) | ❌ (no xG model exists) |
| Shot-location SV% | ✅ (EDGE L/R/HD zones) | ❌ (overall SV% only) |
| Glove/Blocker split | ✅ (EDGE left/right) | ❌ (synthesized from overall SV%) |
| Shrink-to-mid target | 0.55 ("average+") | 0.55 (same — AHL goalies are still professionals) |
| Reliability legs | 4 (GP, GS, TOI, SA) | 2 (GP, SA approximated) |

#### Gap-Filling: Glove/Blocker Without L/R Data

NHL uses EDGE left/right save percentages. AHL has only overall Sv%. The model splits into Glove and Blocker using:
- Overall Sv% as the base index for both
- PA-scaled differentiation (higher PA → allow more spread between Glove and Blocker)
- Stable deterministic noise (from playerId hash) to prevent every goalie from having identical Glove = Blocker

This is explicitly acknowledged as less precise than EDGE-driven L/R splits.

### Script: `model_ahl_phase3_ca_role_vNext.py` (1,915 lines)

Mirrors NHL Phase 8 with AHL-appropriate composite signals and tier tables.

#### Key Differences from NHL Phase 8

| Aspect | NHL Phase 8 | AHL Phase 3 |
|---|---|---|
| Composite signal count | 20+ analytical pillars | ~7 pillars |
| Archetype count | 29 (base/finesse/physical) | 12 (simplified subset) |
| Goalie style engine | ✅ Full (analytics + EP tags) | ✅ Full (EP tags only for most goalies) |
| CA tier range | 60–200 (30 tiers) | 60–120 (AHL-calibrated tiers) |
| EP tag integration | ✅ Direct role override + soft boost | ✅ Same (EP coverage applies to AHL players) |

---

## European Pipeline

### Architecture

```
Build European Master → Skaters Model (all attrs) → Phase 3: CA/Role
```

Like AHL, the European pipeline compresses NHL Phases 2–7 into a single skater script. However, European leagues have **meaningfully richer data** than the AHL — possession metrics, primary assists, faceoff zone splits, PIM type breakdown, and real TOI — which allows the model to be significantly more precise than AHL.

### Script: `model_european_skaters_vNext.py` (3,697 lines)

#### Multi-League Architecture

The script supports 10 leagues across 5 countries via a `LEAGUE_CONFIG` system:

| Country | Leagues | Prefix | PA Center | CA Band | Tier |
|---|---|---|---|---|---|
| Sweden | SHL, HockeyAllsvenskan | `shl_`, `ha_` | 100, 75 | 80–120, 60–90 | 2, 5 |
| Czechia | Extraliga (ELH), MAXA | `elh_`, `maxa_` | 100, 75 | 80–120, 60–90 | 2, 5 |
| Finland | Liiga, Mestis | `liiga_`, `mestis_` | 100, 70 | 80–120, 55–85 | 2, 5 |
| Germany | DEL, DEL2 | `del_`, `del2_` | 100, 70 | 80–120, 55–85 | 2, 5 |

Each league has country-specific spine builders (`_build_spine_swe`, `_build_spine_cze`, `_build_spine_fin`, `_build_spine_ger`) that handle different column naming, different available stats, and different statistical conventions.

#### European Data Advantages Over AHL

These are the signals European leagues have that AHL lacks entirely:

| Signal | Impact on Modeling |
|---|---|
| **CF%/FF%/CCF%** | Real puck possession → Positioning, Checking get direct defensive signal instead of AHL proxy |
| **Primary assists (A1/A2)** | Clean playmaker identification → Passing, Creativity directly measurable |
| **FO% with OFO/DFO splits** | Real faceoff data → Faceoffs modeled from evidence, not synthetic |
| **PIM EFF/PERS split** | Clean aggression vs dirtiness separation → Aggression and Dirtiness properly differentiated |
| **TOI PP/SH/EV splits** | Real deployment context → DefensiveRole, OffensiveRole driven by actual usage |
| **Game-state goals (SHL)** | Trail/Tied/Lead scoring → unique clutch/pressure signal not available even in NHL |
| **Hits/Blocks per game** | Direct engagement data → Hitting and Checking have real signal instead of PIM proxy |

#### League-Calibrated PA Normalization

Each league has its own `pa01()` calibration:

```python
def pa01(pa, cfg):
    """League-calibrated PA normalization.
    SHL: PA 100 → ~0.40, PA 80 → ~0.05, PA 120 → ~0.65
    HA:  PA 75 → ~0.40, PA 60 → ~0.10, PA 90 → ~0.65"""
    center = cfg["pa_center"]
    lo, hi = cfg["pa_range"]
    return ((pa - lo) / (hi - lo)).clip(0, 1)
```

This is critical — a PA-100 player is elite in HockeyAllsvenskan but average in SHL. The same raw score must map to different ratings depending on the league context.

#### League-Specific Ceiling/Floor System

Each league defines family-specific caps that prevent lower-tier league players from reaching NHL-grade ratings:

```python
# SHL (Tier 2 — strong pro league, NHL-adjacent ceilings)
"ceil": {"role": 19, "skill": 18, "def": 18, "skate": 18, "phys": 18}

# HA (Tier 5 — developmental league, lower ceilings)  
"ceil": {"role": 18, "skill": 16, "def": 16, "skate": 17, "phys": 17}
```

A dominant SHL player can reach 18 in skill attributes. A dominant HA player caps at 16. This reflects the real hockey quality difference between the leagues.

#### Country-Specific Spine Builders

The core innovation of the European pipeline. Each country's data has different column names, different available stats, and different statistical conventions. The spine builder normalizes everything into a common signal dictionary:

**Swedish spine (`_build_spine_swe`):**
- CF%/FF%/CCF%/PDO available → direct possession signals
- A1/A2 splits → clean playmaker identification
- FO% with OFO/DFO → zone-contextual faceoff data
- Trail/Tied/Lead goals → unique clutch signal

**Czech spine (`_build_spine_cze`):**
- SCF% Rel (Scoring Chances For% Relative) → best single defensive signal in Czech data
- IGP%/IA1P% → individual contribution rates
- ZSR% → zone start context
- GF% replaces EV +/- for on-ice goal impact

**Finnish spine (`_build_spine_fin`):**
- Liiga has CF% but lacks CCF% (close-game Corsi)
- Mestis has CCF% + zone starts + avg skating speed per game
- No xG/xGE in either

**German spine (`_build_spine_ger`):**
- DEL has NHL EDGE-equivalent tracking data → skating metrics, shot velocity, xG model, puck possession
- DEL2 has basic stats only
- penny-del.org provides the tracking data source

#### Attribute Modeling Pattern (Score → Band → Rating)

The European pipeline follows the same `mk()` helper pattern as AHL:

```python
def mk(attr, family, score):
    lo, hi = pa_band(pa, family, cfg)  # league-calibrated
    return score_to_rating(score.clip(0, 1), lo, hi)
```

Each attribute builds a weighted composite score from 0–1 signals, then maps through PA-shaped bands to a rating. The weights are hand-tuned per attribute but follow a consistent pattern:

```python
# Example: Creativity in SHL
creativity_score = (
    0.36                          # base (everyone starts near mid)
    + 0.34 * pa01                 # PA shapes the ceiling
    + 0.10 * impact               # overall production level
    + 0.12 * (play01 - 0.5)       # playmaking signal (A1 rate)
    + 0.06 * (pp_driver - 0.5)    # PP deployment (creative players on PP)
    + 0.06 * (p1_pct - 0.5)       # primary point contribution
    + 0.04 * finesse_flag          # role context
    - 0.03 * physical_flag         # role context (inverse)
).clip(0, 1)
```

The `0.36` base ensures nobody drops below ~5–6. The `0.34 * pa01` ensures PA shapes the range. The remaining weights (summing to ~0.30) let actual performance data differentiate within the PA band.

### Script: `model_european_phase3_ca_role_vNext.py` (1,427 lines)

#### League-Specific CA Tier Distributions

Each league has its own tier table calibrated to its CA band:

**SHL Forwards (CA band 80–120):**

| Percentile | Tier | CA Range |
|---|---|---|
| Top 2% | SHL Elite F | 118–120 |
| Top 5% | SHL High First-Line F | 115–117 |
| Top 12% | SHL First-Line F | 110–114 |
| Top 25% | SHL Top-Six F | 105–109 |
| Top 50% | SHL Middle-Six F | 100–104 |
| Top 75% | SHL Bottom-Six F | 95–99 |
| Top 90% | SHL Depth F | 90–94 |
| Bottom 5% | SHL Fringe F | 85–89 |
| Bottom | SHL Replacement F | 80–84 |

**HockeyAllsvenskan Forwards (CA band 60–90):**

| Percentile | Tier | CA Range |
|---|---|---|
| Top 2% | HA Elite F | 88–90 |
| Top 8% | HA First-Line F | 85–87 |
| Top 20% | HA Top-Six F | 80–84 |
| Top 50% | HA Middle-Six F | 75–79 |
| Top 75% | HA Bottom-Six F | 70–74 |
| Top 90% | HA Depth F | 65–69 |
| Bottom | HA Replacement F | 60–64 |

#### Role Engine Adaptation

European Phase 3 uses the same archetype scoring as NHL Phase 8 but with **EP tags carrying more weight** because analytics signals are weaker. When EP scouts call an SHL player a "Power Forward" and the Hits/BKS data confirms physical engagement, the system has high confidence in that assignment — even without RAPM or LB Hockey grades to confirm isolated impact.

---

## Gap-Filling Techniques — Reference Summary

| Technique | Where Used | What It Solves |
|---|---|---|
| **PA-shaped score bands** | AHL, European (all) | Maps a 0–1 score to ratings where PA controls the ceiling, not just the floor. Prevents low-data clustering. |
| **TOI proxy from events** | AHL only | Synthesizes deployment context from on-ice event frequency when real TOI is unavailable. |
| **Synthetic faceoffs** | AHL only | Generates position-appropriate faceoff ratings from PA + role when zero FO data exists. |
| **PIM as hitting proxy** | AHL only | Uses penalty rate + body size + role flags to estimate physical engagement when no hit tracking exists. |
| **Career signals integration** | AHL, European (all) | Provides NHL-quality mental/character modeling regardless of league-level data limitations. Universal signal source. |
| **League-calibrated PA normalization** | European (all) | Same PA value means different things in different leagues. `pa01()` centers each league appropriately. |
| **Country-specific spine builders** | European (all) | Handles different column names, different available stats, and different statistical conventions across countries. |
| **League ceiling/floor system** | AHL, European (all) | Prevents lower-tier league players from rolling NHL-grade attributes. SHL skill cap = 18, HA skill cap = 16. |
| **Possession proxies** | European (SHL, ELH, Liiga) | CF%/FF%/SCF% provide puck possession signal that AHL completely lacks. Feeds Positioning and defensive attribution. |
| **Deterministic noise from playerId** | AHL, European | Prevents clustering when signal is weak. `zlib.crc32(playerId)` generates reproducible per-player variation. |
| **Role-template base values** | AHL, European (weaker leagues) | When performance data can't differentiate, role archetype provides the shape and PA provides the level. |
| **EP tag attribute nudges** | AHL, European (all) | Post-model ±1 to ±4 adjustments from EP scouting tags. Particularly valuable when analytics signals are thin. |
| **Bayesian FO% shrinkage** | European (SHL, ELH, Liiga, DEL) | European FO data exists but sample sizes are small. Bayesian shrinkage prevents extreme ratings from thin samples. |
| **Clutch goal percentage** | European (SHL only) | Trail/Tied/Lead goal splits provide a unique pressure/clutch signal not available anywhere else — including NHL. |

---

## Porting Checklist — Adding a New League

When extending the pipeline to a new league, follow this checklist:

### 1. Data Audit
- [ ] Inventory all available stats columns with sample data
- [ ] Classify each signal against the availability matrix above
- [ ] Identify the closest existing league template (SHL? AHL? Czech?)

### 2. League Configuration
- [ ] Define `LEAGUE_CONFIG` entry: prefix, PA center, PA range, CA band, tier, ceilings, floors, GP full season
- [ ] Calibrate `pa01()` for the league's PA population distribution
- [ ] Build league-specific CA tier distribution tables (percentile → tier label → CA range)

### 3. Spine Builder
- [ ] Create country/league-specific `_build_spine_*()` function
- [ ] Map all available columns to the standard signal dictionary
- [ ] Identify which signals are missing → document which gap-filling technique to use

### 4. Attribute Modeling
- [ ] For each of the ~39 modeled attributes, determine:
  - Is there direct data? → model from signal
  - Is there proxy data? → model from proxy with reliability discount
  - Is there no data at all? → use PA + role template (Tier 3 synthetic)
- [ ] Set league-appropriate ceilings per attribute family
- [ ] Calibrate NHL lift constants (if any) for the league level

### 5. Validation
- [ ] Run distribution QA — do ratings cluster appropriately for the league level?
- [ ] Spot-check known players — do elite SHL players separate from depth players?
- [ ] Compare cross-league: does a SHL star's rating look right relative to an NHL regular's?
- [ ] Verify ceiling enforcement — no lower-tier player should exceed the league cap

---

*This spec should be updated whenever an AHL or European modeling script is modified, or when a new league is added. Last updated: March 2026.*
