# Tokyo Refill Infrastructure Decision Tool

**A map-based decision-support tool for identifying refill deserts and optimising new refill point placement across Tokyo**

[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-deployed-blue)](https://aneesafarhan.github.io/minerva-hackathon-track-a/)

---

## Overview

Japan generated **7.69 million tonnes of plastic waste in 2023**, and despite an official recycling rate above 85%, only **6% is actually recycled and reused domestically** — the rest is thermally recycled, meaning incinerated. Japan is the world's second-largest producer of plastic packaging waste per capita, with containers and packaging making up **64% of plastic waste discharged at waste stations** in FY2023.

**mymizu**, Japan's water refill app, has built an open dataset of over 200,000 refill spots worldwide with dense coverage across Tokyo — one of the most granular pictures of sustainable infrastructure in the city. But the network has gaps, and no existing tool identifies *where* new refill points would meaningfully reduce single-use bottle dependence.

Built for the **Minerva Hackathon Technical Track A — Build for Zero Waste**, this tool answers one focused planning question: **where should new refill points be added in Tokyo to reduce single-use bottle dependence most effectively?**

---

## Live Demo

**[https://aneesafarhan.github.io/minerva-hackathon-track-a/](https://aneesafarhan.github.io/minerva-hackathon-track-a/)**

The interactive dashboard lets you explore:
- A live heatmap of refill deserts and demand hotspots across Tokyo's 23 Special Wards
- The top 15 recommended new refill sites, ranked and explained in plain language
- Adjustable scoring weights (gap / demand / spacing) that re-rank recommendations live
- A before/after coverage simulator for any number of new refill points (0–30)

---

## Problem Statement

| Dimension | Detail |
|-----------|--------|
| Scope | Tokyo's 23 Special Wards |
| Existing refill points | 112 (mymizu-modeled distribution) |
| Candidate new sites | 64 real locations (stations, libraries, parks, universities) |
| Analysis grid | ~2.8k hex cells, ~1.1 km resolution |
| Challenge | No existing tool combines refill coverage, building/demand data, and ward statistics into a single actionable ranking of where new refill points should go |

---

## System

### Stage 1 — Hex Grid & Scoring
A uniform hex grid (~1.1 km cells) is built over the Tokyo 23-ward bounding box. Every hex and every candidate site receives three 0–100 scores:

- **Gap score** — distance to the nearest existing refill point (`min(100, nearest_m / 12)` for candidates, `/15` for hex cells)
- **Demand score** — exponentially decaying weighted sum of proximity to 48 transit/commercial demand anchors, matching a ~0.9 km walkable service radius
- **Combined score** — user-controlled weighted sum of gap, demand, and spacing (default weights: 0.50 / 0.30 / 0.20)

A hex is flagged a **desert** past 800 m from the nearest refill point (~10 min walk), and **covered** within 500 m — the threshold the simulator reports against.

### Stage 2 — Greedy Ranking with Spacing Correction
Naive top-N-by-score selection clusters recommendations in the same underserved pocket. Instead, candidates are selected greedily one at a time: after each pick, every remaining candidate's spacing score is recalculated against everything already selected (existing points + prior picks), spreading recommendations across the city without hard distance constraints.

### Stage 3 — Scenario Simulation
For any chosen N (0–30 new points) and any scoring weights, the simulator computes before/after coverage at 500 m, mean nearest-distance, deserts resolved, and high-demand areas newly served — rendered as a dual-line coverage curve across 200–1500 m thresholds, so the full impact curve is visible, not just a single coverage number.

---

## Repository Structure

```
minerva-hackathon-track-a/
├── index.html                      # The application
├── README.md
├── LICENSE                         # MIT
├── .gitignore
├── .nojekyll                       # Tells GitHub Pages to skip Jekyll
├── .github/
│   └── workflows/
│       └── deploy.yml              # Auto-deploy to GitHub Pages on push to main
└── data/
    ├── refill_points.json          # 112 existing refill points
    ├── candidate_sites.json        # 64 candidate new locations
    ├── tokyo_wards.json            # 23 wards + real stats
    └── demand_anchors.json         # 48 demand-proxy anchors
```

---

## Tech Stack

| Technology | Role |
|------------|------|
| Leaflet 1.9.4 | Map rendering, layer management |
| Turf.js 6.5.0 | Hex grid generation, geodesic distance |
| Chart.js 4.4.1 | Before/after coverage curves |
| Vanilla JS (~1000 lines) | Scoring, ranking, simulation, UI |

Pure static site — no backend, no build step, no framework. All libraries load from public CDNs; only the initial page load needs internet, after which the hex grid computes and stays cached locally.

---

## Data Sources & Attribution

- **mymizu Refill Map** — [github.com/mymizu/mymizu-web](https://github.com/mymizu/mymizu-web) — open dataset behind the mymizu app, 200,000+ refill spots worldwide with detailed Japan coverage. `data/refill_points.json` is a curated demo set modeled on the real mymizu distribution; production use should pull the live API feed.
- **mymizu / Social Innovation Japan** — [mymizu.co/home-en](https://mymizu.co/home-en) — the NPO behind mymizu. Attribution required for any use of their dataset.
- **PLATEAU Project (MLIT)** — [mlit.go.jp/plateau/en](https://www.mlit.go.jp/plateau/en/) — Japan's official 3D city model (CC BY 4.0). In production, building footprints and floor-area from the 23-ward CityGML dataset would replace the coarse anchor-based demand proxy used here.
- **OpenStreetMap** — [openstreetmap.org](https://www.openstreetmap.org/) — the 48 demand anchors in `data/demand_anchors.json` are real station/landmark coordinates.
- **Tokyo Metropolitan Government Open Data** — [portal.data.metro.tokyo.lg.jp](https://portal.data.metro.tokyo.lg.jp/) — ward boundaries, population, daytime population; `data/tokyo_wards.json` uses 2024 TMG estimates.
- **Ministry of the Environment Japan** — [env.go.jp](https://www.env.go.jp/en/) — FY2023 national plastic and municipal waste survey data, used for framing.

The 64 candidate sites in `data/candidate_sites.json` are real Tokyo locations weighted toward underserved outer wards.

---

## Key Design Decisions

**Why a hex grid instead of ward boundaries?**
Tokyo's wards are huge and uneven (Chiyoda 11 km², Ota 62 km²). A uniform ~1.1 km hex grid gives consistent spatial resolution that matches how planners actually think about walkable service areas.

**Why greedy selection over an ILP solver?**
64 candidates × 30 picks is fast and fully explainable with greedy selection — a meaningful property when presenting to non-technical stakeholders. An ILP formulation would be a natural production upgrade.

**Why an anchor-based demand proxy instead of real footfall data?**
Building-level floor area and ridership data (PLATEAU, MLIT) would be more accurate, but the 48 hand-weighted transit/commercial anchors give a reasonable v1 proxy without requiring a full CityGML pipeline in a one-day hackathon window.

**Why show the full coverage curve instead of a single number?**
A scenario that modestly improves 500 m coverage but dramatically reduces deep-desert hexes would look unremarkable at a single threshold. The dual-line chart across 200–1500 m makes that kind of improvement visible.

---

## Extending for Production

- **Live data pipeline** — swap `data/refill_points.json` for the live mymizu API feed; extract building floor area from PLATEAU CityGML to replace the anchor-based demand proxy; pull real ward polygons from TMG Open Data.
- **Better demand proxy** — combine OSM amenity density, JR/Metro ridership (MLIT), daytime population (TMG), and PLATEAU floor area into a normalized, configurable-weight demand score.
- **Real cost constraints** — add per-site install cost and replace greedy selection with a budget-constrained knapsack over total impact.
- **Equity layer** — add a ward-level socioeconomic index and surface scenarios that explicitly prioritize lower-income wards.

---

## License

MIT. See `LICENSE`. Data attributions: mymizu / Social Innovation Japan, OpenStreetMap contributors, CARTO, MLIT PLATEAU, Tokyo Metropolitan Government.
