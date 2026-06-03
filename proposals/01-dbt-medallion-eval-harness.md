# Where Coding Agents Break at Data Engineering: A dbt Failure Taxonomy and Correctness Loop

**One-liner:** A characterization of *where coding agents actually go wrong* when they write transformations over data (the gap between "compiles" and "correct" in dbt medallion pipelines), plus an agent+judge loop that turns that map into reliably correct pipelines, not just a ranked list.
**Area:** ML / Infra   •   **Difficulty:** Intermediate–Advanced

## Motivation
Coding agents get pointed at data-engineering tasks all the time, and they almost always produce something that builds. The interesting and unanswered question is *where* they break: a medallion (bronze/silver/gold) pipeline has correctness surfaces compilation never touches (layering discipline, incremental logic, test coverage, contracts, source freshness), and an agent can produce models that build cleanly and are still semantically wrong. We want to map those failure modes into a **taxonomy of correct-vs-merely-compiles for agents writing code over data**: which classes of error are common, which are dangerous, which an agent can self-correct given the right signal. That map is the transferable output: the same failure structure shows up wherever an agent writes code that has to be *right about data*, not just syntactically valid. The eval harness is how we earn the map and then close the loop: feeding the right signals back so an agent converges on a correct pipeline. The "so what": a reusable diagnosis of agent data-engineering competence, and a demonstrated loop that produces correct pipelines rather than a leaderboard that merely sorts them.

## Research Question(s)
1. What is the taxonomy of failure when an agent builds a dbt medallion pipeline, which error classes (silent layering violations, broken incremental keys, under-tested marts, contract gaps) recur, and which separate "compiles" from "correct"? Does that taxonomy generalize beyond dbt to other "agent writes code over data" settings?
2. Which of those failures can an agent *fix on its own* when the harness feeds back the right diagnostic signal, so the loop yields a correct pipeline instead of just a score?
3. For the soft failures deterministic checks can't see (layering sense, idiomatic dbt), where can an LLM-as-judge be trusted as a reliable diagnostic signal, and where does it mislead?

## Scope
- In scope:
  - A task runner that hands an agent a sandboxed repo (raw seed/source data, a natural-language spec, empty `models/`) and collects the resulting dbt project.
  - A pluggable agent-driver interface. Pi (pi.dev), a multi-model terminal agent with a print/JSON mode and SDK, is the primary driver. The Claude Agent SDK or Claude Code headless mode can serve as a secondary adapter if convenient. The interface exists so arbitrary models plug in behind a small contract.
  - A grader that runs against DuckDB via `dbt-duckdb` (cheap, local) and emits a structured diagnosis: which failure classes fired, on which models.
  - Deterministic checks that surface failure classes: `dbt build` passes, tests pass, gold output row-diff vs. a reference (via `dbt-audit-helper`), incremental models behave correctly on a second run, DAG/layering structure checks.
  - A correctness loop: the grader's diagnosis is fed back to the agent so it can attempt a fix, letting us measure which failure classes are self-correctable and produce a correct pipeline as the actual output.
  - Judged checks via an LLM-as-judge on idiomatic-ness and sensible layering, with judge output cross-checked against the deterministic signals so we know how much to trust it as a diagnostic signal.
  - A handful of tasks across a couple of difficulty tiers, each with raw data, spec, and reference gold output. This is the substrate the taxonomy is built from.
- Out of scope:
  - Building a strong agent. Agents are inputs to the harness, not a deliverable.
  - Cloud warehouses (Snowflake/BigQuery/Databricks) beyond a documented adapter stub. DuckDB is the reference engine.
  - Streaming/real-time, orchestration tools (Airflow/Dagster), and non-dbt transformation frameworks.

## Suggested Approach
1. Define the medallion rubric before writing any code; this is the skeleton the failure taxonomy hangs on. Per layer, what does "good" mean?
   - Bronze/staging: one model per source, light cleanup only (rename, cast, basic typing), no joins, `source()`-based, sources declared with freshness.
   - Silver/intermediate: dedup (window functions), joins, business entities; no raw `source()` refs; uses `ref()`.
   - Gold/marts: fact/dim tables, aggregations, business logic; consumed by exposures; contracts enforced.
