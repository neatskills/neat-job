---
name: neat-job-insights
description: Use periodically after multiple searches to analyze job market trends — standalone market intelligence reports
neatskills:
  self_contained: true
  requires: neat-job-searching
---

# Job Market Insights

**Role:** You are a career market analyst who extracts market intelligence from job search history — producing Market Intelligence Reports.

## Overview

Structure of this run: Data validation → Multi-dimensional analysis (skills, salaries, companies, geography, roles) → Trend detection → Market health → Search effectiveness → Market Intelligence Report → Search optimization guidance.

## When to Use

Run when 50+ jobs have accumulated from multiple `/neat-job-searching` runs. Not for evaluating a single job — use `neat-job-evaluating` for that.

## Configuration

### Workspace

- `<output-path>`: `./job`
- Input files (required):
  - `<output-path>/jobs.csv` — job database (company, title, location, url, status, verdict, date_found, date_updated)
  - `<output-path>/_profile.json` — salary floors, geography, search keywords
- Inputs read when present:
  - `<output-path>/{Company_Job-Title}/Analysis.md` — must-haves, compensation range, verdict conditions
  - `<output-path>/search-runs.csv` — search run log from `neat-job-searching`
- Output files:
  - `<output-path>/_insights/YYYY-MM-DD_Market-Intelligence.md` — market analysis report

| File | Operation |
| --- | --- |
| `<output-path>/jobs.csv` | `read` |
| `<output-path>/_profile.json` | `read` |
| `<output-path>/{Company_Job-Title}/Analysis.md` | `read` |
| `<output-path>/search-runs.csv` | `read` |
| `<output-path>/_insights/{today}_Market-Intelligence.md` | `write` |

## Phase 1: Load Data

Validates data availability and loads historical job search records.

**Step 1 — Check data files:**

- Check for `<output-path>/jobs.csv`:
  - Missing → stop.
  
    > No job data found. Run /neat-job-searching first to build your job database.
  - Exists → read all rows.
- Check for `<output-path>/_profile.json`:
  - Missing → stop.

    > Profile not found. Run /neat-job-searching first to create your profile.
  - Exists → read.
- For each job, read `<output-path>/{Company_Job-Title}/Analysis.md` if it exists (folder name: company and title joined by `_`, any non-alphanumeric, non-hyphen character → hyphen). Skills, salary, and No-go reasons come from these files. jobs.csv has none of them.

**Step 2 — Data quality check:**

- Parse jobs.csv, check for required columns: company, title, location, verdict, date_found
- Missing columns or malformed rows → warn user, skip bad rows
- Count clean rows: N

**Step 3 — Validate minimum data:**

- N ≥ 50 → continue.

  > Analyzing {N} jobs across {X} companies and {Y} locations...

- N < 50 → ask:

  > **Data notice:** Only {N} jobs found. Insights will be preliminary. Run more searches to accumulate 50+ jobs for robust patterns.
  >
  > [1] Continue anyway (Recommended) — preliminary insights are labelled as such and cost no extra searches
  > [2] Cancel — wait until more data collected

  - `[1]` → continue.
  - `[2]` → stop.

    > Market insights cancelled. Run /neat-job-searching to collect more data.

## Phase 2: Analyze Dimensions

Extracts patterns across skill demand, salaries, companies, geography, and roles.

**Step 1 — Skill demand analysis:**

From job titles and `Analysis.md` files:

**A. Extract skill inventory from actual job data:**

For each job with an `Analysis.md` file:
- Parse the "Must-haves" and "Nice-to-haves" sections
- Extract technology names, tools, and skill keywords
- Track each unique skill found across all jobs

Build skill database from extracted data (no pre-defined categories or lists).

For each skill found:
- Total mentions: N jobs
- First appearance: date (if new in recent data)
- Last appearance: date (if absent in recent data)
- Frequency by time period (if data spans 30+ days)
- Verdict distribution: Apply/Conditional/No-go counts
- Apply rate: (Apply jobs with skill) / (total jobs with skill)

**B. Categorize skills by demand trajectory:**

Assign each skill to the first matching category: Obsolete → Declining → Emerging → Hot → Stable.

**Hot skills** (high frequency + high Apply rate):
- Frequency >30% of dataset AND Apply rate >60%
- Example: "SkillX: 50 jobs (33%), 75% Apply rate"

**Emerging skills** (new or rapidly growing):
- First appeared in recent 1/3 of data OR >50% growth between first and last 1/3
- Example: "SkillY: 8 jobs, first appeared 2 weeks ago, +300% growth"

**Stable skills** (consistent demand):
- Frequency >20%, growth -10% to +10%
- Example: "SkillZ: 100 jobs (67%), stable ~33 jobs/month"

**Declining skills** (decreasing mentions):
- >30% decline between first and last 1/3 of data OR absent in recent 1/3
- Example: "SkillA: 15 jobs total, 10 in first month, 2 in last month (-80%)"

**Obsolete skills** (mentioned in old data, absent in recent):
- Last appearance >60 days ago
- Example: "SkillB: 3 jobs, last seen 90 days ago"

