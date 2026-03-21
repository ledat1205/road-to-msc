![[dbt_cheatsheet.html]]
Here are some of the **most common points of confusion, misconceptions, and mistakes** that appear repeatedly in industry dbt (data build tool) projects, especially once teams move beyond simple proof-of-concepts into production-scale analytics engineering.

1. **dbt compile can catch SQL syntax & semantic errors**  
   Many people assume `dbt compile` (or even `dbt run --dry-run`) will reliably detect broken SQL. In reality it mostly checks Jinja → SQL rendering and basic DAG structure — **not** full SQL syntax validation or schema existence in many cases. Real syntax errors or missing columns often only blow up at runtime.

2. **Staging / intermediate / marts is the only / best layering strategy**  
   Teams treat this as a rigid religion and force everything into exactly three layers. In practice many mature projects need more nuance (e.g. separate layers for conformed dimensions, audit/history tables, export-ready views, semantic-layer friendly entities, or even "reporting intermediate" vs "analytical intermediate"). Blindly following "one true layering" creates overly fragmented or duplicated logic.

3. **Incremental models with delete+insert when unique_key isn't really unique**  
   People configure incremental models with `delete+insert` strategy and a `unique_key` that has duplicates (or is NULLable / not enforced). This silently produces wrong results or duplicates instead of failing loudly.

4. **Using materialized='table' everywhere "just to be safe"**  
   This causes full table rewrites on every run → massive compute cost explosion and slow runs. Many teams don't realize how much cheaper/faster ephemeral views or incremental tables can be when used appropriately.

5. **Confusing dbt's role: thinking it's an orchestrator / scheduler**  
   dbt is a **transformation engine**, not Airflow / Dagster / Prefect. A very common misconception is "dbt will handle scheduling and retries" — it won't. People get surprised when runs fail silently overnight or dependencies aren't retried.

6. **Not testing seriously → "tests are optional / nice-to-have"**  
   The single most damaging habit in production teams. No (or very weak) generic + singular + custom tests means bad data flows downstream for weeks/months before anyone notices. Related: writing tests but never running them in CI/CD.

7. **Hardcoding schema / database names everywhere**  
   This breaks lineage, makes environments (dev / staging / prod) painful to manage, and kills `dbt run --select ...` reusability. People keep confusing "I can just change it later" with actually doing it properly via `{{ target.schema }}` + custom targets.

8. **Over-relying on views for everything → performance death spiral**  
   Views are great for dev speed, but on large datasets they cause repeated computation, query-plan explosion, and warehouses choking. Teams often realize too late that 80% of their "core" should be tables / incrementals.

9. **Thinking dbt Semantic Layer fixes everything automatically**  
   Many assume slapping metrics on top magically creates "one version of the truth". In reality it only helps if the **underlying models** are clean, tested, and consistently defined — otherwise you get beautifully consistent wrong numbers.

10. **No governance → "anyone can write models anywhere"**  
    Without style guides, naming conventions, review processes, or dbt Explorer / docs usage, projects turn into unmaintainable spaghetti very quickly (especially >10–15 people contributing).

11. **Expecting Python models to behave exactly like SQL models**  
    With newer Python support, people assume full feature parity (incrementals, tests, etc.). Behavior, performance, and limitations still differ significantly in many adapters.

12. **Ignoring costs until it's too late**  
    dbt makes transformation feel "free" during development → teams write very expensive patterns (window functions over billions of rows, unpruned joins, full scans in loops) that only become obvious on the monthly cloud bill.

These show up over and over again in production environments (Reddit threads, blog post mortems, LinkedIn rants, Slack horror stories). The pattern is almost always the same: things look fine with 5–10 models, then explode at 100–300+ models when real data volumes, team size, and stakeholder pressure hit.

