# Gulf Exits: Kuwait and Saudi Arabia's Routes to Market

Microsoft Fabric portfolio project. Working title (see D9).

Last updated: 21 September 2026 (v2, scope expanded to the Saudi Red Sea side)

---

## 1. The one-line pitch

Since 28 February 2026, the Strait of Hormuz has been effectively closed to routine commercial shipping. Kuwait has no way around it. Saudi Arabia had one: the East-West pipeline to Yanbu on the Red Sea, which ran at full capacity for six months until it was halted in September 2026. This project measures which of the Gulf's exits stayed open and which closed, built end to end on Microsoft Fabric, with a public dashboard that stays live after the Fabric trial ends.

## 2. Story in three acts (drives the dashboard layout)

1. **The main door closes.** Hormuz traffic drops from roughly 85 ships a day to single digits. A brief partial reopening from mid-June collapses in early July.
2. **Saudi opens the back door.** Tankers reroute to Yanbu. The East-West pipeline reaches its full 7 million barrels a day by late March.
3. **The back door closes too.** Drone attacks halt the pipeline in September and Yanbu loadings are suspended. Trade squeezes through what is left: ship-to-ship transfers offshore Oman, Fujairah, dark tankers.

Kuwait is the constant: the country with no back door. Saudi Arabia is the country that had one and lost it.

## 3. Decisions

See DECISIONS.md for the full log with reasons. Summary: Kuwait and Saudi Arabia are the two main characters, batch data from IMF PortWatch, Claude Code builds, React is the public face, Power BI on Direct Lake is the Fabric proof, all Fabric evidence captured before the trial ends.

## 4. Timeline (compressed, v2)

Trial started around 12 August 2026, so it ends around 11 October 2026 (confirm on Day 1, D11). Rule: everything that needs Fabric alive is finished and captured by 9 October. Everything after that uses exported files only.

| Dates | Phase | Milestone | Needs Fabric? |
|-------|-------|-----------|---------------|
| 21 to 22 Sep | Phase 0 | Repo, workspace, Lakehouse, Git integration, reference CSVs | Yes |
| 23 to 26 Sep | Phase 1 | Bronze tables for all locations, data profiled | Yes |
| 27 Sep to 1 Oct | Phase 2 | Silver, gold, KPI table, data quality | Yes |
| 2 to 4 Oct | Phase 3 | Weekly pipeline scheduled, JSON export to GitHub | Yes |
| 5 to 7 Oct | Phase 4 | Power BI on Direct Lake, screenshots and video | Yes |
| 8 to 9 Oct | Phase 7a | Data Agent tested, evaluation table and video captured | Yes |
| 10 to 11 Oct | Buffer | Final full export, final evidence capture | Yes |
| 12 to 23 Oct | Phase 5 | React dashboard live on Cloudflare Pages | No |
| 24 to 30 Oct | Phase 7b | README, article, LinkedIn post | No |

Phase 6 (live AIS) is dropped unless Phases 0 to 3 finish early (D10).

After the trial ends the dashboard data is frozen at the last export. The page states the data cut-off date clearly. If Fabric access continues (extension or paid capacity), the pipeline resumes and nothing else changes.

## 5. Data sources

### Primary: IMF PortWatch (free, weekly)

Joint IMF and University of Oxford platform. Processes satellite AIS signals. Publishes every Tuesday with daily data for the previous week. History from 2019. CSV download, plus an ArcGIS REST endpoint that we verify in Phase 1.

Locations, grouped by role in the story. Final list confirmed in Phase 1 against what PortWatch actually covers.

| Group | Locations | Why |
|-------|-----------|-----|
| The main door | Strait of Hormuz (chokepoint) | The closure itself |
| Kuwait (no back door) | Mina Al Ahmadi, Shuaiba, Shuwaikh | Local main character |
| Saudi Gulf coast | Ras Tanura, Dammam, Jubail | What Saudi lost on the Gulf side |
| Saudi Red Sea (the back door) | Yanbu, Jeddah | Rerouting and its collapse |
| Red Sea exits | Bab el-Mandeb, Suez Canal (chokepoints) | Where Red Sea cargo goes next |
| Other back doors | Fujairah, Sohar | UAE pipeline exit, Oman outside the strait |
| Reference | Cape of Good Hope (chokepoint) | Shows global rerouting |

### Reference data (small, hand-built CSVs)

