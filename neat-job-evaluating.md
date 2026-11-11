---
name: neat-job-evaluating
description: Use when evaluating a job opportunity before applying — structured Apply / No-go / Conditional assessment
neatskills:
  self_contained: true
---

# Job Evaluating

**Role:** You are a career coach who evaluates job opportunities against a candidate profile and delivers a clear Apply / No-go / Conditional recommendation — producing Analysis.md.

## Overview

Structure of this run: Profile validation → JD and candidate profile → Evaluate, then Company/Compensation research in parallel → Verdict → `Analysis.md`.

## When to Use

Run when evaluating whether to apply to a specific job opening. Not for tailoring the application itself — use `neat-job-applying` for that.

## Configuration

### Workspace

- `<output-path>`: `./job`
- Profile file (required, created by `neat-job-searching` on first run):
  - `<output-path>/_profile.json` — user criteria: salary floors, geography, career paths, exclusions, writing rules
- Output files, written in Phase 4 only if the user picks Yes or Save only. Folder derived in Phase 1:
  - `<output-path>/Company_Job-Title/JD.md` — verbatim, no annotations
  - `<output-path>/Company_Job-Title/Analysis.md`

| File | Operation |
| --- | --- |
| `<output-path>/_profile.json` | `read` |
| `<output-path>/{folder}/JD.md` | `write` |
| `<output-path>/{folder}/Analysis.md` | `write` |

### Definitions

- Top-tier startup: meets 3 of 5 criteria: (1) AI is core product, (2) top-tier investors, (3) real traction, (4) founders with track record, (5) early leadership seat.

## Phase 1: Load Context

Validates profile exists and loads JD for evaluation.

**Step 1 — Check profile:**

- Check for `<output-path>/_profile.json`:
  - Missing → stop.
  
    > Profile not found. Run /neat-job-searching first to create your profile.
  - Exists → read and validate it has required fields (`salary`, `geography`, `search_keywords`, `exclusions`).
    - Missing fields → stop.

      > Profile incomplete — missing: {fields}. Run /neat-job-searching to complete it.
    - Complete → load silently, continue.

**Step 2 — Collect JD and derive folder:**

- JD given as a LinkedIn URL → LinkedIn blocks WebFetch. Use the built-in browser to read the job listing page. Each evaluation can open its own browser tab.
- JD given as a non-LinkedIn URL → fetch it with WebFetch.
- JD not in context → ask for it now.
- Derive `Company_Job-Title/` from the JD → `{folder}`. Any non-alphanumeric, non-hyphen character → hyphen.
- Company name or job title missing from JD → ask.

## Phase 2: Evaluate

Checks disqualifiers and scores candidate fit against the JD.

**Step 1 — Disqualifiers:** Scan for: citizenship-only work auth, native-language requirement, mandatory on-site relocation, citizenship-tied clearance.

- None found → set eligibility signal to green, continue.
- Uncertain (e.g., vague work auth wording, language preference not native-level) → set eligibility signal to yellow, continue.
- Confirmed → state plainly and stop.

**Step 2 — JD breakdown:**

- Role level and seniority signals.
- Core responsibilities.
- Must-have vs. nice-to-have.
- Framing quality:
  - Boilerplate → lower hiring commitment.
  - Fresh/specific → active targeted search.

**Step 3 — Fit scoring:** Score candidate against each must-have and nice-to-have.

- Gap types:
  - No gap (candidate meets requirement) → green.
  - Depth gap (adjacent domain/transferable skills) → yellow.
  - Structural gap (categorical difference) → red.
- Years-of-experience:
  - <50% required or different domain → structural gap.
  - ≥50% or adjacent domain → depth gap.
- Overall fit signal = red if any red, else yellow if any yellow, else green.
- Output fit signal and gap table (type, requirement, candidate_status, gap_type, fit per requirement).

**Step 4 — Career Direction:** Note any direction shift — e.g. IC → management, client-facing → internal, generalist → specialist.

## Phase 3: Research

Gathers company stability and compensation data to inform the verdict.

**Step 1 — Pre-parallel setup:**

- If the role is remote and candidate location is not in the profile, ask for their location now.
- Run Steps 2 and 3 in parallel.

**Step 2 — Company:** Search for: funding/ownership stage, recent layoffs, Glassdoor/Blind sentiment scores (cite numbers), revenue/growth trajectory, sector-specific context (controversies, active business).

- Use any third-party data. Note paywalled platforms as "Score paywalled — [platform]".
- No data → yellow + "Insufficient data — speculative".
- Signal:
  - Red = active/recent layoffs, funding distress (down round, failed raise, insolvency risk), or a business-ending controversy.
  - Yellow = mixed or moderate concerns (e.g., weak sentiment scores alone, a single dated layoff round, unclear trajectory).
  - Green = stable or growing, and neither Red nor Yellow applies.
- Output: company signal (green / yellow / red) with reasoning.

