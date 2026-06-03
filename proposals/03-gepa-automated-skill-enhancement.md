# A System That Automatically Authors and Improves Agent Skills

**One-liner:** Build a system that automatically authors and improves agent "skills" (instructions, scripts, references, and all), handing back a better skill artifact you can drop in and use, and along the way learn *what actually makes a skill effective* (which instructions and structure matter), an insight transferable to skill and playbook design broadly.

**Area:** Research / ML   •   **Difficulty:** Intermediate–Advanced

## Motivation
Agent Skills (a `SKILL.md` instruction module plus optional scripts, the open format used by Claude Code and the Claude Agent SDK) are now a primary way to steer agents, but they're hand-authored and tuned by trial and error. There's no closed loop and no theory of what makes one skill better than another. Authors guess at structure, phrasing, and which scripts to bundle. This project builds the missing capability: a system that takes a skill, runs it, learns from where it fails, and rewrites it into something that works better, and then reads the diffs to extract a generalizable account of which edits actually moved the needle.

The optimization engine is interchangeable. GEPA (reflective prompt evolution) is the headline candidate: its paper shows an LLM reflecting in natural language on its own traces can evolve text artifacts and beat RL-style tuning at a fraction of the rollouts, and the repo lists "coding agent skills" as a use case (a 55%→82% resolve-rate claim; see Open Questions, the repo's own number, not independently verified). But the capability we're after, *automatically author a better skill, and understand why it's better*, is the goal; GEPA is one means to it among several.

## Research Question(s)
1. Can a system automatically rewrite a skill so an agent using it succeeds more often on held-out tasks than with the original human-authored skill, i.e., does the authoring capability deliver a usable, improved artifact?
2. **What makes a skill effective?** Across optimized skills, which kinds of edits recur and carry the gains: explicit step ordering, validation checks, worked examples, tighter descriptions, fixed scripts? Can we distill transferable skill-authoring guidance from the diffs?
3. Can the system improve efficiency (fewer tool calls, lower token cost) without hurting success, surfacing useful success/cost trade-offs rather than a single point?
4. Does the optimized skill generalize to unseen task variants, or does it overfit and reward-hack the training tasks?

## Scope
- In scope:
  - A **skill-authoring capability**: feed it a skill plus its tasks, get back an improved skill an agent can load and use directly. The deliverable is a usable system plus a findings report on what makes skills work, not a metrics dashboard.
  - One skill end-to-end is the real target (e.g. a code-task or document-processing skill) with a set of tasks it's meant to cover. A second skill is a stretch goal, not a commitment.
  - The optimizable artifact is the **whole skill** (the `SKILL.md` body, its `description`, and its bundled scripts and reference files), not just the instruction prose.
  - A model-agnostic harness: run an agent with the skill loaded, capture the trace, score it, and produce textual feedback on failures.
  - A **pluggable optimizer interface**, with GEPA as the primary backend and at least one alternative (see Suggested Approach) so the capability doesn't hinge on a single optimizer.
  - The evolve-then-evaluate loop; compare optimized vs. original on held-out tasks; and an analysis pass over the diffs to extract what changed and which edits carried the gains.
- Out of scope:
  - NL-to-SQL prompt optimization and introspection-style work (covered by sibling GEPA proposals #2 and #7; do not duplicate). This proposal is specifically about the *skill-artifact authoring capability*, producing a better drop-in skill, not query generation or agent self-inspection.
  - Inventing a new optimizer or modifying GEPA internals.
  - Multi-skill orchestration or training a model. This optimizes skill text and bundled files only, not weights.

## Suggested Approach
A starting path, not prescriptive:

1. **Pick a seed with headroom, and prove it.** This is a method step, not a caveat. Choose seed skills that are plausible but have *visible failure modes* on the task set. Then, before any optimization, run the seed and report its own baseline held-out performance and a short list of where and why it fails. A seed only qualifies once you've shown it genuinely fails on a non-trivial slice of tasks (say, leaves real headroom rather than already passing nearly everything). This guarantees there is something to improve and makes the seed's starting point an explicit, reported number rather than an assumption. If a candidate seed turns out to be near-perfect on the tasks, swap it or harden the tasks until measurable headroom exists.
2. Frame the artifact. The optimizer evolves the full skill: the `SKILL.md` body, its `description` (which drives discovery), and any bundled scripts or reference files. The human-authored seed is the starting candidate.
3. Pick tasks and split. Choose a set of tasks the skill should help with (enough to learn from and still hold some out), split into train / validation / held-out test. Prefer programmatically checkable tasks (file produced, tests pass, expected fields present) so scoring is objective. Small held-out sets are noisy, so favor as many test tasks as the rollout budget allows and don't read too much into small swings.
4. Build the agent rollout. Use the Claude Agent SDK to run an agent with the skill loaded from `.claude/skills/<name>/` (`setting_sources=["project"]`, `skills=[name]`). Capture the full message stream and tool calls as the trace. Keep the runner model-agnostic so a candidate skill can be swapped in by writing the candidate's full skill dir (SKILL.md plus scripts/references) into place.
5. Define the metric and feedback. Implement a metric returning `{score, feedback}`: `score` captures success and cost (see the multi-objective target below); `feedback` is a natural-language diagnosis of why a run failed (step skipped, wrong tool, missing validation, ambiguous instruction, broken bundled script). This textual feedback is what reflective optimizers depend on most, so invest here.
6. Wire up the optimizer behind a thin interface. Treat the optimizer as a pluggable backend so the capability isn't bet on one tool:
   - **GEPA (primary):** the standalone `gepa` library (`GEPAAdapter`: implement `evaluate` and `make_reflective_dataset`, or start from `optimize_anything()` for an arbitrary text/file artifact) with a strong reflection LM, or `dspy.GEPA`. Use its multi-objective / Pareto support so success and cost are optimized jointly. Start with a modest budget (`auto="light"`, or a small `max_metric_calls`) and grow it only if the run looks promising.
   - **At least one alternative**, to show the result is about the capability not the optimizer: e.g. DSPy `MIPROv2`, TextGrad, or a plain LLM-driven iterative rewrite loop (propose edit → run → read feedback → repeat), the simplest backend, and a useful floor.
   Run the same skill/tasks/metric through more than one backend and compare; convergent edits across optimizers are stronger evidence for the "what makes a skill effective" finding.
7. Evolve, then evaluate. Let each optimizer mutate the skill against train/val, take the best candidate(s) (the Pareto frontier for GEPA), and evaluate on the held-out test set. Diff the original and evolved skill (body, scripts, references) to see what changed.
8. **Mine the diffs for the insight.** Categorize the edits the optimizers made (added validation, reordered steps, added examples, tightened the description, fixed a script, removed misleading prose) and tie them to the score deltas they produced. This is where the generalizable skill-authoring guidance comes from: write it up as findings, not just a metric.
9. Baselines. (a) Original human-authored seed (already measured in step 1). (b) Single-shot rewrite: ask an LLM to improve the skill once, no iteration, isolating iterative/Pareto lift from a one-shot polish. If budget allows, add best-of-N single-shot rewrites scored on validation, which controls for an optimizer simply spending more sampling.
10. Guard against overfitting and reward hacking. Hold out task variants the optimizer never saw. Check whether the evolved skill hard-codes answers or game-able shortcuts. Report the train-vs-held-out gap.

## Deliverables
- A working skill-authoring system (script/CLI): given a skill dir, task set, and metric, it runs the agent, scores it, produces textual feedback, evolves the skill, and writes out an optimized skill dir an agent can load and use as-is.
- A pluggable optimizer interface with a GEPA backend (whole skill, SKILL.md body, description, bundled scripts and reference files, as the evolvable artifact) plus at least one alternative backend.
- The original vs. optimized skill, with a readable diff and an analysis of what changed and why.
- The seed's own baseline held-out performance and documented failure modes (from step 1), so the improvement is measured against a real starting point.
- Results comparing seed / single-shot-rewrite / optimized backends on success and cost (tool-call count, token/cost), on train vs. held-out, reported in a way that reflects run-to-run noise rather than single point estimates.
- **A short findings write-up on what makes a skill effective** (the recurring, gain-carrying edits distilled into transferable skill-authoring guidance), plus failure modes and the overfitting analysis.

## Demo
A runnable script/notebook that: (1) loads the seed skill and shows its baseline held-out performance and known failure modes; (2) runs an optimizer (GEPA by default) to evolve it; (3) writes out the optimized skill dir and shows the diff and its held-out performance side-by-side with the seed and the single-shot-rewrite baseline. One command, reproducible, and the optimized skill it produces is immediately usable by an agent.

## Success Criteria
- Done: the system runs end-to-end on at least one skill and emits an optimized skill dir an agent can load, with the seed's baseline + failure modes, held-out evaluation, and both baselines.
- Good: the optimized skill helps an agent succeed more on held-out tasks than the human-authored seed (whose own held-out number and headroom are reported), and/or matches its success at lower cost, with the improvement clearly outside run-to-run noise rather than a single lucky run.
- Credible: the held-out gain survives the overfitting check (train-vs-held-out gap reported, no hard-coded shortcuts found) and beats the single-shot-rewrite baseline; the multi-objective run shows a sensible success/cost trade-off; **and the write-up names at least a few concrete, transferable patterns about what makes a skill effective**, ideally ones that recur across optimizer backends. A null result that's clearly attributable (seed already strong, feedback too thin, test set too small to detect the effect) is a legitimate finding, not a failure.

## Tech & Prerequisites
- **Languages/frameworks:** Python; `gepa` (and optionally `dspy` for `GEPA`/`MIPROv2`, or `textgrad`); Claude Agent SDK (Python or TS) for agent rollouts.
- **Background:** comfort with LLM prompting, reading agent execution traces, writing an evaluation metric, and basic experiment hygiene (train/val/test splits, avoiding leakage).
- **Access:** an API key for at least one agent model and one (ideally stronger) reflection model.

## Phases
- Explore: Read the GEPA paper and repo and the Agent Skills docs. Pick a seed skill and task set, and confirm the seed has measurable headroom (run it, record baseline + failure modes). Define and hand-check the success/cost metric on a few traces. Confirm how to swap a full candidate skill dir into the SDK runner.
- Build: Agent rollout harness, scoring, and textual feedback; the optimizer interface with a GEPA backend and one alternative; baselines.
- Evaluate: Run the optimizers, evaluate on the held-out set, compare to both baselines, run the overfitting and robustness analysis, and mine the diffs for what changed.
- Polish: Clean the demo into one reproducible command; write up the "what makes a skill effective" findings, diffs, and open questions.

## Stretch Goals
- Optimize across **multiple models** (Haiku/Sonnet/Opus or non-Anthropic) and report whether the evolved skill is model-robust or model-specific.
- Measure **discovery/triggering** specifically: since the description is already in scope, evaluate whether the evolved description makes the agent pick the skill at the right time.
- Try a second skill type (code vs. document/data) to test whether the method is skill-agnostic.
- Add a third optimizer backend and check whether the "what makes a skill effective" patterns hold across all of them.
- Compare against the `skill-creator` skill's built-in optimize/eval flow if it exposes one.

## Starter References
- **GEPA paper:** "GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning": https://arxiv.org/abs/2507.19457
- **GEPA reference implementation (library, adapters, `optimize_anything`):** https://github.com/gepa-ai/gepa
- **`dspy.GEPA` optimizer (metric/feedback API, budgets):** https://dspy.ai/api/optimizers/GEPA/overview/
- **Agent Skills overview (SKILL.md format, progressive disclosure):** https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- **Skill authoring best practices (what makes a skill good; eval-driven dev):** https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- **Agent Skills in the SDK (loading/running skills programmatically):** https://code.claude.com/docs/en/agent-sdk/skills

## Open Questions / Assumptions
- **Decided, optimization target:** Multi-objective. Optimize success and cost jointly using GEPA's Pareto frontier, rather than collapsing to a single hand-tuned blend.
- **Decided, optimization scope:** The whole skill artifact (SKILL.md body, description, and bundled scripts/reference files) is evolvable, not just the instruction prose.
- **Decided, optimizer is pluggable:** GEPA is the primary backend, but the system runs behind a thin optimizer interface with at least one alternative (DSPy `MIPROv2`, TextGrad, or a plain LLM iterative-rewrite loop), so the capability and the findings don't hinge on a single optimizer.
- **Decided, seed must have headroom:** The seed's failure modes and baseline held-out score are established and reported *before* optimization (Approach step 1). This is built into the method to ensure there's genuinely something to improve, not left as a risk.
- **Assumption:** The "coding agent skills 55%→82%" figure comes from the GEPA repo's own marketing; treat it as motivation, not a target you must hit. Achievable gains depend on the seed's headroom, which is why the method requires a seed with demonstrated failures.
- **Open:** Exactly how to swap a full candidate skill dir into the SDK runner cleanly (rewrite the dir in place per candidate, or template a per-candidate dir + `cwd`). The SDK has no programmatic skill-registration API (skills are filesystem artifacts), so the harness must manage files, including the bundled scripts. Confirm during Explore.
- **Open:** Trace fidelity. Capturing tool calls and intermediate steps richly enough to produce *useful* textual feedback is the crux; thin feedback will cap GEPA's gains. Validate the feedback quality on a few traces before scaling the run.
- **Open:** Cost/latency of running many agent rollouts per GEPA iteration; start with a small task set and a "light" budget.
- **Risk:** Evolving bundled scripts widens the search space and adds a safety surface: GEPA could write scripts that break or behave unexpectedly. Sandbox script execution and treat broken scripts as failed runs with feedback.
- **Risk:** Agent rollouts are stochastic, so the same skill scores differently across runs. The original-vs-evolved comparison is confounded by this noise, so compare across repeated runs, not single ones, so reported gains aren't just variance.
- **Note (handled in method, not an open risk):** The optimizer only helps if the seed has headroom: too strong and there's nothing to find, too weak (a strawman) and any gain is uninteresting. Approach step 1 addresses this directly: pick a plausible seed, demonstrate its visible failure modes, and report its baseline held-out performance before optimizing.
