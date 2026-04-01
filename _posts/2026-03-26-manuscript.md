---
title: OnboardingFlow - Journal
description: Journal of an project idea based on the dataset LKML5Ws
date: 2026-03-26 15:25:00 +/-TTTT
categories: [Software Visualization]
tags: [dataset, etl, spark, LKML5Ws]
author: <author_id> # for single entry
---

## Introduction

This post documents the development of an project idea - OnboardingFlow - based on the LKML5Ws dataset. The core objective is to analyze the onboarding funnel of new contributors within a specific Linux kernel mailing list, focusing on how contributors evolve from their first interaction to long-term engagement.

The initial analysis was conducted using Apache Spark, executed in a Google Colab environment. As a starting point, I narrowed the scope to a single mailing list: netdev. This list was strategically selected due to its high volume of patches and reviews, well-established maintainer culture, and strong signal for contributor interactions. Additionally, it provides clear visibility into newcomer activity, with most discussion threads remaining contained within the list itself. This makes it particularly suitable for tracking the full lifecycle of contributions—from first patch submission to first review acknowledgment and eventual retention—across a long historical timeframe.

To operationalize this analysis, I designed and implemented an end-to-end ETL pipeline to process and structure the raw mailing list data. On top of this data layer, I built a dashboard to enable exploratory analysis and facilitate visualization of key funnel stages, such as initial contribution, peer validation, and retention over time.

## Data Extraction 

For the initial phase of this project, I decided to focus exclusively on the netdev mailing list. It is an ideal starting point due to the steady influx of new developers and its highly structured submission rules. By requiring specific prefixes — `[PATCH net]` for bug fixes and `[PATCH net-next]` for new features—the list simplifies the data extraction process. These conventions also make it easier to answer the primary questions of this study:

- **Success Rate**: Was the patch ultimately accepted?
- **Iteration**: How many revisions (v2, v3, etc.) were required?
- **Persistence**: Do new developers tend to give up after an initial rejection?"

### Schema Design

To build a comprehensive picture of contributor behavior, I focused the extraction on a **20-year historical window (2004–2024)**. This extensive timeframe allows for a deep longitudinal analysis — essentially letting us watch how onboarding patterns have evolved over two decades of Linux development.

##### **Filtering for Signals**

To cut through the noise of a high-volume mailing list, I applied targeted filtering logic:

- **Identifying Patches**: I used a case-insensitive regex `(?i)\[patch` to catch submissions in the subject lines.
- **Capturing Feedback**: To track the "success signals" of a patch, I scanned message bodies for standard maintainer tags like **Reviewed-by:** and **Acked-by:** using the pattern: `(?i)(Reviewed-by|Acked-by)\s*:`

###### Future-Proofing the Pipeline

Even though I’m starting with a single list, I’ve designed the pipeline for scale. By introducing an explicit `list` column early on, I’ve ensured that future iterations can easily ingest data from multiple subsystems without losing track of data lineage.

The resulting raw dataset uses a rich, denormalized schema that captures everything from thread metadata to the specific code snippets and trailers:

```
=== Schema ===
root
 |-- from: string (nullable = true)
 |-- to: array (nullable = true)
 |    |-- element: string (containsNull = true)
 |-- cc: array (nullable = true)
 |    |-- element: string (containsNull = true)
 |-- subject: string (nullable = true)
 |-- date: timestamp_ntz (nullable = true)
 |-- message-id: string (nullable = true)
 |-- in-reply-to: string (nullable = true)
 |-- references: array (nullable = true)
 |    |-- element: string (containsNull = true)
 |-- x-mailing-list: string (nullable = true)
 |-- trailers: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- attribution: string (nullable = true)
 |    |    |-- identification: string (nullable = true)
 |-- code: array (nullable = true)
 |    |-- element: string (containsNull = true)
 |-- raw_body: string (nullable = true)
 |-- list: string (nullable = false)
```

### Refining the Data: Normalization & Threading

With the raw data in hand, the next step was to trim the noise. I normalized the dataset to retain only the essential columns, reducing memory overhead and speeding up subsequent execution. During this phase, I also cached the DataFrame to prevent redundant computations.

###### The Logic Behind `thread_id`

The most critical addition here was the creation of a unique `thread_id`. Since mailing list conversations can be fragmented, I implemented a fallback logic to reconstruct the discussion flow:

