# A Self-Improving NL-to-SQL System for Complex Schemas

**One-liner:** Build an NL-to-SQL system that *measurably improves itself* on hard, enterprise-style schemas by automatically rewriting its own prompt from execution feedback, and pin down *where* the accuracy gain comes from (schema linking vs. dialect handling vs. example selection), an insight that transfers to NL-to-SQL broadly.

**Area:** Research / ML   •   **Difficulty:** Intermediate–Advanced

## Motivation
For NL-to-SQL, a lot of the accuracy lives in the prompt rather than the model: how the schema is serialized, which examples are shown, how the SQL dialect and error-recovery behavior are described. On real enterprise schemas with many tables, ambiguous column names, and undocumented business logic, even strong LLMs lose a large fraction of the accuracy they show on academic benchmarks. (One often-cited example is DAIL-SQL with GPT-4o dropping from the mid-80s on Spider 1.0 to single digits on Spider 2.0; treat the exact figures as illustrative until verified against the Spider 2.0 leaderboard.) Today that prompt is tuned by hand, one schema at a time.

The capability this project builds is an NL-to-SQL system that closes that gap *on its own*: it runs queries, reads the failures, and rewrites its instructions to do better next time, getting measurably more accurate on complex schemas without anyone hand-editing the prompt. We drive that loop with automatic prompt optimization. GEPA (Genetic-Pareto reflective prompt evolution) is our primary engine because it reflects on execution traces in *natural language* (a good fit for the rich textual feedback SQL execution produces), but it is one option among several (e.g. DSPy's MIPROv2, TextGrad), and the project is designed around the capability and the resulting insight, not around any single optimizer.

The generalizable payoff is the insight, not just the score: by ablating the optimized prompts we learn *which* lever actually moves NL-to-SQL accuracy on hard schemas: schema linking, dialect handling, error recovery, or example choice. That answer is useful to anyone building NL-to-SQL, whatever optimizer (or hand-tuning) they use.

## Research Question(s)
1. **Capability:** Can an NL-to-SQL system that rewrites its own prompt from execution feedback achieve measurably higher held-out execution accuracy on complex/enterprise schemas than a strong hand-written or few-shot system, under a fair comparison (similar context budget, same eval harness)?
2. **Insight (the transferable part):** *Where* does the improvement come from: schema linking, dialect handling, error-recovery instructions, or example selection? Which lever generalizes across schemas and which is schema-specific?
3. **Robustness of the insight:** Does the finding depend on the optimizer (GEPA vs. an alternative like MIPROv2/TextGrad) and on the generator LLM, or do the same levers dominate regardless?

## Scope
- **In scope:** A real, runnable, model-agnostic NL-to-SQL system (prompt → LLM → SQL → execute → grade) that someone can point at a complex schema and get better answers as it self-optimizes; an execution-accuracy metric that also returns textual failure feedback; a self-improvement loop driven by an automatic prompt optimizer (GEPA primary, with at least one alternative such as MIPROv2 to check the optimizer isn't the story); baselines (hand-written, few-shot); evaluation on a held-out split with an ablation that locates *where* the gain comes from.
- **Out of scope:** Fine-tuning model weights; building a new benchmark; agentic multi-step SQL workflows (Spider 2.0-style toolchains); production deployment or polished UI; topping a public leaderboard.

## Suggested Approach
1. **Work over genuinely complex schemas.** Use BIRD (real databases, external "evidence" knowledge, execution-accuracy metric), BEAVER (enterprise text-to-SQL over real data-warehouse schemas), and TPC-DS (the standard decision-support star/snowflake schema and query workload), picked for coverage of large, enterprise-style schemas, not leaderboard-chasing. Start from a small subset of each so iteration stays cheap, and expand once the system runs end to end.
2. **Build the generator as a DSPy module** (a `dspy.Predict` or `ChainOfThought` over a `question, schema, evidence -> sql` signature). Schema serialization and instructions live in the signature/system prompt; that text is what the optimizer rewrites, which keeps the system optimizer-agnostic.
3. **Write the metric/feedback function** that closes the self-improvement loop. Score is execution accuracy: does the predicted SQL return the same row multiset as gold on the real DB. Feedback is the *reason* for failure in plain text: syntax error, empty result, wrong column or join, dialect mismatch. This textual feedback is the signal the system learns from. It carries far more than a scalar reward, and it's exactly what reflective optimizers exploit. (For GEPA this is the `metric(gold, pred, trace, pred_name, pred_trace) -> {'score': float, 'feedback': str}` signature.)
4. **Run the self-improvement loop with GEPA as the primary optimizer** (`reflection_lm` a strong model, `auto='light'` first to control cost, `candidate_selection_strategy='pareto'`). Keep train/validation small at first, grow as budget allows. Then **re-run the loop with at least one alternative optimizer** (e.g. `dspy.MIPROv2`, or TextGrad-style textual-gradient optimization) at a comparable budget, not to crown a winner, but to confirm the capability and the "where it comes from" insight hold across methods rather than being an artifact of one optimizer.
5. **Baselines, eval, and the ablation that delivers the insight:** a hand-written prompt and a few-shot baseline, kept comparable in context budget so you're testing prompt *content*, not just "more context wins." Evaluate every condition on a frozen held-out split with the same execution harness. Then ablate the optimized prompt (strip or swap its schema-linking, dialect, error-recovery, and example-selection content) to attribute the gain to specific levers. Hold the generator LLM fixed across conditions, then repeat with a second LLM to see whether the same levers dominate.

## Deliverables
- A runnable, self-improving NL-to-SQL system: data/schema loader, DSPy module, execution-accuracy + feedback metric, the self-improvement loop (GEPA plus a pluggable alternative optimizer), baseline scripts, eval harness, usable on a fresh complex schema, not just the benchmark splits.
- The optimized, self-rewritten system prompt(s), saved and human-readable.
- The headline insight, written up: an attribution of *where* the accuracy gain comes from (schema linking vs. dialect vs. error recovery vs. examples), backed by the ablation, plus an execution-accuracy comparison (baselines vs. self-improved system, across at least a second generator LLM and a second optimizer), an error-category breakdown, and rollout/API-cost counts per condition.

## Demo
One command (`python evaluate.py --split held_out`) that runs all conditions and prints a comparison table of execution accuracy, plus a notebook cell that shows the self-optimized prompt next to the hand-written one with a 2–3 sentence note on *what the system changed and which lever made the difference*.

## Success Criteria
- **Done:** a working self-improving NL-to-SQL system with execution-based grading on a held-out complex schema; the optimization loop (GEPA + at least one alternative) and the baselines run end-to-end; comparison, ablation, and error analysis produced; runs reproducible from a fixed config.
- **Good:** A demonstrated capability: the system measurably improves its own held-out execution accuracy on hard schemas over the best baseline under a fair setup, with enough sense of the noise that a tiny "win" isn't mistaken for a real one, *and* a defensible insight: the ablation attributes the gain to specific levers, the attribution is shown to hold (or not) across a second LLM and a second optimizer. A carefully measured null/negative on the capability still counts as Good if the *where-the-accuracy-lives* analysis is solid, since that insight transfers regardless.

## Tech & Prerequisites
Python; DSPy (`dspy.GEPA`, `dspy.MIPROv2`); SQLite/PostgreSQL to execute queries; an LLM API (or local model) for generation plus a strong reflection model. Background: SQL and joins, basic prompting, and comfort designing experiments and reading evaluation metrics. No RL or fine-tuning background needed.

## Phases
- **Explore:** Read on the optimizers (GEPA paper + DSPy GEPA tutorial, MIPROv2/TextGrad for the alternative); load BIRD, BEAVER, and TPC-DS; get a few DBs executing; measure a hand-written-prompt accuracy to anchor the rest.
- **Build:** DSPy module, execution-accuracy metric with textual feedback, the self-improvement loop (GEPA primary + a pluggable alternative optimizer), baselines.
- **Evaluate:** Frozen held-out split; self-improved system vs. baselines across at least a second generator LLM and a second optimizer; the ablation that attributes the gain to specific levers; error analysis with a sense of the noise band.
- **Polish:** One-command demo, saved prompts, written report leading with the where-the-accuracy-lives insight and the comparison table.

## Stretch Goals
- Transfer test: self-optimize on one DB family, evaluate on unseen schemas: does the capability generalize, or re-learn per schema?
- Add automatic few-shot example selection and let the loop co-optimize it.
- Optimizer frontier: GEPA vs. MIPROv2 (and/or TextGrad) on the rollout-budget vs. accuracy curve, to see if the cheaper path to the same insight differs.
- A small Spider 2.0 slice to probe limits on genuinely hard schemas.
- Ablate the schema-serialization strategies the optimized prompt converges on.

## Starter References
- GEPA paper: *GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning* (Agrawal et al., 2025): https://arxiv.org/abs/2507.19457
- `dspy.GEPA` API + tutorial: https://dspy.ai/api/optimizers/GEPA/overview/ and https://dspy.ai/tutorials/gepa_ai_program/
- GEPA reference implementation (standalone library): https://github.com/gepa-ai/gepa
- Alternative optimizers (for the optimizer-agnostic check): DSPy MIPROv2 (https://dspy.ai/api/optimizers/MIPROv2/) and TextGrad: *TextGrad: Automatic "Differentiation" via Text* (Yuksekgonul et al., 2024): https://arxiv.org/abs/2406.07496
- BIRD benchmark (real-DB text-to-SQL, EX + VES metrics): https://arxiv.org/abs/2305.03111 and https://bird-bench.github.io/
- BEAVER (enterprise text-to-SQL over real data-warehouse schemas, where LLMs drop sharply): https://arxiv.org/abs/2409.02038 and https://beaverbench.github.io/
- TPC-DS (standard decision-support benchmark; complex star/snowflake schema and query workload): https://www.tpc.org/tpcds/ and spec PDF https://www.tpc.org/TPC_Documents_Current_Versions/pdf/TPC-DS_v4.0.0.pdf
- Spider 2.0 (enterprise text-to-SQL, harder/agentic): https://github.com/xlang-ai/Spider2

## Open Questions / Assumptions
- **Benchmarks are settled:** BIRD, BEAVER, and TPC-DS, chosen for coverage of complex/enterprise schemas. TPC-DS ships a schema and query generator rather than NL questions, so part of the work is producing natural-language questions paired with gold SQL over it (or reusing existing question sets); decide how much of TPC-DS to cover early.
- **Cost/budget:** GEPA plus a strong reflection LM plus repeated SQL execution over real DBs has non-trivial API and compute cost. Reviewer should set an LLM budget and confirm which generator models are allowed before the Build phase.
- **The capability is the open empirical question:** automatic prompt optimizers report gains on general tasks, not NL-to-SQL over hard schemas. Whether a self-improvement loop actually lifts held-out accuracy here is what the project demonstrates; let the hand-written baseline set expectations rather than committing to a target up front. The headline insight (which lever drives the gain) is designed to be valuable even if the lift is modest.
- **Optimizer is a means, not the point:** GEPA is the primary engine, but the project deliberately runs a second optimizer (MIPROv2/TextGrad) so the capability and the where-it-comes-from insight aren't pinned to one method or one preprint. Reviewer can swap which alternative is used.
- **Overfitting risk:** With small train sets, GEPA can overfit the prompt to the questions/DBs it saw. This is why the held-out split should use schemas GEPA never saw: hold out whole schemas, not just questions.
- **Metric false positives:** Row-multiset matching can mark a wrong query "correct" when it coincidentally returns the same rows (or both return empty). Spot-check a sample of "correct" predictions by hand.
- **Fair comparison:** GEPA's prompt is usually longer than the hand-written one, so a raw comparison partly measures context budget. Keep the few-shot baseline comparable in budget so the comparison reflects GEPA's content, not just more context.
- **VES (efficiency):** Out of scope by default. Flag if the reviewer wants query efficiency measured alongside accuracy.
