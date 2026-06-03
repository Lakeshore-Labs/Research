# Introspective Self-Improvement: Agents That Read Their Own Eval Report Card

**One-liner:** Give an LLM agent first-class, queryable access to its own evaluation results (failing cases, per-dimension scores, and execution traces) and test whether self-diagnosis drives better, faster, and more interpretable improvement than a black-box optimizer that only sees a scalar score.

**Area:** Research (with ML/SW build)   •   **Difficulty:** Advanced

## The Insight We're After
**When does introspective self-access beat black-box optimization for self-improving agents?** Self-improvement loops face a design fork: let an external optimizer tune the model from the outside against a scalar reward, or let the model read its own failures and reason about why it failed. The generalizable finding this project chases is the *condition* that decides which wins: when transparent self-access (failures, traces, structured scores) is more sample-efficient and more interpretable than scalar-only feedback, and when it isn't. That answer is a design principle for building self-improving agents in general, not a number attached to one task. The single task domain here is just the cleanest place to isolate and observe the effect.

## Motivation
Most automatic improvement loops treat the model as a black box to be tuned. An external optimizer watches a scalar reward and mutates prompts or weights. But a capable LLM can in principle read its own failures and reason about why it failed, given the data. The question is whether transparent self-access produces improvement that is more sample-efficient and more interpretable than scalar-only feedback, and that is what generalizes beyond any single domain.

