# Implementation Plan — Healthcare Data Roles: India Job Market Analysis

**Goal:** Collect live healthcare / health-tech job postings in India via the Adzuna API, clean and model them in SQLite, answer seven SQL questions, and publish a Tableau Public dashboard — evidencing the full data-analysis lifecycle (extract → clean → model → query → visualise → report) on self-collected data.

**One claim to evidence:** *"I can own a data analysis from extraction through to a published dashboard, on data I collected myself."*

**Estimated effort:** ~16 hours (a weekend plus two evenings).

---

## Phase 0 — Feasibility & coverage check *(gate, ~20 min)*

> Do this **before** any other commitment. If India healthcare coverage is thin, the plan changes here and nothing downstream is wasted.

- [ ] Sign up at `developer.adzuna.com`, obtain free `app_id` and `app_key` (instant, no card).
- [ ] Run a handful of manual test queries against the India endpoint for a few role terms (`data analyst`, `healthcare analyst`, `clinical data`) across 1–2 cities.
- [ ] Count results per query. Estimate whether ≥1,000 unique postings is realistic.
- [ ] **Decision point:**
  - Coverage healthy → proceed to Phase 1.
  - Coverage thin → broaden role terms, **or** reframe as *India vs. another market* comparison (arguably more interesting).

**Exit criteria:** credentials working; a documented estimate of expected posting volume.

---

## Phase 1 — Repository scaffolding & environment *(~30 min)*

- [ ] Initialise project structure:
  ```
  collect.py            # API extraction
  clean.py              # cleaning + loading into SQLite
  analysis.sql          # the seven commented queries
  /exports              # CSVs behind the dashboard
  README.md             # the document that carries the project
  requirements.txt      # requests, etc.
  .env.example          # ADZUNA_APP_ID, ADZUNA_APP_KEY (never commit real keys)
  .gitignore            # .env, __pycache__, *.sqlite (decide: commit DB or not)
  ```
- [ ] Pin config: search terms (roles), cities, pages-per-call depth (3), API rate limits.
- [ ] Set up secret handling via environment variables (keys stay out of git).

**Search terms to collect:**
- **Roles:** `data analyst`, `business analyst`, `healthcare analyst`, `clinical data`, `health informatics`, `medical coder`, `healthcare data`
- **Cities:** Bangalore, Gurugram, Hyderabad, Pune, Noida, Mumbai, Chennai
- ~150 calls total (7 roles × 7 cities × 3 pages) — comfortably inside one day's free quota (250/day, 25/min).

**Exit criteria:** repo skeleton committed; config centralised; a dry-run of the call-count math confirmed under quota.

---

## Phase 2 — Extraction: `collect.py` → `postings_raw` *(~3–4 hrs)*

- [ ] One Python script paging through each role × city combination.
- [ ] Endpoint: `https://api.adzuna.com/v1/api/jobs/in/search/{page}`, `app_id` + `app_key` as query params, up to 50 results/call, 3 pages deep.
- [ ] Write **raw, untouched JSON** straight into `postings_raw` (never edited — keeping it means a cleaning mistake costs nothing).
- [ ] **Sleep between calls** to stay under 25/minute.
- [ ] Never re-run a collection without versioning the output (record collection date).
- [ ] Capture fields: `title`, `company`, location hierarchy, `category`, `description`, `salary_min`, `salary_max`, `salary_is_predicted`, creation date, redirect URL.

**Exit criteria:** `postings_raw` populated; raw row count logged; collection date recorded.

---

## Phase 3 — Cleaning & modelling: `clean.py` → SQLite *(~3–4 hrs)*

> The cleaning decisions are the most valuable part of the project. **Every rule and its reasoning goes in the README.**

**Data model (three tables in SQLite):**
- `postings_raw` — untouched API output.
- `postings` — one row per cleaned posting (`posting_id` PK, `title_raw`, `role_bucket`, `seniority`, `company`, `city`, `salary_min`, `salary_max`, `salary_is_predicted`, `created_date`, `description`).
- `posting_skills` — one row per skill found in a posting (`posting_id`, `skill`). Separate table so one posting carries many skills — this is what makes the co-occurrence (self-join) query possible.

**Cleaning rules (each documented with reasoning):**
- [ ] **Deduplicate** on `posting_id` (same posting appears under several search terms). Record how many removed.
- [ ] **Normalise job titles** → `role_bucket` (Data Analyst, Business Analyst, Clinical/Health Data, Other) by keyword in priority order. Log every title falling through to `Other`. If `Other` > 20% of rows, the mapping needs work.
- [ ] **Normalise cities** (collapse Bangalore/Bengaluru, Gurgaon/Gurugram, Bombay/Mumbai).
- [ ] **Salaries** — the decision that matters most. Many are model-predicted (`salary_is_predicted = 1`). Salary analysis uses only rows where the flag is `0`. README states how many rows survived and why the rest were excluded.
- [ ] **Extract skills** → `posting_skills`. Case-insensitive search of the description per term, **require word boundaries**, skip single-letter terms like `R` (cannot be matched reliably — say so in the README rather than pretend the match is clean).
  - Skills to detect: SQL, Python, Excel, Power BI, Tableau, R, SAS, AWS, Azure, Snowflake, Databricks, EHR, HL7, FHIR, ICD, claims, HIPAA.
