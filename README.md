# Game Product Analytics — Level Difficulty & Player Behavior
> **Context:** Intern screening test — End-to-end data analysis pipeline to identify problematic game levels and design appropriate visualizations for monitoring.

---

## 📌 Problem Statement

Given a player event log dataset, the goal is to:
- Identify levels with abnormal difficulty, high drop-off, or poor player experience
- Build a composite scoring framework to rank level difficulty objectively
- Visualize key metrics in an interactive dashboard for game designers to monitor and act on

---

## 🗂️ Dataset Overview

| Metric | Value |
|---|---|
| Total records | 116,644 |
| Unique users | 994 |
| Unique levels | 6,324 |
| Overall completion rate | 82.5% |
| Avg time played | 224.8 seconds |
| Level range | 1 – 9,046 |

**Event types:** `level_start` and `level_end`

**Data quality issues addressed:**
- 666 rows with missing `user_id` (anonymous sessions) → excluded from retention analysis
- 618 rows with missing `level` → excluded from progression analysis
- 4,396 duplicate events → deduplicated before counting
- `time_played` max of 23.1 hours (AFK/tracking error) → capped at 99th percentile
- 28 user-level pairs with 50+ attempts (e.g., Level 225 with 905 attempts) → flagged for balance review

---

## 🔍 Analytical Approach

### 1. Drop-off Analysis — 3-Method Framework

Rather than relying on a single metric, drop-off is measured from three angles for a more complete picture:

| Method | Definition | Strength |
|---|---|---|
| **Funnel Drop-off Rate** | % of users who played level N but never attempted N+1 | Measures inter-level retention |
| **Never-Won Rate** | % of users who attempted but never won a level | Identifies "stuck" players |
| **Avg Attempts to Win** | Average tries before first win (winners only) | Quantifies effort required |

> Only levels with ≥ 10 unique users were included to ensure statistical reliability.

### 2. Composite Difficulty Score

A normalized composite score (0–1 scale) built from three equally-weighted components:

$$\text{Difficulty Score} = \frac{\text{norm(Loss Rate)} + \text{norm(Avg Attempts)} + \text{norm(Median Time Played)}}{3}$$

**Difficulty tiers (percentile-based):**

| Label | Threshold |
|---|---|
| Very Hard | ≥ 90th percentile |
| Hard | 75th – 90th percentile |
| Medium | Median – 75th percentile |
| Easy | < Median |

---

## 🚨 Key Findings

### Critical Levels

| Level | Issue | Key Metrics | Priority |
|---|---|---|---|
| **225** | Extreme difficulty outlier | Win rate 2.1%, avg attempts 50+, difficulty score 0.79 | 🔴 High |
| **3, 4** | Early-game drop-off | Funnel drop-off 42% and 34% — players quit despite low difficulty | 🔴 High |
| **440, 477** | Mid-game difficulty cluster (Level 400–500) | Win rate ~3–4%, high attempts | 🔴 High |
| **129** | Session time anomaly | Median ~26 min/attempt vs. dataset avg ~2.7 min | 🟡 Investigate |
| **1, 9** | Onboarding churn | Drop-off >25% at game start | 🟠 Medium |

### Platform Insight
Mobile win rate (60.8%) is significantly lower than Tablet (93.8%). iOS (55.3%) and Android (63.2%) show the largest gap — suggesting UI/UX issues on smaller screens rather than difficulty problems.

---

## 📊 Power BI Dashboard

Two-page interactive dashboard built from Python-exported CSVs:

**Page 1 — Executive Overview**
- 5 KPI cards: Total Players, Win Rate, Avg Difficulty, Hard Levels, Total Levels
- Level distribution by difficulty tier (column chart)
- Top 10 most difficult levels by composite score (bar chart)
- Top drop-off levels with churn risk >15% (bar chart)

**Page 2 — Progression Deep Dive**
- Player success rate trend across levels (line chart)
- Difficulty score trend across levels (line chart)
- Win rate vs. avg attempts scatter plot (bubble size = total attempts, color = difficulty tier)
- Slicers: Country / Operating System / Game Mode

**DAX Measures:**
```dax
Total Players    = DISTINCTCOUNT(raw_events[user_id])
Overall Win Rate = DIVIDE(COUNTROWS(FILTER(raw_events, raw_events[is_success] = TRUE)), COUNTROWS(raw_events))
Avg Difficulty   = AVERAGE(level_metrics[difficulty_score])
Hard Levels      = COUNTROWS(FILTER(level_metrics, level_metrics[difficulty_label] IN {"Very Hard", "Hard"}))
```

---

## 🛠️ Tech Stack

| Layer | Tools |
|---|---|
| Data Processing & EDA | Python — Pandas, NumPy |
| Visualization (EDA) | Seaborn, Matplotlib |
| BI Dashboard | Power BI (DAX, Power Query) |
| Environment | Jupyter Notebook |

---

## 📁 Repository Structure

```
game-analytics/
│
├── game_analytics.ipynb       # EDA + feature engineering + export
├── data/
│   ├── level_full_metrics.csv # Aggregated level-level metrics
│   └── level_end_raw.csv      # Raw event log (level_end events)
├── dashboard/
│   └── game_dashboard.pbix    # Power BI dashboard file
├── report/
│   └── report.pdf             
└── README.md
```

---

## 💡 Recommendations

- **Level 225**: Audit for intentional boss design vs. unintentional spike. Add signposting if intentional; re-balance if not. Hint system strongly recommended.
- **Levels 3–4 (Onboarding)**: Problem is retention, not difficulty. Players win but don't return — review first-session UX and tutorial flow.
- **Level 400–500 cluster**: Cross-check with player progression timelines to determine if this is a designed mid-game wall or an accidental difficulty jump. Consider smoothing the difficulty curve.
- **Mobile platform**: Conduct UI/UX review specifically for small-screen devices; test performance on mid-range hardware.
- **17.5% mid-session exits**: Analyze exit timing within sessions to pinpoint the exact moment players quit — baseline data for game loop improvements.
