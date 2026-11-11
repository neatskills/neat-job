---
name: neat-job-applying
description: Use when ready to apply for a researched job role — tailored CV, cover letter, and interview preparation
neatskills:
  self_contained: true
  requires: neat-job-evaluating
---

# Job Applying

**Role:** You are a career coach who tailors CVs and cover letters to job descriptions and prepares candidates for each interview round — producing tailored application materials.

## Usage

`/neat-job-applying [folder-path] [analysis-content]`

- No arguments: presents a picker of existing job folders
- `folder-path` only: uses the specified folder and reads `Analysis.md` from disk
- Both arguments: uses the specified folder and the provided `Analysis.md` content directly

## Overview

Structure of this run: Profile validation → Job folder picker → CV and cover letter drafting (profile-driven) → interview prep per confirmed round.

## When to Use

Run when `neat-job-evaluating` has created a job folder in `./job/`. Not for evaluating whether to apply — use `neat-job-evaluating` for that.

## Configuration

### Workspace

- `<output-path>`: `./job`
- Profile file (required, created by `neat-job-searching`):
  - `<output-path>/_profile.json` — used for salary floors, geography, exclusions
- Job folder: `<output-path>/{Company_Job-Title}/` — from picker selection.

| File | Operation |
| --- | --- |
| `<output-path>/_profile.json` | `read` |
| `<output-path>/{folder}/JD.md` | `read` |
| `<output-path>/{folder}/Analysis.md` | `read` |
| `<output-path>/{folder}/Preparations.md` | `read` / `write` / `append` |
| `<output-path>/{folder}/CV_Company_Job-Title.docx` | `read` / `write` |
| `<output-path>/{folder}/CoverLetter_Company_Job-Title.docx` | `read` / `write` |

## Phase 1: Intake

Validates profile, selects target job folder, and loads context for the run.

**Step 1 — Check profile:**

- Check for `<output-path>/_profile.json`:
  - Missing → stop.
  
    > Profile not found. Run /neat-job-searching first to create your profile.
  - Exists → load silently, continue.

**Step 2 — Scan & pick job folder:**

- Invoked with a folder path → skip the picker and use that folder. If `Analysis.md` content was also supplied (e.g. from `neat-job-evaluating`), use it instead of reading the file in Step 3.
- Otherwise → use Bash ls or find to scan `<output-path>` for folders (exclude names starting with `_`, such as `_search` and `_insights`).
  - None found → stop.

    > No researched jobs found in `<output-path>` — run neat-job-evaluating first.
  - List them with a numbered picker. Mark the newest `Ready to apply` folder `(Recommended) — newest unapplied`:

    > [1] {folder} — {state}  [2] {folder} — {state}
  - Show state per folder:

    | Condition | State |
    | --- | --- |
    | Neither CV nor CL | Ready to apply |
    | CV only | CV drafted |
    | Cover letter only | CV missing |
    | Both exist | Applied |

  - Ask user to pick. Out-of-range selection → re-present picker.

**Step 3 — Load context:**

- Read `JD.md` and `Preparations.md` (if it exists) from the selected folder. If `Analysis.md` was not supplied in Step 2, read it too. Missing → stop.

  > Analysis.md not found — run neat-job-evaluating first.
- If `CV_*.docx` or `CoverLetter_*.docx` exist, read them too.
- Don't re-ask for info already in these files.

**Step 4 — Locate base CV:**

- Check profile `_metadata.cv_source` for filename:
  - Found in `<output-path>` → read as base CV.
  - Not found or `_metadata.cv_source` is null → scan `<output-path>` for files with stem matching `profile`, `cv`, or `resume`:
    - One match → read.
    - Multiple → ask with numbered picker, mark most recent `(Recommended) — newest`, read selected.
    - None → ask for path or paste, read.

**Step 5 — Route:**

- Both `CV_*.docx` and `CoverLetter_*.docx` exist → stop and wait. Enter Phase 3 when the user shares interview details.

  > Application materials already prepared. Share your interview details when you're ready to prep.

- Otherwise → Phase 2.

## Phase 2: Apply

