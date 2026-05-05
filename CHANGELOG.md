# Changelog

## [vNext+] — May 2026

### Added
- Multi-league pipeline expansion: SHL, HockeyAllsvenskan, Liiga, Mestis, DEL, DEL2, Czech Extraliga, MAXA liga, Swiss NL, Swiss League, Slovak Extraliga, 1. liga, KHL, VHL, ECHL, SPHL all modeled
- Junior pipeline: OHL, WHL, QMJHL, NCAA, USHL, NAHL with unified scouting-heavy approach
- Cross-pipeline reconciliation system for multi-league season players (callups, loans, mid-season trades)
- 630+ NHL prospects scouted from Scott Wheeler (The Athletic) and EliteProspects U23 prose
- Hand-curated CHL/NCAA scouting overlays covering top ~215 prospects per league
- Future draft class (2031–2100) cleanup for long-career save coherence
- Goalie career mental modeling (Phase 7b) mirroring skater approach
- Cross-pipeline reconciliation handling for multi-league seasons

### Changed
- Career history now drives mental attributes for all leagues, not just NHL/AHL
- All leagues use same absolute Current Ability scale for cross-league comparability
- ECHL/SPHL modeled for the first time

### Documentation
- Public documentation repository launched at github.com/xECK29x/pivot-pipeline
- Pipeline Guide HTML, Pipeline Flow visual, Data Sources by League with 51 player profiles
- NHL Pipeline Design Spec (gold standard) and Tier 2 Pipeline Design Spec (AHL/European adaptations) published

## [vNext] — March 2026

### Added
- Phase 8: CA & Role Suggestion Engine
- Phase 5b: Career-informed mental attributes (NHL/AHL skaters)
- NHL EDGE tracking integration for skating, shot speed, zone time, goalie positioning
- Bayesian shrinkage for shooting attributes
- Z-score / normal-CDF distribution mapping (replaced percentile-rank)
- 28-tier goalie CA system from High Franchise (200) down through ECHL Backup (60)
- Deployment floors for trusted NHL coaching-staff regulars

### Changed
- NHL/AHL pipeline rebuilt from the ground up
- Natural Stat Trick dependency fully removed; replaced with MoneyPuck + Evolving Hockey equivalents
- NHL/AHL merge logic switched to "quality-preferred" (NHL data wins by default for analytical depth)

### Fixed
- AHL games-played silent-zero merge bug

## [Earlier 2026] — Original re-rates pipeline

- Initial public release of analytics-driven re-rates approach
- Natural Stat Trick + MoneyPuck data foundation
