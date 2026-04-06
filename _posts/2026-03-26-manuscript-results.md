---
title: OnboardingFlow Dashboard — Interactive Visualization of the netdev Contributor Funnel
description: Journal of an project idea based on the dataset LKML5Ws
date: 2026-03-26 15:25:00 +/-TTTT
categories: [Software Visualization]
tags: [linux-kernel, netdev, onboarding, contributor-funnel, streamlit, data-visualization]
author: <author_id> # for single entry
---

# OnboardingFlow Dashboard — Interactive Visualization of the netdev Contributor Funnel

**Following up on the [OnboardingFlow Journal](https://lauraarakakii.github.io/posts/manuscript/) (Mar 26, 2026)**

In the previous post I walked through the end-to-end Spark ETL pipeline that turned 20+ years of raw `lore.kernel.org` archives (list `org.kernel.vger.netdev`) into clean, analyzable artifacts:

- `funnel_summary.json` → global funnel metrics (2004–2024)
- `cohort_retention.csv` → year-by-year progression tables

Now I’m releasing the **interactive dashboard** that was built on top of those two files. This is the final visualization layer of the OnboardingFlow project — the place where all the numbers come alive.

You can explore the live dashboard here:  
**[→ netdev Contributor Funnel Dashboard](https://dashflow.streamlit.app/)** 

---

## Achievements 

From raw mailing-list archives to a fully interactive web dashboard, the OnboardingFlow project now delivers:

- A complete picture of how new contributors move through the netdev list
- Year-by-year tracking of every stage of the contributor journey
- Beautiful, publication-ready visualizations that anyone can explore

---

## Core data files

The two core data files (`funnel_summary.json` and `cohort_retention.csv`) contain everything the dashboard needs.

From `funnel_summary.json`:

- **New contributors**: **33,404**
- **Sent at least 1 patch**: **17,475** (52.31 %)
- **Received at least 1 review**: **3,679** (11.01 %)
- **Retained after 6 months**: **1,946** (5.83 %)
- **Retained after 12 months**: **1,527** (4.57 %)
- **Mean Time to First Patch (MTTFP)**: **58.3 days**
- **Patch → Review conversion**: **20.65 %**

The `cohort_retention.csv` breaks down the exact same metrics for every single entry year, making it possible to see how the funnel evolved over two decades.

---

## Dashboard Structure

The app (`app.py`) is a fully custom dark-themed Streamlit application with four clean sections:

### 1. 📊 **Visão Geral** (Overview)
- KPI metric cards with gradient accents
- Consolidated funnel chart (Plotly Funnel)
- Evolution of conversion rates over 20 years (line chart)
- Key insights panel

### 2. 🔬 **Análise de Coorte** (Cohort Analysis)
- Stacked bar chart of cohort sizes
- Retention heatmap (perfect for spotting year-by-year trends)
- Waterfall + conversion-rate breakdown for any selected year
- Scatter plot relating cohort size to long-term retention

### 3. 🚰 **Funil de Contribuição** (Contribution Funnel)
- Full Sankey diagram showing exact flow and drop-offs
- Global funnel from the JSON file
- Stacked area chart of volume over time

### 4. 📖 **Glossário**
- Complete terminology reference
- Methodological notes about how “new contributor”, “patch”, “review”, and “retention” were defined

Every page respects a year-range filter and includes export-friendly Plotly charts.

---

## Technical Highlights

- **Framework**: Streamlit + Plotly (fully interactive)
- **Theme**: Hand-crafted dark GitHub-style CSS with JetBrains Mono & Sora fonts
- **Caching**: `@st.cache_data` for instant loading of the two small files
- **No backend required** — everything runs client-side from the two provided artifacts
- **Reproducible**: just drop the CSV + JSON in the same folder as `app.py` and run `streamlit run app.py`

The complete source code is available in the repository:  
[`app.py`](https://github.com/lauraarakakii/onboardingflow/blob/main/app.py) (38 kB of clean, commented code).

---

## Main Findings

1. **Huge front-end barrier** — only ~52 % of people who write their first email ever submit a patch.
2. **Review is the real filter** — of those who patch, only **20.65 %** receive formal review feedback.
3. **Long-term retention is low** — fewer than **5 %** of new contributors are still active after one year.
4. **Lurking phase is real** — it takes on average **58 days** between first message and first patch.
5. **Stable over two decades** — retention rates have been remarkably consistent (and low) across 20 years, pointing to structural patterns rather than temporary issues.

All of these numbers are now beautifully visualized and fully explorable in the dashboard.

---

## What’s Next?

- Deploy the dashboard publicly (GitHub Pages + Streamlit Community Cloud)
- Add export buttons (PNG + CSV) for easy sharing of charts
- Possibly add a “Compare two periods” mode
- Open the repo for community contributions (new mailing lists, more metrics)

The dashboard closes the loop of the OnboardingFlow project: from raw mailing-list archives → Spark ETL → clean artifacts → interactive storytelling.

If you are a Linux kernel maintainer, OSS researcher, or just curious about how newcomers experience the netdev list, I’d love your feedback on the dashboard.

---

**Repository**: [github.com/lauraarakakii/onboardingflow](https://github.com/lauraarakakii/onboarding-flow)  
**Previous post**: [OnboardingFlow Journal → ETL Pipeline](https://lauraarakakii.github.io/posts/manuscript/)  
**Live Dashboard**: [Dashboard Netdev](https://dashflow.streamlit.app/)