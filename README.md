# Pivot Pipeline

> **A public documentation repo for the Pivot/TBL Re-Rates analytics pipeline used in the Eastside Hockey Manager database.**

Welcome. This is the home of the documentation, design specs, dashboards, and interactive guides behind the analytics-driven player ratings shipped in the Premier Pivot Rosters / TBL Database for Eastside Hockey Manager.

If you've ever loaded a save and wondered _why_ a particular SHL forward is rated where he is, or _how_ junior prospects are evaluated when their stats are so noisy, or _what_ data sources actually drive the ratings for each league — this is where you can find out.

## 📖 Read the docs

**→ [Visit the documentation site](https://xeck29x.github.io/pivot-pipeline/)** _(GitHub Pages)_

The site includes:

- **[The Pipeline Guide](https://xeck29x.github.io/pivot-pipeline/pipeline-guide.html)** — a visual walkthrough of how raw data becomes player ratings, written for hockey fans first, technical readers second
- **[Data Sources by League](https://xeck29x.github.io/pivot-pipeline/data-sources.html)** — what data each of the 17 league pipelines uses, with three example players per league
- **[Pipeline Flow](https://xeck29x.github.io/pivot-pipeline/pipeline-flow.html)** — the overall architecture from data collection through final database output
- **[Design Specs](./specs/)** — the technical reference docs for the research team

## 🏒 What this project is

The Pivot Pipeline is a multi-stage analytics system that ingests real-world hockey data from 60+ sources across 17 league pipelines (NHL, AHL, ECHL, SPHL, 12 European leagues, KHL/VHL, CHL/OHL/WHL/QMJHL, NCAA, USHL, NAHL) and produces EHM-compatible player ratings for the entire playable hockey world.

The ratings cover:

- **~880 NHL players** — full analytics depth (MoneyPuck, Evolving Hockey GAR/RAPM, NHL EDGE tracking, LB Hockey, HockeyStatCards)
- **~2,000 AHL players** — single-source AHLTracker data with NHL-pipeline architecture adapted for thinner signal
- **~5,500 European players** across SHL, HockeyAllsvenskan, Liiga, Mestis, DEL, DEL2, Swiss NL, Swiss League, Czech Extraliga, MAXA liga, Slovak Extraliga, 1. liga
- **~1,500 KHL/VHL players**
- **~4,000 junior/college players** across CHL (OHL/WHL/QMJHL), NCAA, USHL, NAHL
- **~800 ECHL/SPHL players** — modeled for the first time in the May 2026 release
- **630+ NHL prospects** with hand-scouted overlay nudges from Scott Wheeler (The Athletic) and EliteProspects U23 reports

Every player is rated on the same absolute Current Ability scale, so an SHL first-liner at CA 110 sits exactly where you'd expect relative to an AHL top-six forward or NHL bottom-six player.

## 🤖 AI assistance disclosure

This project uses substantial AI assistance for scripting, modeling, and documentation work — originally ChatGPT, currently Claude (Anthropic). The AI does the implementation work; the human (xECK29x) provides hockey expertise, oversight, validation, and direction. This repo and the documentation it hosts are AI-assisted but human-curated.

## 📂 What's in this repo

```
pivot-pipeline/
├── README.md                  # You are here
├── LICENSE                    # MIT
├── docs/                      # GitHub Pages site
│   ├── index.html             # Landing page
│   ├── pipeline-guide.html    # The Pipeline Guide (visual)
│   ├── pipeline-flow.html     # Overall architecture flow
│   ├── data-sources.html      # Per-league data sources + example players
│   └── assets/                # CSS, fonts, images
├── specs/                     # Design specs (markdown)
│   ├── nhl-pipeline-spec.md   # Gold-standard NHL pipeline reference
│   ├── tier2-pipeline-spec.md # AHL + European adaptations
│   └── ca-tier-reference.md   # CA tier system v2.1
└── CHANGELOG.md               # Per-release history
```

The actual modeling code lives in a private repo and is shared selectively with research collaborators.

## 🐛 Found something wrong?

**Rating issues / data questions** — open a GitHub Issue here, or post on:
- TBL Forum (preferred for database-level questions)
- [r/EHM](https://reddit.com/r/EHM) on Reddit
- HFBoards EHM thread

**Methodology disagreements** — those are welcome too. The whole point of this repo existing is making the methodology auditable. If you think a formula is wrong, file an issue with your reasoning.

## 📜 License

This documentation is published under the [MIT License](./LICENSE). You're free to copy, adapt, redistribute, and reference any of this material with attribution.

The underlying code, data sources, and API integrations are not covered by this license and remain in a private repo.

## 🙏 Credits

**Project lead and maintainer:** [xECK29x](https://github.com/xECK29x)

**Modeling and documentation assistance:** Claude (Anthropic)

**Data sources:** MoneyPuck, Evolving Hockey, NHL.com, NHL EDGE, LB Hockey, HockeyStatCards, hockeyeloratings.com, AHLTracker, Sportality, hokej.cz, liiga.fi, Mestis, penny-del.org, nlicedata.com, hockeyslovakia.sk via Hudl, khl.ru, chl.ca, College Hockey News, HockeyTech, GameSheet, Elite Prospects, The Athletic (Scott Wheeler), and many more.

**Community contributors:** Backa, Exqua, ArchibauldUK, Filip85, Vik3tbotselektor, Citron, and the broader EHM community on TBL Forum, r/EHM, and HFBoards.

---

*Last updated: May 2026 (vNext+ release)*