- **Primary**: Use the first element of the `references` array (the "root" ancestor).
- **Secondary**: If references are missing, fall back to the `in-reply-to` field.
- **Final**: If both are absent, the message is treated as a root, and its own `message_id` becomes the `thread_id`.

This normalization transformed our schema into a lean, query-ready format:

```
=== Schema ===
root
 |-- contributor_hash: string (nullable = true)
 |-- date: timestamp_ntz (nullable = true)
 |-- subject: string (nullable = true)
 |-- message_id: string (nullable = true)
 |-- in_reply_to: string (nullable = true)
 |-- thread_id: string (nullable = true)
 |-- body: string (nullable = true)
```

By the end of this stage, I was able to calculate high-level metrics for our 20-year window, such as the **total message count, distinct thread volume, and the average depth of discussions**.

## Data Transformation

### Stage 1: Detecting New Contributors

To understand onboarding, I had to shift my perspective from **events** (individual messages) to **users** (the people behind them). In this project, a contributor is defined as "new" on the exact date they send their very first message to the list.

###### Transforming the Data

I aggregated the event-level data to create a single, unique profile for each `contributor_hash`. By grouping the data, I was able to extract:

- **The Origin Point**: Using the `min(date)` to find their `first_message_date`.
- **Activity Volume**: A `total_messages` count to distinguish between "one-hit wonders" and power users.

###### Time-Bucketings for Cohort Analysis

To make the data useful for long-term trends, I introduced two specific time-based columns:

1. `entry_year`: For high-level annual trends.
2. `entry_month`: I used **time bucketing** (truncating the date to the first of the month) rather than just a month number. This preserves the year context, allowing for precise monthly cohort analysis and retention slicing.

This transformation allowed me to finally identify our core metrics: **the total number of unique contributors, our most active "top" contributors, and the specific cohorts that joined during key Linux milestones**.

### Stage 2: Identifying the "First Patch"

Not every message on a mailing list is a contribution. To isolate actual code submissions, I created a dedicated filtering layer to identify patches.

###### The Filtering Logic

I defined a message as a "patch" if the subject line contained the `[PATCH]` tag. Using a case-insensitive regex — `(?i)\[patch` — I filtered the normalized dataset to create a new, patch-specific DataFrame.

###### Key Aggregations:

- **The Contribution Milestone**: By grouping this new DataFrame by `contributor_hash`, I captured the `first_patch_date` for every user.
- **Contributor Segmentation**: This allowed me to clearly distinguish between "General Participants" and "Code Contributors" (those with ≥1 patch submission).

This stage is vital for our onboarding analysis, as it marks the exact moment a new developer attempts to move their first piece of code into the kernel.

### Stage 3: Detecting the First Review Tag

In the Linux Kernel world, a `Reviewed-by:` or `Acked-by:` tag is the gold standard of peer approval. To track these signals, I implemented a regex filter — `(?i)(Reviewed-by|Acked-by)\s*:` — across the message bodies to identify "review events."

To find the date of a contributor's first review, I experimented with two distinct strategies:

###### Approach A: The Direct Flag
Initially, I simply flagged any message containing a review tag and associated the earliest date with the sender. While this provided a quick snapshot of "review activity," it didn't account for the complex relationship between a patch and its subsequent discussion.

###### Approach B: Thread-Linked Validation (The Refined Strategy)
To gain deeper accuracy, I utilized the `thread_id` created during the normalization phase. By joining the review signals back to the original patch author via the thread linkage, I could precisely identify when a specific contributor's work received its first official nod.

By grouping the data by `contributor_hash` and taking the `min(date)`, I successfully pinpointed the "First Review" milestone. This allowed me to measure not just how many contributors received feedback, but how many managed to cross the high bar of receiving multiple review tags across their career.

### Stage 4: Measuring Retention

The ultimate question for any open-source project is: **Do contributors come back?** To answer this, I implemented a formal retention analysis to see if a developer was still active at specific milestones (e.g., 3 months, 6 months, or 1 year after their first post).

###### The Retention Logic
I defined a contributor as "retained" at a checkpoint of *k* **months** if they posted at least one message within a 30-day window following that milestone.

###### How the `compute_retention` Function Works:

To operationalize this, I built a dedicated function that compares the full message history against each user's "birth date" (their first message). The process follows three steps:

