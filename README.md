## App Link: https://study-planner-app.apps.lemma.work/
# AI Student Study Planner

A focused one-table-one-agent-one-app pod that takes a student's **goal**, **daily
hours**, and **deadline**, generates a personalized day-by-day study plan via an
LLM, and saves it to a shared table so previous plans show up in the app
alongside any new ones.

## Layout

```
study-planner-bundle/
  pod.json
  tables/studyplans/studyplans.json      # StudyPlans table (RAG off, shared)
  agents/study-planner/study-planner.json + instruction.md
  apps/study-planner-app/study-planner-app.json + html.html
  seed/seed.sh                            # CLI script to add the 3 sample rows
```

The CLI normalises table names to lowercase, so the technical table name is
`studyplans` even though the user-facing label is **StudyPlans** (set on the
table description and on the app).

## Resources

| Resource           | Name                  | Notes |
| ------------------ | --------------------- | ----- |
| Datastore table    | `studyplans`          | 7 user columns (see below); RLS off so seed rows are visible across users. |
| AI agent           | `study-planner`       | Toolsets `POD`; grants `studyplans:read+write`. Greets, asks for the four facts, generates a markdown plan, and saves one new row. |
| App                | `study-planner-app`   | Single HTML page: New plan form on the left, library on the right. |

### `studyplans` columns

| Column              | Type     | Required | Notes |
| ------------------- | -------- | -------- | ----- |
| `student_name`      | TEXT     | yes      | `max_length: 120` |
| `study_goal`        | TEXT     | yes      | `max_length: 500` |
| `daily_study_hours` | FLOAT    | yes      | 0.5–12 typical |
| `deadline`          | DATE     | yes      | target completion date |
| `generated_plan`    | TEXT     | yes      | full markdown plan |
| `progress`          | INTEGER  | yes      | 0–100, default 0 |
| `created_date`      | DATE     | yes      | explicit per the spec; system `created_at` is also stamped automatically |

System-managed columns (`id`, `created_at`, `updated_at`) are added automatically.

## Import

The pod shell already exists; the bundle imports via upsert by resource name:

```bash
cd study-planner-bundle
lemma pods import . --dry-run
lemma pods import .
```

Errors fail fast on the first issue encountered. Re-imports update rather than
duplicate (`agents`/`functions`/`workflows` replace grants from the bundle JSON).

## Seed (creates sample plans so the app demos itself)

```bash
./seed/seed.sh
```

Inserts three plans — **Maria / Spanish in 21 days**, **Tom / GRE Quant in 6 weeks**,
**Aisha / Python data analysis in 30 days** — each with a different progress %
and a multi-week markdown plan, so the **Previous plans** panel is alive on
first open.

## Verify the hero moment

```bash
# 1. Confirm the table has the seed rows.
lemma records list studyplans --limit 5

# 2. Chat with the agent directly (it should collect missing facts and save).
lemma agents chat study-planner "Hi, I want to learn to bake sourdough"

# 3. Open the deployed app in the host browser.
lemma apps open study-planner-app
```

## Hero scenario

1. Open the app — three sample plans are visible on the right.
2. Fill in: name, goal, daily hours, deadline → click **Generate plan**.
3. The agent (visible in-page via `<lemma-agent-task>`) drafts and saves the new plan.
4. The library auto-refreshes — the new plan is at the top, click **Show plan** to read it.
5. Open the row in the table (`lemma records list studyplans`) — full plan and metadata are persisted.