1. `locations.csv`: name, PortWatch id, type (port or chokepoint), country, coast (Gulf, Red Sea, Gulf of Oman, other), story group, has_bypass flag, latitude, longitude.
2. `events.csv`: dated, factual, sourced list of key events since 28 February 2026. Includes Hormuz closure, April ceasefire, June memorandum and reopening, July breakdown, East-West pipeline at full capacity, pipeline halted, Yanbu loadings suspended.
3. `vessel_types.csv`: PortWatch categories mapped to three buckets (tankers, cargo, other).

### Optional context: Brent crude daily price (FRED CSV)

Only if Phase 2 finishes early.

### Dropped for now: aisstream.io live AIS

See D10. Key can still be created, it costs nothing.

### Known limitation (becomes a dashboard panel)

PortWatch counts AIS-broadcasting ships only. Ships that switch off their transponders ("dark" ships) are not counted, and some have been observed calling at Yanbu. Ship-to-ship transfers offshore Oman may not appear as port calls. Data arrives about a week late. The dashboard shows all of this.

## 6. KPIs

Headline: the exit board. One tile per door, each showing the latest 7-day average as percent of normal, with a status colour.

1. Strait of Hormuz
2. Kuwait ports (combined)
3. Saudi Red Sea (Yanbu and Jeddah combined)
4. Saudi Gulf coast (Ras Tanura, Dammam, Jubail combined)
5. Fujairah

Supporting views:

6. Timeline since January 2026, one line per door, events marked
7. Tankers vs cargo at Hormuz, percent of normal
8. Kuwait vs Saudi side by side: the "no back door vs lost back door" comparison
9. Red Sea exits (Bab el-Mandeb, Suez) vs Cape of Good Hope
10. "What this data cannot see": dark ships, ship-to-ship transfers, publication lag, last data date, missing days, revisions since last load

Every KPI is a count or a percent of a fixed baseline (D6). No statistics lecture needed.

## 7. Fabric architecture (unchanged by the scope expansion)

```
IMF PortWatch (CSV / REST)
        |
        v
 [Notebook 01_ingest_portwatch]
        |
        v
 Lakehouse lh_gulf: Files/bronze/  -->  Tables: bronze_chokepoints, bronze_ports
        |
        v
 [Notebook 02_build_silver]  -->  silver_daily_traffic (one row per location per day per bucket)
        |
        v
 [Notebook 03_build_gold]    -->  fact_daily_traffic, dim_location, dim_date, dim_event, kpi_current
        |
        v
 [Notebook 04_data_quality]  -->  dq_results
        |
        v
 [Pipeline pl_weekly_refresh, Tuesday 18:00 Kuwait]
        |
        +--> [Notebook 05_export_json] --> GitHub dashboard/public/data/ --> Cloudflare Pages
        +--> Semantic model (Direct Lake) --> Power BI report
        +--> Fabric Data Agent
```

Adding the Saudi side means more rows in `locations.csv`. No new notebooks, tables or pipelines.

## 8. Phases

### Phase 0: Foundations (21 to 22 Sep)

Build:
1. GitHub repo (public). Folders: `notebooks/`, `data/reference/`, `dashboard/`, `docs/`. Files: README.md, DECISIONS.md, CLAUDE.md, docs/PROJECT_PLAN.md.
2. Fabric workspace, trial license mode.
3. Lakehouse `lh_gulf` with Files folders `bronze/`, `reference/`, `export/`.
4. Trial end date confirmed and logged.
5. Git integration between the workspace and the repo.
6. Three reference CSVs, first draft, uploaded to `Files/reference/`.
7. CLAUDE.md so Claude Code knows the project, the rules and the folder layout.

Teach: Fabric as one building (done), Files vs Tables (done), Lakehouse vs Warehouse, Git integration.

Done when: a notebook in Fabric reads `locations.csv` and the notebook appears in GitHub after commit.

### Phase 1: Bronze ingestion (23 to 26 Sep)

Build:
1. Verify PortWatch download paths and REST endpoint. Confirm which locations in section 5 exist in PortWatch and their ids.
2. `01_ingest_portwatch`: raw CSV to `Files/bronze/<dataset>/<load_date>/`, then append to bronze Delta tables with `load_ts` and `source_file`. No transformation.
3. `00_explore`: date ranges, row counts per location, nulls, duplicates, gaps.

Teach: bronze as the evidence locker, Delta tables, reading a profile.

Claude Code prompts: ingest notebook, profiling notebook.

Done when: bronze tables cover every confirmed location, and DECISIONS.md records any location PortWatch does not cover.

### Phase 2: Silver and gold (27 Sep to 1 Oct)

Build:
1. `02_build_silver`: types, deduplication (latest load wins), vessel buckets.
2. `03_build_gold`: dimensions, `fact_daily_traffic` with 7-day average, baseline and percent of normal, `kpi_current` with one row per exit board tile. Grouped KPIs (Kuwait ports combined, Saudi Red Sea combined) use `story_group` from `dim_location`.
3. `04_data_quality`: freshness, missing days, revisions.