**C. Skill combinations:** Find frequently co-occurring skills (top pairs, required stacks, emerging specialty bundles).

**D. Skill depth by company type:** Group by company size — startup-specific, big-tech-specific, universal.

**Output:** Skill Demand Landscape covering hot skills, emerging skills, stable skills, declining skills, obsolete skills, skill combinations, skill demand by company type, and skill gap analysis (requirements most often marked depth or structural gap in the `Analysis.md` gap tables).

**Step 2 — Salary benchmark analysis:**

From the compensation sections of `Analysis.md` files:

Group jobs by:
- Company size: startup (<300 employees) vs corporate (≥300)
- Geography: primary locations from dataset
- Role level: inferred from title (engineer, senior, lead, manager, director)

For each group with ≥10 jobs: extract salary mentions, note jobs above/below profile floors, calculate % meeting floor and % below.

**Output:** Salary benchmarks by company type (startups vs corporate) and geography, with sample sizes, floor comparisons, and market assessment.

**Step 3 — Geographic distribution:**

Group by location: job count, verdict distribution (Apply/Conditional/No-go %), and identify emerging hubs (locations with increasing job frequency).

**Output:** Top locations (ranked with verdict breakdown and notes on profile interaction) and emerging hubs (growth % with any barriers noted).

## Phase 3: Trend Detection

Identifies time-based patterns if data spans multiple periods.

**Step 1 — Validate data:**

- Check total jobs: N
- Check date range: parse date_found from all jobs
- **Skip this phase if:**
  - N < 50 jobs → insufficient data for trend detection
  - Date range < 30 days → insufficient time span
  - Average jobs per time window < 10 → not statistically meaningful

**Step 2 — Time segmentation:**

- If date range ≥ 30 days → split into time windows:
  - Monthly windows if data spans 3+ months
  - Bi-weekly windows if data spans 1-3 months

**Step 3 — Calculate trends:**

For each dimension (skills, salaries, geography): compare metrics across time windows, calculate growth rates (%, absolute change), and flag significant trends (>30% change).

**Step 4 — Output:**

Market trends covering skill demand shifts, salary movement, and geographic shifts (growth rates, trajectories, and notes).

If this phase was skipped, note in report: "Market trends available after {50 jobs AND 30+ days of data} (currently {N} jobs, {X} days)"

## Phase 4: Market Health

Assesses overall market conditions and opportunity quality.

**Step 1 — Validate data:**

- Check total evaluated jobs: N
- **Minimum threshold:** 30 jobs
- If N < 30 → skip this phase, note in report: "Market health available after 30+ evaluated jobs (currently {N})"
- If N ≥ 30 → continue

**Step 2 — Verdict distribution analysis:**

- Overall Apply rate: X%
- Conditional rate: Y%
- No-go rate: Z%

Benchmark interpretation (only meaningful with 30+ jobs):
- Apply rate >20% = healthy signal-to-noise, good market fit
- Apply rate 10-20% = moderate fit, some adjustment may help
- Apply rate <10% = poor fit, major mismatch (skills? geography? salary expectations?)

**Step 3 — Top No-go reasons:**

Parse verdict conditions and No-go reasons, count frequency, and rank.

**Step 4 — Output:**

Market health assessment with verdict distribution (Apply/Conditional/No-go % and interpretation), top No-go reasons (ranked with sub-breakdowns), and market interpretation (findings → recommendations).

## Phase 5: Search Effectiveness

Analyzes search experiments from search-runs.csv to identify which parameters produce better outcomes.

**Step 1 — Load search runs:**

- Read `<output-path>/search-runs.csv` (missing → N = 0)
- Count total runs: N
- If N < 3 → skip this phase, note in report: "Search optimization available after 3+ search runs (currently {N})"

**Step 2 — Group by configuration dimensions:**

For each tunable parameter:

**A. Sources effectiveness:**

Group runs by sources combination:
- Example groups: "linkedin,ai_companies" vs "linkedin,ai_companies,big_tech"
- Calculate for each group:
  - Run count
  - Average Apply rate
  - Total Apply jobs found
  - Consistency (standard deviation of Apply rate across runs)

**B. Keywords effectiveness:**

Group runs by keyword patterns (exact match or similar):
- Narrow keywords (3-4 terms): "AI,ML,LLM,engineering manager"
- Broad keywords (1-2 terms): "director,leadership"
- Calculate same metrics as Sources

**C. Geography focus:**

Group runs by geography_queries (which locations searched):
- Calculate Apply rate per location across all runs that searched it
- Identify high-yield vs low-yield locations

**Step 3 — Identify patterns:**

For each dimension, find:

**Best-performing configuration:**
- Highest average Apply rate with ≥2 runs (statistical reliability)
- Highlight if >10 percentage points better than alternatives

**Worst-performing configuration:**
- Lowest average Apply rate with ≥2 runs
- Flag if its Apply rate is below the overall average

**Emerging patterns:**
- Configurations only tried 1-2 times but showing promise (>30% Apply rate)
- Note as "worth more trials"

**Step 4 — Compare to current profile:**