1. Define the Window: For each user, calculate the target interval:
```
[first_message_date + k months]
to
[first_message_date + k months + 30 days]
```
2. Filter Activity: Scan the entire dataset for any activity falling strictly within that 30-day slice.
3. Flag Retention: If a match is found, the user is flagged as `retained_{k}m = True`.

By using this **filter-based model**, I can efficiently generate a list of "survivors" for any given month, allowing us to see exactly when the "drop-off" points occur in the contributor lifecycle.

## Data Loading

We have successfully completed the transformation funnel. The result is a unified DataFrame that integrates our three key performance metrics: **Patch Submission, Peer Validation, and Long-term Retention**.

### The Funnel Table Schema:

The final table is optimized for longitudinal analysis, featuring calculated fields that resolve the time-gap between activity milestones:

```
root
 |-- contributor_hash: string (nullable = true)
 |-- first_message_date: timestamp_ntz (nullable = true)
 |-- total_messages: long (nullable = false)
 |-- entry_year: integer (nullable = true)
 |-- entry_month: date (nullable = true)
 |-- first_patch_date: timestamp_ntz (nullable = true)
 |-- first_review_date: timestamp_ntz (nullable = true)
 |-- retained_6m: boolean (nullable = false)
 |-- retained_12m: boolean (nullable = false)
 |-- reached_patch: boolean (nullable = false)
 |-- reached_review: boolean (nullable = false)
 |-- days_to_first_patch: integer (nullable = true)
```

With this dataset, we are no longer looking at raw logs; we are looking at **contributor lifecycles**. We can now pivot by `entry_year` to see if the netdev onboarding process has become faster or more difficult over the last two decades.

---

### Metric Computation

#### Mean Time to First Patch (MTTFP)

To quantify onboarding efficiency, I computed the **Mean Time to First Patch (MTTFP)**. This metric captures the average time it takes for a new contributor to transition from their first interaction to submitting their first patch.

Only contributors who successfully reached the patch stage were considered, and negative or invalid durations were excluded to ensure data quality.

For this metric, three statistical descriptors were computed:

- **Mean Time to First Patch** — captures the overall onboarding speed across the population
- **Median Time to First Patch** — represents the typical contributor experience, reducing the impact of outliers
- **Standard Deviation** — measures variability, indicating how consistent (or uneven) the onboarding process is

This combination allows for a more robust interpretation of onboarding dynamics, avoiding misleading conclusions that could arise from relying solely on averages.

##### Analytical Relevance

MTTFP functions as a **leading indicator of contributor friction** within the onboarding pipeline:

- Lower values suggest a more accessible and efficient contribution process
- Higher values indicate potential bottlenecks, such as unclear contribution guidelines, delayed feedback cycles, or social barriers within the review process

By segmenting this metric across `entry_year`, it becomes possible to assess how onboarding efficiency has evolved over time, revealing whether the project has become more inclusive or more restrictive for new contributors.

#### Review Conversion Rate

To evaluate progression within the onboarding funnel, I defined the **Review Conversion Rate** as the proportion of contributors who, after submitting their first patch, successfully received a formal review signal (e.g., *Reviewed-by* or *Acked-by*).

This metric focuses exclusively on contributors who reached the patch submission stage, ensuring that the analysis reflects **conversion efficiency between consecutive funnel steps**, rather than overall participation.

Formally, the metric is defined as:

- **Total Patch Contributors** — number of contributors who submitted at least one patch
- **Reviewed Contributors** — subset of those contributors who received a review signal
- **Review Conversion Rate (%)** — percentage of patch contributors who progressed to the review stage

##### Analytical Relevance

The Review Conversion Rate serves as a **core indicator of validation efficiency and community responsiveness**:

- Higher conversion rates suggest an active and engaged reviewer base, where contributions are acknowledged and processed
- Lower conversion rates may indicate bottlenecks in the review pipeline, lack of reviewer availability, or barriers in contribution quality expectations

Unlike time-based metrics, this measure captures **structural friction** within the contribution workflow — specifically, whether submitted work is being recognized and integrated into the collaborative process.

##### Interpretation Layer

This metric becomes significantly more powerful when analyzed alongside other funnel dimensions:

- In combination with **MTTFP**, it differentiates between *fast but ineffective onboarding* and *slower but successful integration*
- When segmented by `entry_year`, it reveals whether the project’s ability to **absorb and validate contributions** has improved or degraded over time

A declining conversion rate over time may signal **scalability issues in the review process**, even if contribution volume is increasing.

