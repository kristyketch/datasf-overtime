# SF employee overtime pipeline

Loads the city's Employee Compensation dataset into Snowflake every day and
builds two tables for an overtime dashboard: one that flags employees with
unusually high overtime, and one that summarises pay and overtime by
department.

## How the data flows

1. **Load.** `ingest/load_employee_compensation.py` downloads the full
   dataset and copies it into `RAW.CONTROLLER.EMPLOYEE_COMPENSATION`, one JSON
   value per row. It skips the load when the source has not published since
   the last one, and logs every run to `RAW.CONTROLLER.LOAD_AUDIT`.
2. **Staging.** `stg_controller__employee_compensation` turns the JSON into
   typed columns, keeps only the newest load, and adds a row id.
3. **Intermediate.** `int_compensation_fiscal_years` keeps complete fiscal
   years only. Both analytics models read from here, so they report on the
   same records.
4. **Analytics.** `employee_overtime_flags` and `department_overtime_summary`
   are the tables the dashboard reads.

| Layer | Snowflake location | Built as |
|---|---|---|
| Raw | `RAW.CONTROLLER` | Tables, written by the loader |
| Staging | `ANALYTICS_DEV.STAGING` | View |
| Intermediate | `ANALYTICS_DEV.INTERMEDIATE` | View |
| Analytics | `ANALYTICS_DEV.ANALYTICS` | Tables |

`ANALYTICS_DEV` is for development. The daily run builds the same models
into `ANALYTICS` with `dbt build --target prod`.

## Scheduling

`.github/workflows/daily_pipeline.yml` runs every day at 14:00 UTC: it loads
the data, then builds and tests the dbt models. If the load fails, dbt does
not run, so the dashboard keeps the previous day's tables. It needs three
repository secrets: `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_LOADER_KEY` and
`SNOWFLAKE_DBT_KEY` (the contents of the two key files).

## Main transformations

- **Full reload, not incremental.** The source replaces every row on each
  publish and has no stable row key, so there is nothing to merge on.
- **Raw is stored as JSON.** A new or renamed source column cannot break the
  load.
- **A loaded batch is checked before it is kept.** The row count must match
  the source and both year types must be present. Otherwise the batch is
  rolled back and staging keeps reading the previous one.
- **Staging only cleans.** It renames, casts and trims, and drops nothing.
  The only rename is `union` to `union_name`, because `union` is a reserved
  word. Codes stay text, so union code `038` keeps its zero.
- **Row id.** The source has no key, and name + job + department + year is
  not unique, so `compensation_id` hashes the columns that identify a row.
- **Complete fiscal years only** in the analytics models. Each year appears
  twice in the source, once as a calendar year and once as a fiscal year, and
  the city budgets and limits overtime by fiscal year.

### The overtime flag

A record is flagged when both are true:

- its overtime-to-salary ratio is above Q3 + 3 x IQR for its peer group;
- its overtime is at least $10,000.

The peer group is everyone with the same job code that year. A job code with
fewer than 50 records widens to the job family, then the organization group,
then citywide. Where overtime is common, the Q3 + 3 x IQR threshold decides.
Where most of a job has none, the threshold is zero and the $10,000 floor
decides, which catches one person with large overtime in a job where nobody
else has any. The four thresholds sit together at the top of the model.

I chose the ratio to salary because the source has no overtime-hours column.

### The department summary

One row per department per fiscal year: headcount, regular pay, overtime,
overtime per employee and overtime as a share of regular pay. The last is the
fairest comparison between departments. `analysis/department_overtime_questions.sql`
ranks departments both ways for the latest year, limited to departments
with at least 50 employees.

## Assumptions and limitations

- **A row is not a person.** It is one person in one job in one department
  for one year. The source has no employee id, and different people share
  names, so headcount (distinct names) is an approximation.
- **Names change spelling over the years**, so models group by code and show
  the most recent name. Some department codes were also replaced around
  2017, which breaks trends across that year.
- **Before 2017** the source has a number in place of the name, and no hours
  or employment type.
- **A flag means worth a look**, not that anything improper happened.
- **The source is treated as private.** The loader signs in with an API key
  when one is set.

## Tests

| Test | Why |
|---|---|
| `compensation_id` is unique and never null | Proves the row id really identifies a row, and catches duplicates |
| One row per department per year | Catches a join that multiplies rows |
| Department totals match the citywide total | Catches records dropped or counted twice |
| Loader checks: row count and year types | Stops a broken load reaching staging |
| `dbt source freshness` | Fails if no load has finished in 49 hours (defined, not yet in the daily run) |

What I would add next:

- Source freshness in the daily run
- Not-null tests on job code, fiscal year, department code and pay fields
- Accepted values and ranges: valid fiscal years, no unexpected negative pay
- A test that values present in raw are not null after casting in staging
- Every department code maps to a department name
- Tests that each access role sees only its intended rows and fields
- Monitoring of row counts, run time and warehouse usage after each run

## Files

| File | What it does |
|---|---|
| `ingest/load_employee_compensation.py` | Downloads the source and loads it into Snowflake |
| `.github/workflows/daily_pipeline.yml` | Runs the pipeline every day: the loader, then `dbt build` into `ANALYTICS` |
| `snowflake/setup.sql` | Creates the databases, tables, roles and service users |
| `dbt/dbt_project.yml` | Which schema and materialization each folder of models gets |
| `dbt/profiles.yml` | How dbt connects to Snowflake, for dev and prod |
| `dbt/packages.yml` | Installs `dbt_utils`, used for the row id and one test |
| `dbt/macros/generate_schema_name.sql` | Makes schema names exact, such as `ANALYTICS` |
| `dbt/models/schema.yml` | The source, model descriptions and tests |
| `dbt/models/staging/...` | Cleans the raw data |
| `dbt/models/intermediate/...` | Complete fiscal years only |
| `dbt/models/analytics/...` | The two dashboard tables |
| `dbt/tests/assert_department_totals_match_source.sql` | Department totals add up to the citywide total |
| `analysis/department_overtime_questions.sql` | Answers which departments have the most overtime |
| `requirements.txt`, `.env.example`, `.gitignore` | Python packages, settings, and what git ignores |
