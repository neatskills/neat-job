---
name: neat-job-searching
description: Use when searching for new job opportunities or setting up a job-search profile — ranked shortlists of evaluated roles
neatskills:
  self_contained: true
---

# Job Searching

**Role:** You are a job search coordinator who runs multi-source job searches and parallel evaluations — producing ranked shortlists.

## Overview

Structure of this run: Profile setup (first run) or validation → Multi-source search → Basic filtering → Parallel evaluation (max 5 agents at a time) → Aggregation → Ranked shortlist → Optional bulk application.

## When to Use

Run when searching for new job opportunities on LinkedIn. Not for evaluating a single job — use `neat-job-evaluating` for that. Not for applying to already-evaluated jobs — use `neat-job-applying` for that.

## Configuration

### Workspace

- `<output-path>`: `./job`
- Profile file (created on first run if missing):
  - `<output-path>/_profile.json` — used for geography-based search queries, exclusion filters, salary floors
- Tracking files (created if missing):
  - `<output-path>/jobs.csv` — status of every job (columns: company, title, location, url, status, verdict, date_found, date_updated, run_id)
  - `<output-path>/history.csv` — job IDs for deduplication (columns: job_id, url, date_seen)
  - `<output-path>/search-runs.csv` — search experiment tracking (columns: run_id, date, keywords_role_titles, keywords_domains, locations, jobs_found, recruiters_filtered, jobs_evaluated, apply_count, conditional_count, nogo_count, apply_rate)
- Search output (dated files):
  - `<output-path>/_search/YYYY-MM-DD_Shortlist.md` — ranked shortlist with verdicts
  - `<output-path>/_search/YYYY-MM-DD_Search-Log.md` — full search results before filtering

| File | Operation |
| --- | --- |
| `<output-path>/_profile.json` | `read` / `write` |
| `<output-path>/{base CV}` | `read` / `write` |
| `<output-path>/jobs.csv` | `read` / `write` / `append` / `patch status` |
| `<output-path>/history.csv` | `read` / `write` / `append` |
| `<output-path>/search-runs.csv` | `read` / `write` / `append` |
| `<output-path>/_search/{today}_Search-Log.md` | `write` |
| `<output-path>/_search/{today}_Shortlist.md` | `write` |

### Definitions

- Source: LinkedIn aggregate job listing pages only.
- Batch run: for a list of jobs, in batches of 5 (ceil(count / 5) batches):
  - Launch one `general-purpose` Agent per job in a single message, with the phase's name, description, and prompt.
  - Wait for every agent in the batch to finish, then parse each agent's final message as JSON.
  - Output the phase's progress line.

## Phase 1: Profile & Setup

Creates profile on first run, validates on subsequent runs, loads tracking files, confirms search scope.

**Step 1 — Check profile:**

- Check for `<output-path>/_profile.json`:
  - Missing → Step 2 (create profile).
  - Exists → read and validate it has required fields (`salary`, `geography`, `search_keywords`, `exclusions`). 
    - Complete → load silently, Step 6 (tracking).
    - Missing fields → Step 4 (gap questions for the missing fields only), then Step 5 merges the answers into the existing profile.

**Step 2 — Locate base CV:**

- Scan `<output-path>` for files with stem matching `profile`, `cv`, or `resume` (case-insensitive, any extension):
  - One match → read, proceed to Step 3.
  - Multiple → list with numbered picker, mark most recent `(Recommended) — newest`, read selected, proceed to Step 3.
  - None → Step 6 (ask all questions).

**Step 3 — Parse CV:**