Teach: medallion end to end, star schema, fixed baseline, revisions as a trust signal.

Claude Code prompts: silver, gold (schemas as acceptance criteria), data quality.

Done when: the five exit board numbers can be checked by hand against the PortWatch portal, and `dq_results` catches an injected missing day.

### Phase 3: Orchestration and export (2 to 4 Oct)

Build:
1. `05_export_json`: `kpi_current.json`, `daily_traffic.json`, `events.json`, `locations.json`, `dq.json`, pushed to GitHub with a fine-grained token stored as a secret.
2. `pl_weekly_refresh`: notebooks 01 to 05 in order, fail fast, Tuesday 18:00 Kuwait, failure alert.

Teach: pipelines vs notebooks vs Dataflows Gen2, idempotence, secrets.

Done when: a scheduled run commits JSON to GitHub without you touching anything.

### Phase 4: Power BI on Direct Lake (5 to 7 Oct)

Build:
1. Semantic model on gold, Direct Lake, relationships, a few measures.
2. Two pages: exit board with timeline; Kuwait vs Saudi comparison.
3. Full-resolution screenshots and a 60 to 90 second video in `docs/`.

Teach: Direct Lake vs Import vs DirectQuery, measures vs columns.

Done when: Direct Lake confirmed in model settings, evidence committed.

### Phase 5: React dashboard (12 to 23 Oct, after the trial)

Design first (2 days, no code): storyboard for 5 seconds, 30 seconds, 2 minutes.

Visual direction: a map of the Arabian Peninsula showing both coasts, the Gulf and the Red Sea, with each exit as a marker that lights or dims by percent of normal. A time slider or scroll steps through the three acts, so the doors visibly close, open and close again. Below the map: the exit board, the Kuwait vs Saudi comparison, the Red Sea exits view, the "cannot see" panel, method and sources. Dark theme, restrained palette, motion only where it explains something.

Build: Vite, React, TypeScript in `dashboard/`, deployed to Cloudflare Pages, reading `public/data/*.json`. Lightweight SVG map, one chart library. Mobile checked on your phone.

Claude Code prompts: scaffold and data loading, one per section, final polish.

Done when: loads in under 2 seconds on mobile, and a stranger can say in one sentence what happened to Kuwait's and Saudi Arabia's exits.

### Phase 6: Live AIS layer (dropped, see D10)

### Phase 7: Data Agent and write-up

7a (8 to 9 Oct, needs Fabric): Data Agent on gold with instructions covering the baseline rule and data limitations. Ten test questions, evaluation table of right and wrong answers, short video.

7b (24 to 30 Oct): README with architecture diagram and screenshots, article on mohammedjouhar.com, LinkedIn post.

## 9. Working method

1. Each session starts with the phase and day. Claude replies with the goal, the concept (story mode), the steps (ASD-STE100) and the Claude Code prompt. The two teaching modes stay separate.
2. Every Claude Code prompt has four blocks: context, task, constraints, acceptance criteria. CLAUDE.md holds the standing context.
3. You run the code in Fabric, paste back the output or error, and say what you understood. Claude reviews before the next step.
4. Decisions go in DECISIONS.md on the day they are taken.
5. Friday recap: you explain the week's build back in your own words.

## 10. Risks

| Risk | Mitigation |
|------|------------|
| Trial ends before evidence is captured | All Fabric work scheduled before 9 Oct. Screenshots and video are the deliverable. |
| PortWatch does not cover a planned location | Confirmed in Phase 1, logged in DECISIONS.md, location dropped or replaced. |
| Situation changes suddenly (reopening or further closure) | Percent of normal against a fixed baseline works in every state. Events list is updated, not the logic. |
| PortWatch changes columns or endpoint | Bronze keeps raw files, silver has one mapping cell. |
| Trial capacity throttles notebooks | Small data, short Spark sessions, stop sessions when done. |
| Publishing about an active conflict from Kuwait | Aggregate public data, cited sources, events stated factually, no attribution of blame, no predictions, no commentary on any party. |

## 11. What "done" looks like

1. A public URL anyone in the GCC can open on their phone and understand in 10 seconds.
2. A GitHub repo with notebooks, reference data, pipeline and dashboard together.
3. Screenshots and video of Power BI on Direct Lake, a scheduled pipeline run, and a Data Agent evaluation.
4. A DECISIONS.md that reads like an engineer's log.
5. You can draw the architecture from memory and explain why each box exists.