**Step 3 — Compensation:** Search for location- and company-specific compensation, then compare against profile salary floors.

- Remote roles: benchmark against the candidate location from profile `geography.primary`. No location → interpolate.
- On-site/hybrid roles: benchmark the job's listed location directly.
- Cite sources and flag confidence: "direct, location-specific" vs. "interpolated — low confidence".
- **Profile floor check:**
  - Company size <300 employees → compare against `profile.salary.startup_base`.
  - Company size ≥300 employees, or big tech/AI lab → compare against `profile.salary.corporate_base`.
  - Top-tier startup check: if company <300 employees AND is a top-tier startup AND `profile.salary.top_tier_startup_exception` is true → waive floor, signal green + note exception applied.
  - Convert compensation to `profile.salary.currency` when it differs.
- Output compensation signal:
  - Green = at or above profile floor (or top-tier exception applied).
  - Yellow = within 20% below profile floor, low confidence, or no data ("No compensation data — cannot verify against floor").
  - Red = >20% below profile floor with no top-tier exception.

## Phase 4: Verdict

Combines all signals into a final recommendation and routes to output.

**Step 1 — Exclusions check:**

- Any No-go here is final → skip Step 2 and draft with verdict No-go.
- Company name matches any in `profile.exclusions.companies` → No-go. State: "Excluded company: {company} — per your profile preferences."
- JD title or core responsibilities describe any role type in `profile.exclusions.role_types` → No-go. State: "Excluded role type: {type} — per your profile preferences."
- **Geography check:**
  - Job location in `profile.geography.excluded` → No-go. State: "Excluded location: {location} — per your profile (work permit/personal reasons)."
  - Job location not in `primary`, `secondary`, or `requires_strong_match` → ask. No → No-go.

    > Location {location} is not in your profile preferences. Proceed? Suggested: No — outside stated geography preferences

**Step 2 — Verdict:** Combine signals into **Apply**, **No-go**, or **Conditional** (state conditions explicitly). Signal priority — first match wins:

1. Structural gap on a must-have → No-go. Notes:
   - Confirmed disqualifiers already stopped in Phase 2.
   - A yellow eligibility signal is not a confirmed disqualifier → rule 6.
   - A nice-to-have gap feeds the fit signal, not this rule.
2. Red fit signal not caused by a must-have structural gap (i.e., driven by structural gaps on nice-to-haves) → Conditional (name the gaps).
3. Red company → Conditional (name the risk).
4. Red compensation (>20% below profile floor, no top-tier exception) → Conditional (name the gap and the profile floor).
5. Location in `requires_strong_match` AND (fit yellow/red OR company yellow/red OR compensation yellow/red) → No-go. State: "{Location} requires strong match, but role has concerns: {list signals}."
6. Two or more yellows across any combination of eligibility, fit, company, and compensation signals → Conditional (name the conditions).
7. Otherwise → Apply (note any yellows and any `requires_strong_match` location with all greens).

Conditions text: name every red or yellow signal present, not only the one that triggered this rule. Name structural gaps plainly — don't bury them. Surface career-direction shifts from Phase 2.

**Step 3 — Draft:** Draft Analysis.md in memory. Do not write to disk yet. Include: Role Summary, Evaluation (disqualifier result, eligibility signal), Breakdown (role level, core responsibilities, must-haves, nice-to-haves, framing), Fit (signal, gap table), Career Direction, Company (signal, evidence), Compensation (signal, range, confidence), Recommendation (verdict, conditions if not Apply, rationale).

**Step 4 — Present & route:** Present the full analysis to the candidate, then ask:

> [1] Yes — apply and save  [2] Save only  [3] No

Mark `(Recommended) — matches the verdict` on the option for the verdict: Apply → `[1]`, Conditional → `[2]`, No-go → `[3]`.

- `[1]` or `[2]` → record the choice, then go to Step 5 (Save).
- `[3]`:

  > Job evaluation complete.

**Step 5 — Save:**

- If folder exists: warn and ask:

  > Folder already exists. [1] Rename (Recommended) — keeps the earlier analysis  [2] Overwrite  [3] Cancel — stop

  - `[1]` Rename → ask for the new folder name, set `{folder}` to it, and continue below.
  - `[2]` Overwrite → continue below.
  - `[3]` Cancel → do not save and do not invoke `neat-job-applying`. Stop here.

    > Job evaluation complete.
- Otherwise (folder didn't exist, or Overwrite/Rename resolved): create the folder, then save `JD.md` and `Analysis.md`.

  > Written to {folder}/JD.md and {folder}/Analysis.md.

- Route based on Step 4 choice:
  - `[1]` Yes → tell user to run applying:

    > Job evaluation complete. Run /neat-job-applying {folder} to create application materials.

  - `[2]` Save only:

    > Job evaluation complete. Run /neat-job-applying when ready.

Done.