There is a known catch. Models are unreliable graders of their own work and often degrade when they self-correct without an external signal ([Huang et al. 2023](https://arxiv.org/abs/2310.01798)). This study sidesteps the grading problem on purpose: the agent never scores itself. It receives ground-truth eval results computed by a deterministic harness, and its job is to diagnose and fix, not to judge. Answering the question cleanly tells us when introspection beats black-box optimization, which directly informs how we build self-improving agents.

## Research Question(s)
1. **Primary:** Does giving an agent first-class access to its own eval results (failing examples, execution traces, per-dimension scores) let it self-improve more effectively than (a) no feedback and (b) a black-box optimizer that sees only a scalar score?
2. Which representation of eval data (raw traces vs. aggregated "report card" vs. clustered failure modes) most helps the model self-diagnose?
3. Does self-driven improvement generalize to held-out data, or does it overfit / game the eval relative to an external optimizer?

## Scope
- **In scope:**
  - One runnable task domain with an automatic, deterministic metric (a coding or reasoning task; see below).
  - A harness that runs the agent's eval, exposes results to the agent as context and/or tools, lets the agent propose a revised prompt/strategy, and re-tests on train + held-out data.
  - The four-arm comparison (no-feedback, scalar-only, an external optimizer (GEPA), full self-access) as the means to isolate *when* self-access wins, not a benchmark for its own sake.
  - A meta-eval covering improvement over iterations, generalization, compute/sample cost, and interpretability of self-diagnoses.
- **Out of scope:**
  - Weight fine-tuning / RL training. This is a prompt- and strategy-level study; STaR / Self-Rewarding-style fine-tuning is a stretch goal only.
  - Building a new benchmark from scratch. Reuse an existing one.
  - Multi-domain generalization. Pick one domain and go deep.

## Suggested Approach
The goal here is an **actual self-improvement loop someone can run and watch get better**, not a metrics-only study. Build the loop first, make it observable, then layer on the comparisons.

**Domain.** Pick one domain with an automatic, deterministic metric and don't agonize over it. A coding task like [MBPP](https://github.com/google-research/google-research/tree/master/mbpp) (pass/fail from provided unit tests, with rich execution traces to diagnose) is a natural fit; a reasoning task like GSM8K works too if you'd rather avoid a code sandbox. Either gives clean train/held-out partitions and ground-truth scoring. Sandboxing has many off-the-shelf options. Don't let it block the project; if it's awkward, switch to a reasoning task and the rest of the design carries over unchanged.

**Hold out some data.** Reserve a portion of the data for final scoring that the improvement loop never touches. The principle: the agent improves against training data, and you measure real progress on data it hasn't seen, so you can tell genuine improvement from overfitting. Keep held-out results out of the agent's working context so it can't tune to them directly.

**Operationalize "access to its own evals" concretely.** Build a harness (DSPy or LangGraph are reasonable; plain Python is fine) that gives the agent:
- a `run_eval()` tool that scores the current prompt/strategy on the training data and returns aggregate + per-test scores.
- a `get_failures(k)` tool returning the k worst cases with input, model output, gold answer, and the execution trace (generated code, failing test, error).
- a structured report card: pass rate, score per dimension, and clustered failure modes (e.g. "wrong return type", "off-by-one", "ignored constraint").

The agent never grades itself; it always receives ground-truth eval results from the harness, then writes a natural-language self-diagnosis, proposes a revised prompt/strategy, and the harness re-tests. Loop for several iterations, logging an improvement curve on both training and held-out data. Eval calls aren't free, so it's worth tracking them as part of cost rather than treating "just re-run the eval" as a no-op.

**Baselines / arms (how we earn the insight):** the four arms are a contrast designed so the only thing that varies is *how much the agent sees about its own evals*: from nothing, to a scalar, to an external optimizer working the scalar, to full introspective access. The gap between (b)/(c) and (d) is the finding.
- **(a) No feedback:** fixed prompt, no loop. Lower bound.
- **(b) Scalar-only:** agent sees only the scalar training score per iteration and must guess revisions. No traces, no failures.
- **(c) External optimizer (GEPA):** a reflective prompt optimizer that tunes from the outside against the score ([Agrawal et al. 2025](https://arxiv.org/abs/2507.19457), via [DSPy](https://dspy.ai/api/optimizers/GEPA/overview/)). GEPA is one representative external optimizer; DSPy MIPRO or TextGrad could stand in. It's the black-box baseline to match or beat, not the method under test; the project shouldn't hinge on GEPA specifically.
- **(d) Full self-access:** failures + traces + report card, agent-driven. The proposed method.

Keep the arms comparable: same data split, same base model, and a roughly matched compute/rollout budget, so the comparison is honest rather than confounded by one arm simply getting more tries. Optionally add **(e) self-access + GEPA** as a hybrid arm.

**Methods to read first:** Self-Refine (iterative self-feedback), Reflexion (reflect on failures held in episodic memory), and the self-evaluation-reliability literature. TextGrad (textual "backprop" through traces) and DSPy reflective optimizers are useful design references.

## Deliverables
- A runnable harness implementing all four arms, with held-out data kept out of the agent's loop.
- Improvement curves (train + held-out) per arm, plotted against both iteration count and rollout cost.
- A short written analysis answering the headline question: *under what conditions does self-access beat black-box optimization?* Does it beat scalar-only? Does it match or beat the external optimizer (GEPA)? What does the train/held-out gap say about generalization vs. eval-hacking? State the takeaway as a design principle, not just a score.
- A qualitative appendix of the agent's self-diagnoses, each rated for accuracy (did the stated cause match the actual failure cause?).
- Reproducible config and a README to re-run any arm.

## Demo
Start the agent with a deliberately weak prompt. It calls `run_eval()`, pulls its worst failures and traces, prints a self-diagnosis ("I am dropping the unit conversion step"), proposes a revised prompt, re-tests, and repeats. Show the held-out improvement curve climbing over iterations, on the same axes as the scalar-only and GEPA arms, something a reviewer can run and literally watch improve.

## Success Criteria
- **Done:** all four arms run end-to-end with held-out evaluation and reproducible logs, and the self-improvement loop is observable iteration by iteration.
- **Good:**
  - Self-access (d) shows clear held-out improvement over iterations and beats scalar-only (b).
  - Self-access matches or beats the external optimizer (c) on held-out score, or reaches a comparable score with less compute, and you can say *why*, turning the result into a transferable rule of thumb about when to prefer introspection over black-box tuning.
  - **No eval-hacking:** the metric that matters is held-out score, not train score. Watch for the model gaming its own eval, e.g. train climbing while held-out stalls or drops. A sanity re-score on a fresh, never-touched slice is a good final check.
  - Self-diagnoses are interpretable, and diagnosis accuracy tends to track whether the proposed fix actually helped on held-out.

## Tech & Prerequisites
- Python; an LLM API (prompt caching helps with cost); a way to run the chosen task's metric (a code sandbox for a coding task, or a string/numeric check for a reasoning task); plotting.
- Familiarity with at least one agent/optimizer framework (DSPy recommended for the GEPA baseline; LangGraph optional).
- Background: prompting, basic experiment design (train/held-out splits), and reading ML papers.

## Phases
- **Explore:** Read Self-Refine, Reflexion, Huang et al. (self-correct limits), GEPA. Pick the domain and split. Define the metric and the report-card representation.
- **Build:** Get the self-improvement loop working and watchable first, then implement the four arms; keep held-out data out of the loop; wire `run_eval` / `get_failures` / report-card tools.
- **Evaluate:** Run all arms; produce improvement curves (vs. iterations and vs. cost); check for eval-hacking; rate diagnosis accuracy.
- **Polish:** Write up findings, clean the repo, record the demo.

## Stretch Goals
- Add the hybrid arm (self-access feeding GEPA) and measure whether introspection improves the external optimizer.
- Ablate the eval representation (raw traces vs. report card vs. clustered failure modes), addressing RQ2.
- Promote winning self-diagnoses into persistent memory (Reflexion-style) and reuse across runs.
- A STaR / Self-Rewarding-style fine-tuning extension if a tunable open model is available.

## Starter References
- Self-Refine: Iterative Refinement with Self-Feedback. https://arxiv.org/abs/2303.17651 ([repo](https://github.com/madaan/self-refine))
- Reflexion: Language Agents with Verbal Reinforcement Learning. https://arxiv.org/abs/2303.11366
- Self-Rewarding Language Models (Meta). https://arxiv.org/abs/2401.10020
- Large Language Models Cannot Self-Correct Reasoning Yet (the key caveat). https://arxiv.org/abs/2310.01798
- GEPA: Reflective Prompt Evolution Can Outperform RL (the external-optimizer baseline). https://arxiv.org/abs/2507.19457
- Also useful: STaR, https://arxiv.org/abs/2203.14465 ; Constitutional AI (self-critique), https://arxiv.org/abs/2212.08073

## Open Questions / Assumptions
- **The point is a usable loop, not a leaderboard.** Aim first for a self-improvement loop someone can start, watch climb, and inspect. The four-arm comparison answers the research question, but a working, observable loop is the primary artifact.
- **Domain is flexible.** A coding task (e.g. MBPP) or a reasoning task (e.g. GSM8K) both work; pick whichever is easier to stand up. Sandboxing has many options and shouldn't block the project; if it's awkward, use a reasoning task instead.
- **Self-evaluation reliability is the central risk, and the design avoids it.** Huang et al. show models often degrade when self-correcting without an external signal. The principle here: the agent receives ground-truth eval results and never grades itself. This holds cleanly for a unit-test or numeric-answer metric. If a subjective rubric dimension is added later, keep an independent judge or human spot-checks for it.
- **Hold out data and watch for gaming.** Keep held-out data out of the improvement loop and treat held-out score as the real signal. Train climbing while held-out stalls is the tell-tale sign of overfitting/eval-hacking. These are principles to design around, not a fixed rulebook of caps and seeds.
- **Relationship to the GEPA proposals.** Three sibling proposals (NL-to-SQL, skill enhancement, Ralph+GEPA pipeline) use GEPA as the method. This one is deliberately different: the thesis is model-driven introspection on its own evals, and GEPA appears only as the external-optimizer baseline (arm c), optionally as a hybrid (arm e), so the four stay distinct rather than redundant.
- **TextGrad** ([arXiv 2406.07496](https://arxiv.org/abs/2406.07496)) is a relevant "textual backprop through traces" comparison, included as a design reference. Worth a look as a possible additional baseline.
- **Compute budget** for multi-arm runs with a capable model is open. A tight budget may push toward a smaller model or fewer runs; keep arms roughly matched on compute so the comparison stays honest.
- Several 2025–2026 "self-evolving agent" blog/preprint hits (trace-driven improvement loops) align with this thesis but were not traced to primary sources; not cited as facts.
