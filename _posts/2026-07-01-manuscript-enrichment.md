---
title: "OnboardingFlow — Enriching the ETL: RFCs, Revision Cycles & Mentorship Signals"
description: How I improved my first project analysis
date: 2026-07-01 10:00:00 -0300
categories: [Software Visualization]
tags: [linux-kernel, linux-pci, onboarding, contributor-funnel, spark, etl, data-engineering]
---

**Following up on the [OnboardingFlow Journal](https://lauraarakakii.github.io/posts/manuscript/) (Mar 26, 2026)**

When I first built OnboardingFlow, the raw dataset gave me just enough to answer the core question — does a new contributor patch, get reviewed, and stick around? Since then, the underlying LKML5Ws extraction gained several new structured columns, and I switched the case study from `netdev` to a new list: **`org.kernel.vger.linux-pci`**. This post documents how I redesigned the ETL to take advantage of that richer schema without breaking the original funnel.

## Why Re-Enrich Instead of Rebuild

The original three metrics — Mean Time to First Patch, Review Conversion, and Cohort Retention — are the backbone of every number I've published so far. Rewriting that logic from scratch risked breaking comparability with prior results. So instead of a rewrite, I treated this as an **additive enrichment pass**: every new stage is computed independently and joined onto the existing funnel table, and every new column is used only when it's actually present in the dataset. Older dataset snapshots without these columns still run the notebook end-to-end, just skipping the stages that need them.

## The New Columns

| Column | What it unlocks |
|---|---|
| `has_patch_tag` / `has_rfc_tag` | Replaces subject-line regex with a precomputed, exact flag |
| `patch_version` + `untagged_subject` | Groups `v1 → v2 → v3` revisions of the same patchset together |
| `trailers` (struct: `attribution`, `identification`) | Structured `Reviewed-by` / `Acked-by` / `Tested-by` / `Signed-off-by` credit lines |
| `cc` | Who else was copied on a contributor's first patch |
| `code` | Size of the code changes attached to a message |

## New Stage: RFC → Patch

Some contributors test the waters with an `[RFC]` message before committing to a full patch submission. Using `has_rfc_tag`, I now track who does this and whether it converts into an actual patch (`first_patch_date >= first_rfc_date`). This required a new join, isolated in its own cell, so it can't interfere with the original patch-detection logic.

## New Stage: Revision Cycles

Patches are rarely accepted on the first try. By grouping messages on `contributor_hash` + `untagged_subject` (the subject stripped of its `[PATCH vN]` prefix, so every version of the same patch shares a key), and counting distinct `patch_version` values, I can measure how many revisions a patchset needs before it stops changing, and how many days that takes.

## New Stage: Mentorship / Visibility Signal

This one came from a hunch: if a newcomer's very first patch already has other people in `cc`, someone — a maintainer, a subsystem list, a reviewer — is probably already paying attention. I bucket contributors into `high_visibility` / `low_visibility` based on the Cc count on their first patch, and compare downstream outcomes between the two groups.

## New Stage: Patch Complexity

Using the `code` column, I sized each contributor's first patch and bucketed it into `small` / `medium` / `large`. On the `linux-pci` list this round, nearly every first patch fell into the `small` bucket — which tells me the size thresholds (or the way `code` captures a patch's footprint) need calibration before this signal is genuinely useful. I'm leaving the stage in the pipeline, flagged as a known limitation, rather than shipping a bucket boundary I haven't validated.

## A Data-Quality Note on Trailers

While validating this round of results, I found that my `trailers` struct fields were mapped backwards — I was comparing `identification` (the *person* being credited) against tag names like `"Reviewed-by"`, when the tag name actually lives in `attribution`. That silently zeroed out review detection for this run. I've fixed the mapping in the pipeline, but rather than re-run everything under deadline pressure, I decided to **exclude review-conversion numbers from this round's results** and revisit them properly in a follow-up post once I've re-validated the fix against the original netdev numbers.

One unplanned upside: the mislabeled column, read correctly, turned out to be a ranked list of the **people most frequently credited across all trailer types** on the list — which became one of the more interesting findings in the results post.

## Staying Backward-Compatible

The two original export files — `funnel_summary.json` and `cohort_retention.csv` — keep their exact original schema; the new metrics only *add* top-level keys and ship as separate files (`rfc_funnel.json`, `revision_cycles.csv`, `mentorship_signals.csv`, `patch_complexity.csv`, `trailer_breakdown.csv`). That means the dashboard's original three pages keep working untouched, and a new **🧬 Sinais Avançados** page only appears when at least one of the new files is present.

---

The full, real numbers from this pass — RFC conversion, revision cycles, mentorship signal, and the top-credited contributors on `linux-pci` — are in the results post: **[OnboardingFlow v2 — Enriched Signals for the linux-pci Contributor Funnel](https://lauraarakakii.github.io/posts/manuscript-results-v2/)**.

**Repository**: [github.com/lauraarakakii/onboarding-flow](https://github.com/lauraarakakii/onboarding-flow)
**Previous post**: [OnboardingFlow Journal → ETL Pipeline](https://lauraarakakii.github.io/posts/manuscript/)