#### Cohort Retention by Entry Year

To analyze long-term contributor engagement, I constructed a **cohort-based retention model**, grouping contributors by their year of entry (`entry_year`) and tracking their progression across key stages of the onboarding funnel.

Each cohort represents a group of contributors who initiated their participation within the same calendar year, enabling a **longitudinal comparison of onboarding effectiveness and retention dynamics over time**.

For each cohort, the following aggregated metrics were computed:

- **Cohort Size** — total number of new contributors in a given entry year
- **Patch Stage (Conversion)** — number and percentage of contributors who submitted at least one patch
- **Review Stage (Conversion)** — number and percentage of contributors who received a formal review signal
- **Retention Metrics** — number and percentage of contributors who remained active after predefined time windows (e.g., 6 and 12 months)

All stage-level metrics were normalized as percentages relative to the cohort size, enabling direct comparison across years regardless of fluctuations in contributor volume.

##### Funnel Representation

This structure allows each cohort to be interpreted as a multi-stage conversion funnel, where contributors progress through the following lifecycle:

1. Entry (first message)
2. Patch submission
3. Peer validation (review)
4. Long-term retention

By aligning all cohorts under this standardized funnel, it becomes possible to evaluate not only absolute growth, but also **conversion efficiency at each stage over time**.

##### Analytical Relevance

Cohort-based retention provides a **macro-level view of ecosystem health**, capturing both onboarding effectiveness and long-term sustainability:

- Increasing cohort sizes indicate growing interest or visibility
- Stable or improving conversion rates suggest consistent onboarding quality
- Declining retention rates may signal deeper structural issues, such as lack of engagement, insufficient feedback loops, or contributor burnout

Unlike isolated metrics, this approach reveals **where in the lifecycle contributors are being lost**, enabling targeted interventions.

##### Temporal Insights

By analyzing cohorts across multiple years, this metric enables the identification of structural trends:

- **Improved early-stage conversion (patch/review)** → better onboarding processes or tooling
- **Drop in retention despite stable conversion** → potential issues in long-term engagement or community integration
- **Simultaneous decline across all stages** → systemic barriers affecting the entire onboarding pipeline

This longitudinal perspective transforms the dataset into a **time-aware performance model**, allowing us to assess whether the netdev community has become more efficient, more selective, or more constrained in integrating new contributors.

## Data Export and Visualization Layer  
To enable downstream analysis and visualization, the computed metrics were consolidated and exported into structured artifacts. This step transforms the analytical outputs into **portable data products**, decoupling computation from presentation.

Two main outputs were generated: a **global funnel summary** and a **cohort retention table**.

---

### Global Funnel Summary

A consolidated representation of the onboarding funnel was created to capture the overall progression of contributors across key lifecycle stages.

This summary includes:

- **Total number of new contributors** (entry point)
- **Number and percentage of contributors who submitted a first patch**
- **Number and percentage of contributors who received a first review**
- **Number and percentage of contributors retained after 6 and 12 months**

In addition to stage-level counts, the summary also embeds key performance indicators:

- **Mean Time to First Patch (MTTFP)** — measuring onboarding speed
- **Review Conversion Rate** — measuring validation efficiency

The resulting structure is designed to support **funnel visualizations**, such as Sankey diagrams or stage-based bar charts, where both absolute volume and conversion rates are required.

#### Analytical Role

This artifact serves as a **high-level snapshot of the contributor lifecycle**, enabling quick assessment of:

Drop-off between stages
Overall onboarding efficiency
Balance between acquisition and retention

By centralizing these metrics into a single structure, it becomes possible to drive **executive-level insights** without requiring direct interaction with raw data.

---

### Cohort Retention Table

In parallel, a detailed cohort-level dataset was exported to support temporal analysis.

This table contains, for each `entry_year`:

- Cohort size
- Conversion rates for patch and review stages
- Retention rates across predefined time windows (e.g., 6 and 12 months)

This structure is optimized for time-series and **cohort-based visualizations**, such as:

- Retention curves
- Cohort heatmaps
- Temporal trend analysis

#### Analytical Role

While the global funnel summary provides a static overview, the cohort table enables **dynamic exploration** of longitudinal patterns, allowing analysts to:

- Compare onboarding performance across years
- Identify structural changes in contributor behavior
- Detect early signals of ecosystem degradation or improvement