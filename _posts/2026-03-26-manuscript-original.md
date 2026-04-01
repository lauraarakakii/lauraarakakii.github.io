---
title: OnboardingFlow - Journal
description: Journal of an article idea based on the dataset LKML5Ws - OnboardingFlow
date: 2050-03-26 15:25:00 +/-TTTT
categories: [Software Visualization]
tags: [dataset, etl, spark, LKML5Ws]
author: <author_id> 
---

## Introduction

This post is my journal of an article idea based on the dataset LKML5Ws - OnboardingFlow. The main idea is to understand the funnel of a new contributor from a specific mailing list.

The first analisys as made in spark, I ran the code in Google Colab, select just one mailing list in the first moment I consider the `netdev` list, because it has extremely high patch/ review volume, strong maintainer culture, besides that the Newcomers are clearly visible, threads stay mostly inside the list and the data of first-patch → first-review → retention over two decades.

To make a first analysis I decided to build a ETL project and a dashboard for better visualization.

## First Analysis

### Data Loading 

To begin the project I decided to follow with just one mailing list at the first moment. I consider the `netdev` list, in general there are a continuous flow of new developers sending patches and different of smaller subsystems the `netdev` list has speciffic rules about the prefix `[PATCH net]` - for bug fix - or `[PATCH net-next]` - for new features - that helps the proccess of extracting, besides that those rules also helps to answer a few questions that this article wants to achieve, like, "Was the patch acepted?", "How many interactions has to be done?", "The new developer gave up after the first negative?"

#### Extract

To begin the extract proccess I chose the list name as said, filtered a period of 20 years and retetion months for checkpoints. I also filtered the subject of each mail with regex, i used `(?i)\[patch` and the review body `(?i)(Reviewed-by|Acked-by)\s*:`.

I saved the DataFrame and add a column named 'list' with the specific list that I'm currently using

The dataframe schema looked like:

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

#### Normalize Columns

I normalize a few columns just to keep only relevant columns for this analysis, save execution time and memory usage. After that I filter for all my data is betweem the 20 years analysis and I also cached my DataFrame to store memory after first compupation and avoid recomputing transformations.

I also created a new column `thread_id` that uses `references` array lists ancestors oldest→newest; element [0] is the root of the thread. Fall back to `in-reply-to` when references is absent/empty, and finally to the message's own id for root messages.

After the normalization my schema looked like:

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

and I could catch the data of total messages in analysis window, number of distinct threads, thread-root messages and avg messages per thread

### Data Transformation

#### Stage 1 - New Contributor Detection

A contributor is **new** on the date of their very first message to this list. Each unique `contributor_hash` = one person.

Transform **event-level data** into **user-level data**

Input:

- Multiple rows per contributor (each message)

Output:

- One row per contributor with:
- First activity date 
- Total activity volume (with this information we can find top contributors)
- Cohort info (year/month)

To find only the first patch I group the `contributor_hash` and I made a couple of aggregations: 

- Find the `first_message_date` filtering the min from `date` column 
- Create the metric `total_messages` - Count all rows, regardless of nulls - total activity volume per contributor

I also created two other columns to help with **time-based aggregation** for cohort analysis by year, trend analysis and retention slicing:

- `entry_year` - the year of the first message date
- `entry_month` - the date down to the first day of the month - i can not extract only the month number because i could loose year context depending of the analysis, so I make a time bucketing (monthly cohorting)

From this analysis i can find the number of unique contributors, the top contributors and the firsts contributors

#### Stage 2 - First patch detection

A message is a **patch** when its subject matches `[PATCH` (case-insensitive). We record the date of each contributor's first such message.

I created a new DataFrame for the patch filtering, so I filtered the `subject` column with `(?i)\[patch`, grouped by the `contributor_hash` and consider the `first_message_date`

With this df i could find the contributors who submitted >= 1 patch and the first patch of each contributor

#### Stage 3 - First review tag detection

A contributor has been **reviewed** when any message in the thread contains `Reviewed-by:` or `Acked-by:` and the recipient is that contributor.

I created a new DataFrame for reviewed messages, so I filtered the column `body` with `(?i)(Reviewed-by|Acked-by)\s*:` (regex code) and select only the columns `message_id`, `date`, `contributor_hash`.

For this stage I used 2 approachs, as the original dataset doesn't contains the column `thread_id` I didn't consider it for my fisrt cenarium. 

So I will call it **'Approach A'** I flagged any contributor who sent a reviewed msg and created a new df grouping the contributers who had review emails and I got the min date so i would know that was the date of the first review.

However I understand that 1 patch send can have multiples reviews, so I decided to create the column `thread_id` in the normalization step, where we understand how many reviews 1 patch had, how long is that thread, so in this **'Approach B'** I created a new df that join reviewed messages back to the patch author via thread linkage. I grouped by contributor_hash and consider the minimum value for date, finding the first review date

In both ways I can find the number of contributors who have more than 1 review tag

#### Stage 4 - Retention analysis

A contributor is **retained** at checkpoint *k* months if they posted at least one message between 

```
first_message_date + k months
and
first_message_date + k months + 30 days
```

Basically I want to answer the question *“Did this contributor come back after X months?”*

To operationalize this, I implemented a function `compute_retention` that evaluates contributor activity within a defined retention window.

The function takes as input:

- the full message dataset (`df_all`)
- the contributor-level dataset with first activity (`first_message`)
- a retention checkpoint in months (`k`)

For each contributor, it computes a time window relative to their first message date, and filters all messages that fall within this interval.

Contributors with at least one message in the window are considered retained.

The function returns a DataFrame containing:

- contributor_hash
- retained_{k}m (boolean flag set to True)

This implementation follows a **filter-based retention model**, where only retained contributors are explicitly returned, and non-retained contributors are implicitly excluded.

### Data loading

Now that we completed the funnel and compute the three key metrics, we have our filtered DataFrame with all the information that we need

Funnel table schema:

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

#### Metric 1 - Mean time to first patch

We can 