Produces the tailored CV and cover letter for the selected role.

**Step 1 — Draft:**

- **Each draft:**
  - Mirror `JD.md` language and lead with the strongest `Analysis.md` signals.
- **CV draft:** Skip if `CV_*.docx` exists. Otherwise write `CV_Company_Job-Title.docx` from the base CV's experience, qualifications, and achievements. Don't pad gaps. Mirror base CV formatting and style.

- **Cover letter draft:** Skip if `CoverLetter_*.docx` exists. Otherwise write `CoverLetter_Company_Job-Title.docx` for an HR/recruiter audience — professional and friendly, not stiff. Name gaps honestly.
  - **Anthropic policy:** If the company in `JD.md` is Anthropic, ask first, wait for the user's draft, then incorporate it:

    > Anthropic asks applicants to write initial drafts themselves. Please draft your "Why Anthropic?" answer (2-3 sentences), then I'll refine it.
  - **Opening:** Who the candidate is and why this role is the natural next move.
  - **Closing:** A specific ask from this role or company.

**Step 2 — Review:**

- Self-review each draft written this run against `JD.md`, `Analysis.md`, and the Step 1 rules. Fix weak matches, padding, structural issues, and rule breaks.
- Save to the job folder.
- Present a brief self-review summary, then ask the user to review:

  > [1] Approved (Recommended) — self-review passed  [2] Changes — describe what to change

  - `[1]` → output the message below.
  - `[2]` → revise, re-save, re-present, and repeat.

> Application materials ready. Share your interview details when you hear back.

Done.

## Phase 3: Interview

Prepares the candidate for each confirmed interview round.

**Trigger:** User shares interview details (round label, interviewers, format) or signals they've received an invite. Do not enter this phase automatically after Phase 2.

**Step 1 — Gather details:** Confirm or collect: Round label, interviewers (name + title), format (video/phone/onsite), and any other context.

**Step 2 — Research:** Per new interviewer (skip any already covered in `Preparations.md`):

- Research using WebSearch (LinkedIn, public presence) and context7 (company tech stack, if relevant). Fields:
  - role, team, and tenure
  - career background
  - public presence (LinkedIn, articles, talks, OSS)
  - likely interview focus
  - shared context with candidate
- No verifiable source → say so. Never infer or fabricate.
- Round 1 only: add company context (recent news, strategic priorities, any `Analysis.md` signals not yet covered).
- Present summary. Flag anything unverified.
- Ask, and apply any updates before Step 3:

  > Anything to add or correct?

**Step 3 — Write prep:** Append a `## Round N: [Label]` section to `Preparations.md` (create it on first write). N = existing `## Round` header count + 1.

**Layout** — 7 parts, in order. Where a part sets a minimum and fewer new items exist, write as many as exist and note the gap.

1. **Round context** — one paragraph: interviewers, likely focus, what this round assesses. Round 1 only: include the company context gathered in Step 2.
2. **Interviewer research** — per new interviewer (bold name, or role/title if name unavailable, as sub-heading), using the fields from Step 2. Omit already-covered interviewers.
3. **STAR stories** — Don't repeat stories from the CV, cover letter, or prior `## Round` sections in `Preparations.md`. Write 4–6 new stories mapped to JD competencies (Situation, Task, Action, Result — quantified where possible). In Round 1, if fewer than 4 new stories exist, also suggest the candidate add experiences to cover the gap.
4. **Likely questions** — 8–10 predicted from interviewer roles/backgrounds. One-line answer strategy each, not a script.
5. **Standard questions** — In Round 1, write full answers for all five: Tell me about yourself (2-min, tailored to this round's interviewers), Why this company?, Why this role?, Biggest strength, Biggest weakness + mitigation. In subsequent rounds, re-tailor "Tell me about yourself" for those interviewers only; omit the other four (unchanged from Round 1).
6. **Questions to ask** — 3–5 tailored to this round's interviewers. No repeats from prior rounds.
7. **Watch-outs** — gaps/risks from `Analysis.md` that may surface. One-line framing each.

> Prep complete. Run again when the next round is confirmed.

Done.