2. Pick the grading split.
   - Deterministic (machine-checkable): `dbt build` exit code; `dbt test` pass rate (generic tests `unique`, `not_null`, `accepted_values`, `relationships`, plus any singular tests); `dbt source freshness`; gold row-level diff vs reference using `audit_helper.compare_relations` (% identical rows); an incremental re-run check (run once, append new rows to source, run again, assert only new/changed rows landed and counts match expectation); DAG-shape checks by parsing `target/manifest.json` (staging refs only sources, marts don't ref sources directly, no cycles).
   - Judged (soft): idiomatic naming/layering, sensible materializations, test-coverage adequacy. An LLM-as-judge scores each dimension with a rubric prompt, and we look at how its scores line up with the deterministic signals to gauge how far to trust it.
3. Agent-driver interface: a thin `AgentDriver.run(task_dir, spec) -> project_dir`. Pi (pi.dev) is the driver to build first: its print/JSON mode lets the harness invoke it non-interactively and pin a backbone model per run. A second adapter (Claude Agent SDK via `pip install claude-agent-sdk`, or Claude Code headless) validates that the contract generalizes. Keep the contract small so other models plug in.
4. Sandbox and reproducibility: each task runs in an isolated dir (or container) with pinned dbt and adapter versions. Capture the agent transcript, the generated project, and the full report card as artifacts so a run can be inspected and rerun.
5. Closing the loop: when the grader flags a failure class, hand the diagnosis back to the agent and let it retry, so we learn which classes are self-correctable and the run ends on a *correct* pipeline. A hand-written reference sets the ceiling; running a deliberately weak agent (small/cheap model or stripped-down prompt) against a strong one mostly serves to populate the taxonomy with varied failure modes, not to crown a winner.

## Deliverables
- **The headline: a failure taxonomy** of where coding agents break on dbt medallion work: the recurring error classes, which separate "compiles" from "correct," which the agent can self-correct, with a discussion of how the taxonomy transfers to other "agent writes code over data" settings.
- A correctness loop that, given a flagged failure, feeds diagnosis back to the agent and converges on a *correct* pipeline where possible, the demonstrated capability, not just a measurement.
- A usable `dbt-agent-eval` repo as the means to both: task runner, the Pi driver plus at least one more adapter, grader, and a CLI (`eval run --task X --agent Y`).
- A set of tasks (raw data, spec, reference gold, per-task expected-check config) across a couple of difficulty tiers, and a structured diagnosis schema (JSON) per run.
- A short write-up centered on the taxonomy and the self-correction findings; how judge scores tracked the deterministic signals; and, secondarily, how agents compared.

## Demo
One command, `eval run --task all --agent pi`, spins up sandboxes, drives the agent to build each pipeline, runs the grader against DuckDB, and, when a failure class fires, feeds the diagnosis back so the agent can fix it. The payoff slide is the failure taxonomy: which error classes showed up, which were self-correctable in the loop, and a before/after where a flagged pipeline gets driven to correct.

## Success Criteria
- A defensible taxonomy comes out: a set of failure classes grounded in real runs, each tied to a check that detects it, with a clear split between compile-level and correctness-level failures and a note on which generalize past dbt.
- The loop demonstrably fixes at least some flagged failures end-to-end: show a pipeline that fails a check, gets the diagnosis fed back, and reaches correct, and characterize which classes resist self-correction.
- The checks actually detect the failures they claim to: build a fault-injection set of known-bad variants (e.g. a staging model with a join, a broken incremental key, a mart that refs a source directly), show the checks pass the reference and catch the injected bugs, and report how well they discriminate rather than chasing a fixed catch rate.
- An honest read on the judge as a diagnostic signal: show how its scores on the soft dimensions track the deterministic signals, account for judge noise, and call out where it disagrees rather than averaging it away.
- The harness runs end-to-end on more than one agent driver with pinned versions and captured artifacts, and gives repeatable results when rerun, so the taxonomy reflects the agents, not luck.

## Tech & Prerequisites
- Python 3.10+ for the harness, drivers, and glue. SQL and core dbt concepts: sources, `ref`/`source`, materializations, incremental, tests, contracts, `manifest.json`.
- `dbt-core`, `dbt-duckdb`, `dbt-audit-helper`. Agent driver: Pi (pi.dev), with `claude-agent-sdk` or Claude Code as a secondary. An API key for the agent backbones and the judge.
- Useful background: medallion architecture, Kimball fact/dim modeling, and execution-based eval (SWE-bench: sandbox, run real tests, pass/fail).

## Phases
- Explore: build one reference dbt medallion project by hand on DuckDB. Write the rubric. Drive Pi (and a second adapter) on toy tasks to learn each one.
- Build: task runner and sandbox, `AgentDriver` interface with the Pi adapter plus one more, deterministic grader (build/test/freshness/diff/incremental/DAG), then the judge layer.
- Evaluate: author the tasks with references, run fault-injection to confirm the checks catch known bugs, run the agents, then mine the runs into the failure taxonomy, measure which classes the loop self-corrects, and do the judge-vs-deterministic comparison.
- Polish: CLI, report-card formatting, README, write-up, and a documented (stubbed) cloud-warehouse adapter path.

## Stretch Goals
- A snapshot / SCD-type-2 task; model contracts enforced as a graded dimension.
- Auto-generate task variants (perturb specs/data) to grow the benchmark.
- Cost/latency/token accounting per agent; partial-credit scoring instead of pass/fail.
- A cloud adapter (Snowflake or BigQuery) actually wired up.

## Starter References
- dbt incremental models & strategies (merge/append/delete+insert/microbatch): https://docs.getdbt.com/docs/build/incremental-strategy
- dbt data tests (generic + singular) and model contracts: https://docs.getdbt.com/docs/build/data-tests and https://docs.getdbt.com/docs/mesh/govern/model-contracts
- `dbt-audit-helper` for row/column-level reference diffing: https://github.com/dbt-labs/dbt-audit-helper
- Pi coding agent (multi-model terminal agent; print/JSON + SDK for scripting): https://pi.dev/
- Claude Agent SDK overview (programmatic Claude Code harness): https://code.claude.com/docs/en/agent-sdk/overview
- SWE-bench execution-based eval methodology (sandbox + real tests): https://www.swebench.com/ and the paper https://arxiv.org/abs/2310.06770

## Open Questions / Assumptions
- Driver and engine are settled: Pi (pi.dev) is the agent driver, invoked through its print/JSON mode with the backbone model pinned per run, and DuckDB via `dbt-duckdb` is the warehouse engine. The Claude Agent SDK or Claude Code can be a secondary adapter if convenient.
- Reference-output brittleness: gold diffs assume deterministic transforms. Tasks must avoid nondeterminism (timestamps, ordering, floating-point aggregation), or the diff check needs tolerance rules. Assumption: tasks are authored to be deterministic.
- DuckDB vs warehouse fidelity: some incremental strategies (`insert_overwrite`, `microbatch` partitioning) behave differently or aren't supported on `dbt-duckdb`. The incremental check may have to be scoped to `merge`/`delete+insert`/`append`. Confirm adapter support before authoring incremental tasks.
- Judge stability: LLM-as-judge variance is well documented. Run judges with fixed prompts where possible and report the noise; treating the judge as ground truth is not justified, which is partly why research question 3 exists.