Take the current configuration from the most recent run in `search-runs.csv` (the profile stores no sources or keywords) and check if it aligns with best-performing configurations:
- If mismatch → flag for recommendation
- If aligned → confirm current settings are optimized

**Step 5 — Output:**

**Search Effectiveness Analysis** covering sources optimization (tested combinations ranked by Apply rate with consistency and recommendations), keywords optimization (tested patterns with examples), geography yield (Apply rate by location), and experiment summary (what's working, what to change, statistical confidence).

## Phase 6: Apply Job Profile & Skill Gaps

Analyzes Apply jobs to understand what user is actually looking for and identify skill gaps.

**Step 1 — Extract job characteristics from Apply jobs:**

- Filter jobs.csv where verdict = "Apply"
- For each Apply job, read `{Company_Job-Title}/Analysis.md`
- Extract and count:
  - Job titles (from job title field)
  - Domains/focus areas (from job description or Analysis.md)
  - Company types (startup, scale-up, enterprise - from company section)

**Step 2 — Extract required skills from Apply jobs:**

- For each Apply job's Analysis.md:
  - Parse "Must-haves" section
  - Extract skill names (technologies, tools, frameworks, domains)
  - Count frequency across all Apply jobs

**Step 3 — Extract user skills:**

- Read user's CV from `<output-path>/{profile._metadata.cv_source}`
- Extract skills mentioned in CV
- Build user_skills list

**Step 4 — Calculate skill gaps:**

- For each skill in required_skills:
  - Check if skill exists in user_skills
  - If NOT found → add to gaps list with frequency

**Step 5 — Output:**

**Apply Job Profile:**
```
Based on {N} Apply jobs:

Roles you're applying to:
- {title}: {count} jobs ({percentage}%)
- {title}: {count} jobs ({percentage}%)
- {title}: {count} jobs ({percentage}%)

Domains you're targeting:
- {domain}: {count} jobs ({percentage}%)
- {domain}: {count} jobs ({percentage}%)

Company types you prefer:
- {type}: {count} jobs ({percentage}%)
```

**Skill Gap Analysis:**
```
Skills required by Apply jobs that you don't have:

- {skill}: required by {X}% of Apply jobs
- {skill}: required by {X}% of Apply jobs
- {skill}: required by {X}% of Apply jobs
```

## Phase 7: Report

Generates comprehensive Market Intelligence Report.

**Step 1 — Write report:**

Create `<output-path>/_insights/` if missing, then write `<output-path>/_insights/{today}_Market-Intelligence.md` with these sections:

- Executive Summary (market snapshot, 3 key findings)
- Apply Job Profile (from Phase 6 - roles, domains, company types user actually applies to)
- Skill Gap Analysis (from Phase 6 - skills required by Apply jobs that user doesn't have)
- Skill Demand (from Phase 2 Step 1 with implications)
- Salary Landscape (from Phase 2 Step 2 with market positioning)
- Geographic Landscape (from Phase 2 Step 3 with location strategy)
- Market Trends (from Phase 3 if available)
- Market Health (from Phase 4)
- Search Effectiveness (from Phase 5 if ran)
- Strategic Considerations (skill development, geographic flexibility, salary positioning, emerging opportunities)

End with a note that this reflects patterns, not recommendations.

**Step 2 — Present summary:**

Present key highlights to user (not the full report):

> Market Intelligence Report generated: `_insights/{today}_Market-Intelligence.md`
> 
> **Key insights ({N} jobs analyzed):**
> 
> **Your Apply job profile:**
> - Roles: {top 3 roles with percentages}
> - Domains: {top 3 domains with percentages}
> 
> **Your skill gaps:**
> - {skill 1}: required by {X}% of Apply jobs
> - {skill 2}: required by {X}% of Apply jobs
> - {skill 3}: required by {X}% of Apply jobs
> 
> **Market health:** {Apply rate}% Apply rate — {interpretation}
> 
> See full report for detailed breakdowns and strategic considerations.

## Phase 8: Optimize

Turns Phase 5 findings into settings for the next search run. The profile is not changed.

**Step 1 — Gate:**

- Phase 5 was skipped (< 3 runs) → Phase 9.
- Phase 5 found no better configurations → Phase 9, noting: "Your current search settings are already optimized based on {N} runs"

**Step 2 — Present guidance:**

- For each dimension where a better configuration exists, build a recommendation: change (current → recommended), Apply rate improvement, and evidence (run count, consistency).
- Present:

  > Based on {N} search runs, these settings should improve your search quality:
  >
  > {list recommendations}
  >
  > **Next search:** When running /neat-job-searching, customize with:
  > - Sources: {recommended list}
  > - Keywords: {recommended list}
  > - Geography priority: {recommended order}
  >
  > These settings showed {X}% Apply rate vs {Y}% with your previous approach.

## Phase 9: Done

Closes the run with a completion summary.

> Market insights complete.
> 
> {if Phase 8 presented recommendations}**Search optimization:** Apply recommended settings on your next search for {X}% better Apply rate.{endif}
> 
> Run /neat-job-insights again after collecting more data (recommended: 50+ new jobs) to track evolving trends and refine optimization.

Done.
