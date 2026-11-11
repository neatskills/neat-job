# Neat Job

Claude Code skills for job search and career development.

## Skills

### `neat-job-*` — Job search workflow

**Core workflow (profile-driven, self-contained):**

1. **neat-job-searching:** Entry point — creates profile on first run, searches LinkedIn, evaluates jobs in parallel, produces ranked shortlists. User provides keywords (role titles + domains), system learns which keywords perform best.

2. **neat-job-evaluating:** Evaluates a single job opportunity against profile — Apply/No-go/Conditional verdict with Analysis.md.

3. **neat-job-applying:** Tailors CVs and cover letters to job descriptions, prepares for interview rounds — application materials per job.

4. **neat-job-insights:** Periodic market intelligence — analyzes historical search data to show what you're looking for, skill gaps, market trends, and search optimization.

**How they work together:**
```
First run (entry point):
  /neat-job-searching → creates _profile.json from CV → searches → evaluates → shortlist
  
Subsequent searches:
  /neat-job-searching → finds new jobs → evaluates (parallel) → ranked shortlist
    ↓ (pick jobs to apply)
  /neat-job-applying {folder} → CV + cover letter
  
Single job evaluation:
  /neat-job-evaluating → Analysis.md
  /neat-job-applying {folder} → application materials
  
Market intelligence (periodic, standalone):
  /neat-job-insights → skill trends, salary benchmarks, search effectiveness
```

**Profile-driven:** 

- All skills read `_profile.json` (salary floors, geography, search keywords, exclusions).

- Self-improving: learns recruiter patterns, keyword effectiveness, skill gaps from your Apply jobs.

- Portable: upload skills to any LLM environment.

## Insights Provided

**neat-job-insights** analyzes your job search history to provide 7 types of insights:

**Personal Insights** (from your Apply jobs):

1. **Apply Job Profile** — What roles, domains, and company types you're actually applying to *(10+ Apply jobs)*

2. **Skill Gaps** — Skills required by Apply jobs that aren't in your CV *(10+ Apply jobs)*

**Market Insights** (from all evaluated jobs):

3. **Skill Demand** — Hot, emerging, stable, and declining skills in the market *(50+ jobs)*

4. **Salary Landscape** — Compensation benchmarks by company type and geography *(50+ jobs)*

5. **Geographic Landscape** — Job distribution and Apply rates by location *(50+ jobs)*

6. **Market Trends** — Skill, salary, and geographic shifts over time *(50+ jobs, 30+ days)*

7. **Market Health** — Overall Apply rate and top No-go reasons *(30+ jobs)*

**Self-Improving:**

- Tracks which keywords produce higher Apply rates *(3+ search runs)*

- Auto-suggests better keywords for next search based on proven performance

## License

[MIT](LICENSE)
