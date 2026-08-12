# dbt Cloud + Databricks setup notes

Learning project: `jaffle_shop` on a single Databricks workspace, dbt Cloud
(trial), one catalog, schema-per-environment. This documents the setup as
built and verified, and the developer workflow from a code change to
production.

## Architecture

One Databricks workspace, one connection, one catalog (`jaffle_shop`).
Environments are separated by **schema only** — there's no per-environment
warehouse or catalog split.

```
jaffle_shop catalog (Databricks, Unity Catalog)
├── raw          ← seed data (loaded once, not part of normal deploys)
├── dbt_dev      ← personal dev work via the dbt Cloud IDE
├── dbt_stg      ← ephemeral PR/CI builds (auto-created and torn down per PR)
└── dbt_prod     ← production, source of truth for dashboards etc.
```

Schema separation is enforced by `macros/generate_schema_name.sql`, which
uses `target.schema` — i.e. whatever schema the active environment/profile
defines. Seeds always land in `raw` regardless of environment.

### Environments (dbt Cloud)

| Environment | Type | Schema | Notes |
|---|---|---|---|
| `dev` | Development | `dbt_dev` | Used by the IDE only. No jobs run here. |
| `stg` | Staging | `dbt_stg` | Runs the Slim CI job. Marked as the environment type dbt Cloud CI defers *from*'s target, but its own artifacts aren't used for deferral. |
| `prod` | Production | `dbt_prod` | Flagged as the **Production environment** in dbt Cloud — this is what Slim CI defers *to*. |

### Connections / profiles

Because the trial plan doesn't support **Extended Attributes** (the normal
way to override just the `schema` field per environment), we created **three
separate connection profiles** (`dev`, `stg`, `prod`), each pointing at the
same Databricks workspace/HTTP path/catalog but with a different schema
baked in. Each environment is assigned its matching profile.

If/when this project moves to a Team/Enterprise plan, this can be
simplified to one shared profile + per-environment Extended Attributes.

### Jobs (dbt Cloud)

Only two jobs exist:

| Job | Environment | Trigger | Commands |
|---|---|---|---|
| `prod` | `prod` | Manual (schedule can be added later) | `dbt build` (initial run also included `dbt seed --vars '{"load_source_data": true}'`, since seeds are disabled by default and only need loading once) |
| `slim CI` | `stg` | On pull request | `dbt build --select state:modified+`, deferring to the `prod` environment |

One job per environment was sufficient here; jobs are a separate concept
from environments because a real project can have multiple jobs sharing one
environment (e.g. a nightly full rebuild + an hourly incremental job both
running in `prod`).

## Local tooling

- `sqlfluff` lints SQL. `.sqlfluff` dialect is `sparksql` (this was
  originally `snowflake`, a leftover from the upstream template — fixed,
  since Databricks-compiled SQL doesn't parse under the Snowflake dialect).
- `.pre-commit-config.yaml` runs `sqlfluff-lint --templater jinja` alongside
  the existing Python hooks (ruff, YAML/whitespace checks). The `jinja`
  templater is a local approximation of dbt Cloud's `dbt-cloud` templater —
  it won't resolve `ref()`/`source()` to real table names, so a few
  templater-specific issues (e.g. column-order rules on compiled SQL) can
  still only be caught by the CI run itself. Run `pre-commit install` once
  per clone to have it run automatically on every commit; otherwise run it
  manually: `pre-commit run --all-files`.
- Removed the leftover `.github/workflows/{ci,cd_staging,cd_prod}.yml` and
  `dbt_cloud_run_job.py` — these were copied from the upstream dbt Labs
  jaffle-shop template and pointed at unrelated Snowflake/BigQuery/Postgres
  dbt Cloud projects under a different account. CI/CD is handled natively by
  dbt Cloud's GitHub integration instead (see below), so these added no
  value and were actively misleading.

## Developer workflow (verified end-to-end twice)

```
1. git checkout main && git pull
2. git checkout -b feat/my-change
3. edit models/...
4. sqlfluff lint <file> --templater jinja      (or: pre-commit run --all-files)
5. git add / commit / push
6. open PR -> main on GitHub
7. dbt Cloud Slim CI job fires automatically (GitHub webhook)
     - builds only modified models + their downstream dependents
     - defers everything else to the current `prod` state
     - writes to an ephemeral schema: dbt_cloud_pr_<job_id>_<pr_number>
     - posts a check back on the PR
8. review the ephemeral schema in Databricks if needed (only exists
   while the PR is open — dropped on merge/close)
9. merge PR into main
10. trigger the `prod` job manually in dbt Cloud
11. verify the change landed in jaffle_shop.dbt_prod
```

### What Slim CI actually builds

Confirmed by inspecting the ephemeral PR schemas directly in Databricks:

- Change to a **leaf model** (`customers`, no downstream models) → CI schema
  contained only `customers`. Its own upstream refs (`stg_customers`,
  `orders`) were **not** rebuilt — because they're deferred, `ref()` calls
  inside `customers.sql` resolved to the real `jaffle_shop.dbt_prod` tables.
- Change to a **non-leaf model** (`orders`, which `customers` selects from)
  → CI schema contained both `orders` and `customers` (state:modified+
  correctly cascaded downstream). Everything upstream of `orders`
  (`stg_orders`, `order_items`, etc.) stayed deferred to prod.

This is the core value of Slim CI/defer: a PR run only ever rebuilds what
changed plus its downstream dependents, using real production data for
everything else — fast, and representative of the real DAG.

### Gotchas hit along the way

- **CI job branch field**: for a CI-triggered job, dbt Cloud always checks
  out the PR's own branch — the "default branch" field on the job is
  effectively unused for that trigger type. A separate `staging` git branch
  is not needed and was removed.
- **Deferral target changed from job to environment** in the current dbt
  Cloud UI: "Compare changes against an environment" — pick `prod`
  directly, not a specific job.
- **GitHub reuses check results per commit SHA**, not per PR. Reopening a
  PR with the same head commit won't trigger a new CI run; only a new
  commit (even an empty one) does.
- **First prod run must happen before Slim CI can defer anything** — no
  prior successful run in the Production-flagged environment means no
  manifest to diff against.

## Still open / possible next steps

- [ ] Delete the leftover generic `Development` environment (unused,
      auto-created by dbt Cloud, not referenced by any job).
- [ ] Move deployment credentials for `stg`/`prod` off the personal
      Databricks token and onto a service principal, once this goes beyond
      a learning project — currently `stg`/`prod` reuse the same personal
      credential as `dev`.
- [ ] Add a schedule to the `prod` job (currently manual-only).
- [ ] Set up GitHub branch protection on `main` requiring the Slim CI check
      to pass before merge.
- [ ] Consider `pre-commit install` in a repo setup doc/README so the hook
      actually runs on commit for new clones, rather than only manually.