- [ ] **Validate before analysing:** row counts at each stage, nulls per column, date range of postings, share of rows with a stated salary — these numbers go in the README.

**Exit criteria:** three tables loaded; validation numbers captured; ≥1,000 unique cleaned postings; cleaning log written.

---

## Phase 4 — Analysis: `analysis.sql` (seven queries) + CSV exports *(~4–5 hrs)*

> One commented `.sql` file. Each query must be explainable from memory in an interview. This is where SQL is actually learned — budget extra time if the patterns are new.

| # | Question | SQL it teaches |
|---|----------|----------------|
| 1 | Postings per role bucket and city | `GROUP BY`, `COUNT` |
| 2 | Which skills appear most often overall | `JOIN`, `GROUP BY`, `ORDER BY` |
| 3 | Skill demand: Data Analyst vs. Clinical/Health Data | `JOIN` + conditional aggregation |
| 4 | Which skills appear together in the same posting | **self-join** on `posting_skills` *(most likely interview probe)* |
| 5 | Do SQL-mentioning postings pay more? | `CASE WHEN`, `AVG`, filtered to stated salaries |
| 6 | Skills in senior postings but not junior ones | CTE + `EXCEPT` / `NOT IN` subquery *(**the headline finding**)* |
| 7 | Share of postings stating a salary at all | aggregate over a flag |

- [ ] Write and debug all seven queries with inline comments.
- [ ] **Export each result set to CSV** into `/exports` (dashboard connects to CSVs, not SQLite — keeps SQL visible in the repo for reviewers).
- [ ] Note the headline (Q6: junior-vs-senior skill gap → actionable recommendation) and the interview favourite (Q4: self-join).

**Exit criteria:** seven queries run correctly; CSVs in `/exports`; headline finding identified with a number.

---

## Phase 5 — Dashboard: Tableau Public *(~3 hrs)*

> Not optional — a visualisation tool is an explicit hiring requirement, and Tableau Public gives a shareable public URL (Power BI free tier does not, and Power BI Desktop doesn't run on Mac).

Four views, one page, with a **city filter**:
- [ ] **Top skills by demand** — horizontal bar, sorted descending.
- [ ] **Skill demand by role bucket** — grouped bars, Data Analyst vs. Clinical/Health Data.
- [ ] **Junior vs. senior skill gap** — diverging bar (the headline finding).
- [ ] **Postings over time** — line, by week collected.
- [ ] Connect Tableau to the `/exports` CSVs (not SQLite).
- [ ] Publish to Tableau Public; capture the public URL.

**Exit criteria:** dashboard live at a public URL; four views working; city filter functional.

---

## Phase 6 — README, finding & final push *(~2 hrs)*

> A reviewer spends two minutes in the README and may never open the code. **The README carries the project.**

- [ ] Structure the README: the question → the data and how it was collected → the cleaning decisions and why → the findings → the limitations → what I'd tell a student to learn.
- [ ] State **one finding with a number and a recommendation attached** (from Q6).
- [ ] Link the live Tableau dashboard in the README (and ready for the resume).
- [ ] Write the **Known limitations** section (this is the strongest signal in the project):
  - **Coverage** — Adzuna is strongest in UK/Western Europe; India healthcare volume may be thin; data is one aggregator, not the whole market, reflecting the weeks collected (date the collection).
  - **Salary data is partial** — report surviving sample size wherever a salary figure appears.
  - **Keyword skill matching is crude** — word-boundary matching still misses unusually named skills and can't handle single-letter terms; state the method and its failure modes.
  - **No causal claim** — SQL-mentioning postings paying more is association, not causation (senior roles may simply list more tools).

**Done means all five are true:**
- [ ] At least 1,000 unique postings collected and cleaned.
- [ ] Every cleaning decision written down with its reasoning.
- [ ] All seven queries run and are explainable without notes.
- [ ] The dashboard is live at a public URL.
- [ ] The README states one finding with a number and a recommendation attached.

**Out of scope (by design):** machine learning / predictive modelling, scraping job sites directly, any claim about causation.

---

## Phase dependency summary

```
Phase 0 (gate) → Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 ─┐
                                              └──────────────→ Phase 6 (README)
```

Phases 2→4 are strictly sequential (each consumes the previous output). Phase 5 depends on Phase 4's CSV exports. Phase 6 (README) is written throughout but finalised last, pulling in validation numbers, cleaning decisions, the finding, and the dashboard link.