Derive from CV content where present (don't ask, note these for profile creation):

- **Location** (city/country) → seed `geography.primary`
- **Experience level** (years, seniority from dates and titles) → note for career path suggestions
- **Skills/domain** keywords (AI, ML, consulting, engineering, etc.) → note for career path suggestions
- **Current/past employers** (especially Slalom, Amazon, consulting firms) → note mentions

**Step 4 — Gap questions:**

No base CV from Step 2 → first ask:

- Current location?
- Years of experience?
- Primary skills/domain (AI/ML, consulting, backend engineering, data science, etc.)?
- Current role level (IC, lead, manager, director, VP)?

Ask only what CV didn't reveal in Step 3. Present CV-derived values as defaults where applicable.

Must-ask (CV rarely has these):

1. **Salary floor for startups** (base only, before equity/bonus)? Ask for amount and currency.
2. **Salary floor for corporate/big-tech**? Ask for amount in the same currency as Q1.
3. **Top-tier startup exception?** Waive salary floor for top-tier startups? [Yes/No]
4. **Geography preferences** beyond {CV location}? Ask:
   - Primary cities (where you'd move immediately)
   - Secondary countries/cities (open to)
   - Requires strong match (only for exceptional roles)
   - Excluded (work permit issues, personal reasons)
5. **Search keywords?** Ask:
   - Role titles you're looking for? (e.g., "engineering manager, director, head of engineering")
   - Domain/industry focus? (e.g., "AI, ML, machine learning, fintech")
6. **Excluded companies?** Any companies to avoid (bad experiences, competitors, culture mismatch)?
7. **Excluded role types?** In your own words. Suggested: {role types the CV and career paths rule out, e.g. IC roles when the CV shows leadership} — from your CV. Add roles you can do but don't want.

**Step 5 — Write profile:**

Write `<output-path>/_profile.json`:

```jsonc
{
  "salary": {
    "startup_base": 0,                    // Step 4 Q1 — user-provided amount
    "corporate_base": 0,                  // Step 4 Q2 — user-provided amount
    "currency": "",                       // Step 4 Q1 — user-provided currency (EUR, USD, GBP, SGD, etc.)
    "top_tier_startup_exception": true    // Step 4 Q3
  },
  "geography": {                          // Step 4 Q4, seeded from CV location
    "primary": [],
    "secondary": [],
    "requires_strong_match": [],
    "excluded": []
  },
  "search_keywords": {
    "role_titles": [],                    // Step 4 Q5 — user-provided, e.g. ["engineering manager", "director"]
    "domains": []                         // Step 4 Q5 — user-provided, e.g. ["AI", "ML", "fintech"]
  },
  "exclusions": {
    "companies": [],                      // Step 4 Q6
    "role_types": [],                     // Step 4 Q7: free-text phrases, e.g. "quota-carrying sales"
    "recruiters": []                      // Auto-detected + user can add
  },
  "_metadata": {
    "created": "",                        // ISO timestamp
    "cv_source": null,                    // base CV filename, or null
    "cv_refined": false
  }
}
```

Compare your output against this schema — confirm all required keys are present. Present confirmation:

> Profile created: `_profile.json`
> 
> Proceeding to search setup...

**Step 6 — Load or create tracking:**

- Check for `<output-path>/jobs.csv`:
  - Exists → read, note row count (for reporting: "X jobs tracked"), find most recent `date_found` for incremental search.
  - Missing → create with headers: `company,title,location,url,status,verdict,date_found,date_updated,run_id`.

- Check for `<output-path>/history.csv`:
  - Exists → read, load all `job_id` and `url` into seen set.
  - Missing → create with headers: `job_id,url,date_seen`.

- Check for `<output-path>/search-runs.csv`:
  - Exists → read, note run count (for reporting: "Search run #{N+1}"), generate next `run_id`.
  - Missing → create with headers: `run_id,date,keywords_role_titles,keywords_domains,locations,jobs_found,recruiters_filtered,jobs_evaluated,apply_count,conditional_count,nogo_count,apply_rate`. Set first `run_id` = `sr_001`.

- Set incremental search date:
  - If jobs.csv has rows → calculate days between (most recent `date_found` + 1 day) and today. If >7 days, use 7 days ago. Otherwise, use (most recent `date_found` + 1 day).
  - Otherwise → 7 days ago (first search default)

**Step 7 — Confirm search scope:**

Present search scope and ask user to customize:

> **Search run #{N+1}** (based on your profile):
> 
> **Search period:** Jobs posted since {incremental date from Step 6} ({X} days)
> 
> **Keywords:**
> - Role titles: {list profile.search_keywords.role_titles}
> - Domains: {list profile.search_keywords.domains}
> 
> **Geography:**
> - Locations: {list profile.geography.primary + secondary + requires_strong_match}
> 
> **Exclusions:**
> - Companies: {list profile.exclusions.companies}
> - Role types: {list profile.exclusions.role_types}
> - Locations: {list profile.geography.excluded}
> 
> [1] Search with these settings (Recommended) — incremental search
> [2] Customize keywords
> [3] Cancel

- `[1]` → proceed to Phase 2 with defaults.
- `[2]` → ask for custom keywords, proceed to Phase 2:

  > Custom role titles? (Leave blank to use: {profile.search_keywords.role_titles})
  > Custom domains? (Leave blank to use: {profile.search_keywords.domains})

  Record user selections for `keywords` field.
- `[3]` → stop.

  > Job searching cancelled.

## Phase 2: Search

Searches LinkedIn aggregate job listing pages and extracts job data.

Create `<output-path>/_search/` if missing. Set dated filenames: `{today}_Shortlist.md`, `{today}_Search-Log.md` where `{today}` is YYYY-MM-DD.

**Step 1 — Build LinkedIn queries:**

For each location in `profile.geography.primary` + `profile.geography.secondary` + `profile.geography.requires_strong_match`:

- Build search query from user-provided keywords:
  - role_titles = `profile.search_keywords.role_titles` (e.g., ["engineering manager", "director"])
  - domains = `profile.search_keywords.domains` (e.g., ["AI", "ML"])
  - Build query: `site:linkedin.com/jobs ("${role1}" OR "${role2}") (${domain1} OR ${domain2}) ${location}`
  - Example: `site:linkedin.com/jobs ("engineering manager" OR "director") (AI OR ML) London`

**Step 2 — Execute LinkedIn search:**

For each query from Step 1:

- Use WebSearch with the query
- Filter results to aggregate listing pages only:
  - Keep URLs matching pattern: `*/jobs/*-jobs-*` (e.g., `/jobs/machine-learning-engineering-manager-jobs-london`)
  - Skip URLs matching pattern: `*/jobs/view/*` (direct job postings - these expire quickly)
- Result: List of aggregate page URLs

**Step 3 — Fetch aggregate pages:**

For each aggregate page URL:

- Use WebFetch with prompt: "Extract: 1) Total jobs shown, 2) First 20 company names and job titles, 3) Identify any recruitment agencies in the list"
- Parse response to extract:
  - company (string)
  - title (string)
  - source = "linkedin"
- Add to jobs list

**Step 4 — Filter recruiters:**

For each job, check if `company` matches any in `profile.exclusions.recruiters` (case-insensitive, partial match):
- Match found → remove job
- No match → keep job

If `profile.exclusions.recruiters` is empty (first run), no filtering occurs. User will build this list by marking jobs as "No-go - recruiter" during evaluation.

**Step 5 — Search log:**

- Count total jobs found: N
- Count recruiters filtered: R
- Count jobs remaining: M = N - R
- Write `_search/{today}_Search-Log.md` with all M jobs (company, title, source)
- Report to user:

  > Found {N} jobs on LinkedIn
  > Filtered {R} recruiter posts
  > Remaining: {M} jobs from {X} companies
  > 
  > Deduplicating and filtering...

## Phase 3: Filter

Applies basic exclusions from profile and deduplicates against tracking files.

**Step 1 — Deduplicate:**

For each job in search log:
- Generate job_id from URL (normalize domain, extract job ID or hash full URL if no ID)
- Check against `history.csv`:
  - Found → drop job, note in filter log: "Seen before"
  - Not found → continue to Step 2

**Step 2 — Exclusions:**

For each remaining job:

- **Company exclusion:** If job.company matches any in `profile.exclusions.companies` (case-insensitive, partial match) → drop, note: "Excluded company"
- **Location exclusion:** If job.location matches any in `profile.geography.excluded` → drop, note: "Excluded location"
- **Role type exclusion:** If job.title describes any role type in `profile.exclusions.role_types` → drop, note: "Excluded role type: {type}"

**Step 3 — Filter report:**

- Count jobs after filtering: M (from N original)
- Report to user:

  > After filtering: {M} jobs remain (dropped {N-M}: {X} duplicates, {Y} excluded companies, {Z} excluded locations, {W} excluded role types).
  >
  > Launching parallel evaluations (max 5 at a time)...

## Phase 4: Evaluate

Launches parallel `neat-job-evaluating` agents with batching, collecting structured reports.

**Step 1 — Evaluate:** Batch run over the M filtered jobs with:

- name: "eval-{company}-{title-slug}"
- description: "Evaluate {company} {title}"
- prompt: "Run /neat-job-evaluating with job details (company, title, location, URL). You are running as a subagent for batch evaluation. Fetch the JD from URL, use existing profile, execute full evaluation, and choose 'Save only' when asked whether to apply. Do not invoke neat-job-applying. Reply with only this JSON: {\"verdict\": \"Apply | Conditional | No-go\", \"signals\": {\"fit\", \"company\", \"compensation\"}, \"conditions\": [], \"artifacts\": {\"jd_path\", \"analysis_path\"}, \"key_signals\": []}"
- Progress line:

  > Batch {i}/{total_batches} complete ({X} Apply, {Y} Conditional, {Z} No-go so far)...

## Phase 5: Aggregate

Collects structured evaluation results from Phase 4, ranks by verdict and signals, produces shortlist.

**Step 1 — Aggregate results:**

Combine all structured reports from Phase 4 subagents into ranked list:

**Ranking order:**
1. **Apply + all greens** (perfect matches)
2. **Apply + a yellow** (strong matches with minor concerns)
3. **Conditional, no red signals** (yellow fit/company/compensation)
4. **Conditional with a red signal** (red fit on nice-to-haves, red company, or red compensation)
5. **No-go** (excluded or red on must-haves)

**Step 2 — Write shortlist:**

Write `_search/{today}_Shortlist.md` with: search summary (keywords used, jobs found/evaluated, verdict counts), Apply section (table with company, title, location, signals, key signals, link), Conditional section (table with conditions), and No-go section (collapsible table with reasons).

**Step 3 — Update tracking:**

- Append all M evaluated jobs to `jobs.csv` with: company, title, location, url, status="evaluated", verdict, date_found={today}, date_updated={today}, run_id={from Phase 1 Step 6}
- Append all M jobs to `history.csv` with: job_id, url, date_seen={today}

**Step 4 — Present shortlist:**

- Present the full shortlist (or top 20 if >20 results)
- X = Apply count. X = 0 → skip the prompt, Phase 7 (Done).
- Otherwise report:

  > Shortlist saved: `_search/{today}_Shortlist.md`
  > 
  > {X} Apply verdicts ready for application. Would you like to:
  > 
  > [1] Review shortlist only — done for now
  > [2] Auto-apply to all "Apply" jobs — launch parallel application agents
  > [3] Pick specific jobs to apply — I'll ask which ones

  Mark `[2]` `(Recommended) — {X} Apply verdicts, within one review sitting` if X ≤ 10, else `[3]` `(Recommended) — {X} Apply verdicts is too many to auto-apply`.

- `[1]` → Phase 7 (Done).
- `[2]` → proceed to Phase 6 with all "Apply" jobs.
- `[3]` → present "Apply" list with picker, let user select subset → proceed to Phase 6 with selected jobs.

## Phase 6: Apply (Optional)

Launches parallel `neat-job-applying` agents for selected jobs, collecting structured reports.

**Step 1 — Apply:** Batch run over the P jobs selected in Phase 5 with:

- name: "apply-{company}-{title-slug}"
- description: "Apply to {company} {title}"
- prompt: "Run /neat-job-applying for {folder_path}. You are running as a subagent for batch application. Use existing profile and job folder. Generate CV and cover letter, save to folder. Skip user review (batch mode, auto-approve). Do not invoke other skills. Reply with only this JSON: {\"status\": \"completed | failed\", \"artifacts\": {\"cv_path\", \"cover_letter_path\"}, \"notes\": \"issues or special handling\"}"
- Progress line:

  > Application batch {i}/{total_batches} complete ({X} successful, {Y} failed)...

**Step 2 — Update tracking:**

- Update `jobs.csv` for applied jobs:
  - Status "completed" → status="applied", date_updated={today}
  - Status "failed" → status="application_failed", date_updated={today}

**Step 3 — Report completion:**

> Application materials ready for {P-Y} jobs ({Y} failed).
> 
> **Successful applications:**
> {list each with artifact paths}
> 
> {if Y > 0}**Failed applications:**
> {list each with notes}
> {endif}
> 
> **Next steps:**
> 1. Review CVs and cover letters in each job folder
> 2. Upload to application portals or email to recruiters
> 
> Job folders saved under `<output-path>/`

## Phase 7: Done

Records this search run for future optimization, then presents final summary.

**Step 1 — Record search run:**

Append a row to `<output-path>/search-runs.csv`:

- `run_id`: from Phase 1 Step 6
- `date`: {today}
- `keywords_role_titles`: comma-separated list (from profile.search_keywords.role_titles or custom)
- `keywords_domains`: comma-separated list (from profile.search_keywords.domains or custom)
- `locations`: comma-separated list of locations searched (from profile.geography)
- `jobs_found`: N (from Phase 2 Step 5, total before recruiter filter)
- `recruiters_filtered`: R (from Phase 2 Step 4)
- `jobs_evaluated`: M (from Phase 4, jobs after all filtering)
- `apply_count`: X (from Phase 5 aggregation)
- `conditional_count`: Y
- `nogo_count`: Z
- `apply_rate`: X/M as percentage

**Step 2 — Final summary:**

> Job searching complete.
> 
> **Summary:**
> - Jobs found: {N}
> - Jobs evaluated: {M}
> - Apply: {X}, Conditional: {Y}, No-go: {Z}
> - Applications prepared: {P} (if Phase 6 ran)
> 
> **Files saved:**
> - Shortlist: `_search/{today}_Shortlist.md`
> - Search log: `_search/{today}_Search-Log.md`
> - Tracking: `jobs.csv` ({total_rows} jobs tracked)
> 
> Run `/neat-job-searching` again to search for new jobs (deduplication is automatic).

Done.
