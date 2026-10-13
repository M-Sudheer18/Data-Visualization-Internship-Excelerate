# Learner & Opportunity Analytics

**Excelerate Data Visualization Associate Internship, Team 34**
Sudheer Muthyala · Simpy Mandal · Krishna Swamy Tejesh Kumar

An end-to-end analytics project that turns two raw platform exports (**Learner** and **Opportunity** records) into a clean, integrated PostgreSQL dataset and Looker Studio dashboards. The dashboards show applications, payments, workflow stages and engagement across opportunity types.

---

## Table of Contents

1. [Project Objective](#project-objective)
2. [Tech Stack](#tech-stack)
3. [Datasets](#datasets)
4. [Pipeline Overview](#pipeline-overview)
5. [Data Cleaning](#data-cleaning)
6. [Data Integration](#data-integration)
7. [Data Quality and Remaining Missing Values](#data-quality-and-remaining-missing-values)
8. [Key Findings](#key-findings)
9. [Dashboards](#dashboards)
10. [Recommendations](#recommendations)
11. [Assumptions and Design Decisions](#assumptions-and-design-decisions)
12. [Known Limitations](#known-limitations)
13. [Reproducing the Project](#reproducing-the-project)
14. [Repository Structure](#repository-structure)

---

## Project Objective

Excelerate stakeholders need to understand how learners apply to and complete opportunities. The two source exports were messy and not directly joinable, so this project:

- Sets up a cloud PostgreSQL environment and connects it to Looker Studio
- Cleans and validates both datasets
- Integrates them into a single analytical table
- Surfaces KPIs and trends through interactive dashboards
- Turns the findings into evidence-based recommendations

The work ran over **3 weeks**: setup and exploration (Week 1), cleaning, integration and validation (Week 2), dashboards and insights (Week 3).

## Tech Stack

| Layer | Tool |
|---|---|
| Database | PostgreSQL, hosted on **Supabase** |
| Local development / SQL | **pgAdmin 4**, Supabase SQL Editor |
| Visualization | **Google Looker Studio** (PostgreSQL connector, SSL) |
| Language | SQL (PostgreSQL dialect) |

Supabase was chosen because it gives a managed cloud PostgreSQL instance with a built-in SQL editor, and it works directly with Looker Studio's PostgreSQL connector.

## Datasets

| Dataset | Table | Key | Raw rows | Clean rows | Columns |
|---|---|---|---|---|---|
| Learner | `learner` → `learner_clean` | `sk` | 15,029 | **14,793** | 43 |
| Opportunity | `opportunity` → `opportunity_clean` | `opportunity_id` | 5,733 | **5,728** | 33 |

- **Learner data:** application ID, status, assigned cohort, apply and completion dates, payment status, timestamps.
- **Opportunity data:** category, badge, duration and duration type, fee, location, eligibility, rewards, auto-approve and archive flags.

Multiple learners can apply to the same opportunity, which is why the Learner table is much larger.

## Pipeline Overview

```
Raw CSVs ─► PostgreSQL (pgAdmin) ─► Supabase ─► Cleaning ─► Integration ─► Validation ─► Looker Studio
```

1. **Database setup:** create `learner` and `opportunity` tables locally, import the CSVs, verify row counts, migrate to Supabase.
2. **Cleaning:** remove corrupted rows, standardize categories and locations, convert blanks to SQL `NULL`.
3. **Integration:** join the two cleaned tables on a normalized shared ID.
4. **Validation:** verify record counts, duplicate keys and missing-value patterns.
5. **Dashboarding:** build KPI cards and charts in Looker Studio, connected live to Supabase.

## Data Cleaning

### Learner (`learner_clean`)
- Removed **236 corrupted rows** containing embedded PDF/Base64 data
- Replaced invalid category value `"XploreU"` with `NULL`
- Validated `payment_status` (`paid` / `unpaid`) and workflow `status` without altering legitimate source values
- Reviewed date columns for consistency
- Found duplicate `pk` values in Week 1 (304 repeated values), then resolved them during cleaning

### Opportunity (`opportunity_clean`)
- Removed **5 corrupted rows** (malformed IDs, JSON fragments, bad testimonial data)
- Standardized location variants (`virtual`, `Virtual`, `WORK FROM HOME`, `wfm`, `vitrual`, ...) into two labels: **Virtual** and **Work From Home**
- Replaced invalid category `"XploreU"` with `NULL`
- Converted text `"NULL"` strings to real SQL `NULL`
- Confirmed `opportunity_id` is unique (no duplicates)
- Confirmed `pk` is just the constant prefix `Opportunity#`, not a real primary key

## Data Integration

At first there was no obvious shared column. Investigation showed that `learner.sk` and `opportunity.opportunity_id` hold the **same underlying ID with different prefixes**:

```
Learner#0000000010Z56C78JF7M6VQXQ2
Opportunity#0000000010Z56C78JF7M6VQXQ2
```

The prefixes are stripped with `REPLACE()` and the tables are joined with an `INNER JOIN`:

```sql
CREATE TABLE learner_opportunity AS
SELECT
    l.*,
    o.opportunity_id,
    o.name     AS opportunity_name,
    o.category AS opportunity_category,
    o.fee      AS opportunity_fee,
    o.location AS opportunity_location,
    o.reward   AS opportunity_reward
    -- ...plus badge, duration, duration_type, eligibility, is_archived,
    --    is_auto_approve, timestamps, etc.
FROM learner_clean l
JOIN opportunity_clean o
  ON REPLACE(l.sk, 'Learner#', '') = REPLACE(o.opportunity_id, 'Opportunity#', '');
```

**Result: 14,791 matched learner–opportunity records.**

After the join:
- Duplicate column names (`pk`, `category`, `fee`, `created_at`, `modified_at`) were renamed (for example `opportunity_category`, `opportunity_fee`) to avoid ambiguity.
- Blank strings in `status` and `opportunity_location` were converted to `NULL`.

## Data Quality and Remaining Missing Values

A final validation pass quantified the remaining gaps. Each traces back to the source data, so the values were kept as `NULL` rather than guessed.

| Column | Missing | Reason |
|---|---|---|
| `opportunity_name` | 1 | Missing in source |
| `opportunity_category` | 7 | Invalid `XploreU` values intentionally nulled |
| `payment_status` | 0 | Fully populated |
| `status` (workflow) | 5,741 | Not captured for records awaiting a workflow update |
| `opportunity_location` | 3,718 | Left blank in source; not inferred |

Replacing these with defaults such as "Unknown" or "Work From Home" would introduce inaccurate information, so they were left as `NULL`.

## Key Findings

Based on the `learner_opportunity` table (14,791 applications, 5,728 opportunities):

**Applications by category**

| Category | Applications |
|---|---|
| Internship | 5,179 |
| Competition | 2,201 |
| Career | 2,053 |
| Event | 1,746 |
| Course | 1,322 |
| Engagement | 943 |
| Masterclass | 910 |
| JobSimulation | 423 |
| Program / unclassified | 7 / 7 |

**Other highlights**
- **Payment:** 8,529 unpaid, 504 paid and 5,758 not recorded. About 94% of recorded payments are unpaid, so most opportunities are free to join.
- **Location:** Work From Home 6,076 (41%), Virtual 4,941 (33%), not recorded 3,718 (25%).
- **Duration:** `weeks` dominates with 9,650 applications, ahead of `month` (1,151) and `hours` (1,007).
- **Workflow:** status code `1070` is the largest recorded stage (3,335). Codes are numeric platform stages, not descriptive labels.
- **Fees:** 11,708 applications are for fee `0`, followed by fee `1` (1,413).
- **Cohorts:** 9,239 applications have no assigned cohort. Among the rest, `B9ZYF9D` (68) and `B28EPFO` (62) lead.
- **Top opportunities:** "Trainee Internship" (119), "Bulk Users Applied to Internship" (69), "Competition with 4 Members" (43).

## Dashboards

The Looker Studio dashboard connects live to the Supabase database. It is designed to include:

- **KPI cards:** total applications, sign-ups, completion rate, rewards
- **Category and trend charts:** applications by opportunity category and over time
- **Filters:** opportunity category, lead source, scholarship category, completion status, date range
- **Reward and source views:** reward-type distribution and lead-source performance

## Recommendations

1. **Expand internship supply.** Internships draw the most demand by far.
2. **Close the workflow-status gap.** Require or default a workflow status at intake to fix the 5,741 missing records and enable funnel reporting.
3. **Grow low-adoption categories.** Masterclass and JobSimulation have headroom for targeted campaigns.
4. **Standardize location capture.** Make a standardized location field mandatory for new listings.
5. **Favor short formats.** Week-long opportunities dominate applications.
6. **Investigate cohort concentration.** Study high-volume cohorts and replicate what works.

## Assumptions and Design Decisions

- `sk` and `opportunity_id` represent the same entity once prefixes are removed.
- `XploreU` category values are invalid and treated as missing.
- Blank strings (`''`) and the text `"NULL"` both mean missing and become SQL `NULL`.
- Workflow status codes `103` and `110` were kept, since nothing showed they were wrong.
- Source anomalies are preserved, not overwritten, to protect data integrity.
- An `INNER JOIN` keeps only records matching in both tables (14,791 of 14,793 learner rows).

## Known Limitations

- **Week 3 report figures need verifying.** Its sample dashboard screenshots show values such as 12,458 total applications, 82.3% completion rate, $125,400 in rewards, and named opportunities like "Data Analytics Workshop". These do **not** match the SQL-validated dataset (14,791 applications, and the top opportunity is "Trainee Internship"). They look like template or placeholder visuals, and the report's "Observation / Business Insight" text is still unfilled. Replace them with real screenshots from the live dashboard before relying on those numbers.
- About 25% of applications have no location and about 39% have no workflow status, which limits those breakdowns.
- Workflow status codes are numeric and undocumented here. A code-to-label mapping would make funnel analysis much clearer.

## Reproducing the Project

1. **Create tables** in local PostgreSQL (`learner`, `opportunity`), using `TEXT` columns, and import the CSVs.
2. **Verify** the row counts: 15,029 learner rows and 5,733 opportunity rows.
3. **Migrate** the database to a Supabase PostgreSQL project.
4. **Clean:** build `learner_clean` and `opportunity_clean` using the steps above.
5. **Integrate:** run the `CREATE TABLE learner_opportunity AS ...` query.
6. **Validate:** check counts, duplicate `opportunity_id` values and `NULL` counts.
7. **Connect Looker Studio:** use the PostgreSQL connector with the Supabase host, database name, user, password and SSL enabled, then point it at `learner_opportunity`.

Example validation query:

```sql
SELECT
  COUNT(*)                                   AS total_applications,
  COUNT(DISTINCT sk)                         AS unique_learners,
  COUNT(DISTINCT opportunity_id)             AS unique_opportunities
FROM learner_opportunity;
```

## Repository Structure

Suggested layout for the project deliverables:

```
.
├── README.md
├── reports/
│   ├── Week1_Data_Setup_and_EDA.pdf
│   ├── Week2_Cleaning_Integration_Validation.pdf
│   └── Week3_Dashboard_and_Visualization.pdf
├── presentation/
│   └── Final_Presentation_Team34.pdf
└── sql/                 # cleaning, integration and validation queries
```

---

*Excelerate Data Visualization Associate Internship, Team 34.*
