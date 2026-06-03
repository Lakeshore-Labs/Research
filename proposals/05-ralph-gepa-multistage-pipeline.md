# Coupling an Autonomous Build Loop with a Prompt Optimizer: A Ralph × GEPA Recipe

**One-liner:** Work out a general recipe for combining an autonomous agentic build loop (Ralph) with a prompt optimizer (GEPA, but the pattern isn't GEPA-specific), covering when the builder hands off to the optimizer, how optimized prompts survive fresh-context iterations, and what triggers re-optimization, using a deliberately thin multi-stage pipeline as the testbed.

**Area:** Research / Infra / ML   •   **Difficulty:** Advanced (the contribution is the coupling recipe, not a polished full-stack app; the pipeline is a vehicle and is allowed to stay thin)

## Motivation
Two recent techniques sit at opposite ends of building an AI system and nobody has written down how to make them work together. The Ralph loop ([Huntley's blog](https://ghuntley.com/ralph/)) is an autonomous outer loop that re-runs an agent on a spec across fresh-context iterations until it converges; it builds long-horizon systems but drifts and ships placeholder code when the spec is weak, and it deliberately carries no context between iterations, only files and git. GEPA ([arXiv:2507.19457](https://arxiv.org/abs/2507.19457)) is a reflective prompt optimizer that improves LLM programs by reading full execution traces rather than scalar rewards alone. A builder that forgets everything each iteration and an optimizer that produces prompts you want to keep are an awkward but interesting pair: where does the optimized prompt live so the loop doesn't overwrite it, who decides when to stop building and start optimizing, and what makes the loop re-optimize later?

That coupling question is the contribution, and it's transferable: the same hand-off problem shows up with any autonomous build loop and any prompt optimizer, not just this exact stack. We pick a compound, multi-stage pipeline ([BAIR, 2024](https://bair.berkeley.edu/blog/2024/02/18/compound-ai-systems/)) as the testbed because it has several LLM stages worth optimizing, but the pipeline is a vehicle for studying the coupling, not the deliverable.

(Huntley's posts are blog write-ups, not peer-reviewed; the GEPA paper is an arXiv preprint. Treat both as starting points, not settled results.)

## Research Question(s)
1. **(Headline) What's the recipe for coupling an autonomous build loop with a prompt optimizer?** Specifically: at what point does the builder hand off to the optimizer; how do optimized prompts persist across fresh-context build iterations without being clobbered; and what triggers re-optimization (a contract failure, a schedule, a suspected regression)? The answer should read as a pattern someone could reuse with a different loop or a different optimizer.
2. Does the coupled system actually beat the builder-only and optimizer-only arms on a fixed task, or does the hand-off cost more than it returns? A clean negative is a real result.
3. Where does optimization budget pay off across stages (which prompts matter), and where does the build loop drift or waste tokens: the signals a re-optimization trigger would key off.

## Scope
- In scope (the research):
  - **The coupling recipe:** a documented hand-off interface (where optimized prompts live, how they survive fresh-context iterations), a re-optimization trigger, and an acceptance rule, written up so it transfers to other loops/optimizers.
  - A Ralph driver: the `ralph-loop` Claude Code plugin (or a transparent bash-loop equivalent) running against a stable spec, with backpressure (tests/contracts) between iterations.
  - A GEPA harness (`dspy.GEPA`) over the LLM stages that have a measurable target, used here as one concrete optimizer to instantiate the coupling, not as the contribution.
  - An evaluation harness good enough to compare arms: end-to-end metric, per-stage contract checks, cost/iteration logging.
  - A three-way comparison: build-loop-only vs. optimizer-only vs. coupled.
- Testbed (the vehicle, deliberately allowed to be thin):
  - A multi-stage pipeline for one anchor use case: ingestion → extraction → medallion (bronze/silver/gold) → serving → MCP layer → application. **It only needs to be real enough to exercise the coupling** (i.e. at least two LLM stages worth optimizing and an end-to-end metric). Stages with no LLM in them may be scaffolded, stubbed, or shrunk to a single table; a polished production app is explicitly *not* required.
- Out of scope (non-goals):
  - **Building a polished, full-stack production pipeline.** The pipeline is the testbed; do not let it swamp the coupling research. If a stage doesn't bear on the hand-off, keep it thin.
  - NL-to-SQL or skill-enhancement uses of GEPA (covered by sibling proposals).
  - Production infra (real warehouses, k8s, streaming, scale). Use local/embedded tools: DuckDB, SQLite, files.
  - Inventing a new optimizer or agent loop. Use Ralph and GEPA as given.
  - Optimizing every prompt in the stack. Focus the optimizer on the highest-leverage LLM stages.

## Suggested Approach
A starting path, not prescriptive.

1. **Pick a small, verifiable anchor use case** so "the app answers correctly" is checkable. This is the testbed, keep it minimal. Suggested: a Q&A app over a public structured/semi-structured corpus (a CSV/JSON dump of public filings, a GitHub issues export, or a small open dataset) with a fixed set of question→expected-answer pairs as the end-to-end eval. Size the set so it's cheap enough to run many loops yet large enough that an accuracy delta is meaningful. Hold out a portion the GEPA optimization never sees, so reported accuracy isn't fit to the same questions GEPA trained on.
2. **Define the stages and their contracts** (what "good" looks like at each, and which are LLM-driven so the optimizer can touch them). Only the LLM stages need to be substantial; the deterministic stages can be as thin as a single transform if that's enough to feed the next LLM stage:
   - *Ingestion* (deterministic): land raw source verbatim; contract: bytes/rows landed, manifest written.
   - *Extraction* (LLM, GEPA target): parse/structure raw into typed records; contract: schema-validation pass rate against a fixed JSON schema.
   - *Medallion* (deterministic): bronze→silver→gold SQL transforms ([Databricks](https://www.databricks.com/blog/what-is-medallion-architecture)); contract: `dbt test` or hand-written SQL assertions; row-count, not-null, unique checks per layer.
   - *Serving* (deterministic): query/API layer over gold (FastAPI + DuckDB/SQLite); contract: endpoint returns correct rows for canned queries.
   - *MCP layer* (deterministic): expose serving as MCP tools/resources ([MCP](https://modelcontextprotocol.io/)); contract: `tools/list` exposes expected tools, `tools/call` returns valid results.
   - *Application* (LLM, GEPA target second): agent app answering user questions via the MCP tools; contract: end-to-end answer accuracy on the fixed eval set.
3. **Write the Ralph spec + guardrails first.** Ralph's output quality tracks spec quality (Huntley). Use a `PROMPT.md` spec, a living TODO, an explicit "no placeholder implementations" rule, and the per-stage contracts as backpressure between iterations. Run via the plugin: `/ralph-loop "..." --max-iterations N --completion-promise "DONE"`. Set a sane iteration ceiling so the loop can't run unbounded, and treat hitting it without convergence as a result to record, not a failure to hide.
4. **Layer GEPA on the LLM stages.** Express the LLM stages as a DSPy program and run `dspy.GEPA` with a metric that returns textual feedback (a trace), not just a scalar; that feedback channel is GEPA's main advantage over score-only optimizers. Optimize extraction first. Then add the answer stage and run GEPA over both modules at once to test whether joint optimization beats optimizing the highest-leverage stage alone.
5. **Figure out the Ralph×GEPA coupling. This is the headline research question and the main deliverable.** Everything above is plumbing so you can get here. The interface and the trigger logic are what you're discovering, so treat the sketch below as one starting hypothesis, not a spec to implement verbatim. Frame your findings as a recipe, not a one-off: which parts would carry over to a different build loop or a different optimizer, and which are GEPA- or Ralph-specific quirks.
   - *Interface (likely starting point).* Put the optimizable prompts in known files (e.g. `prompts/extraction.txt`, `prompts/answer.txt`) so Ralph reads them and GEPA writes them. This fits Ralph's habit of carrying state in files/git across fresh-context iterations, but whether a file hand-off is the right surface is itself worth testing.
   - *Trigger (genuinely open).* When and how Ralph should hand off to GEPA (on a contract failure, on a fixed schedule, on suspected prompt-caused regressions, or something else) is the core thing to work out. Try a few coupling strategies rather than committing to one up front, and keep each invocation bounded so the nested loops don't run away.
   - *Acceptance.* A GEPA run should only replace the incumbent prompt if it does no worse on the held-out split; otherwise discard its output. This keeps the loop from drifting backward.
   - *Who owns what.* Ralph owns building and wiring; the eval harness owns measuring; GEPA only ever sees the prompt text and the feedback metric. Keeping these separate is what makes the ablation arms comparable.
   - A simple fixed-schedule coupling (run GEPA once after the pipeline builds) is a reasonable first milestone on the way to the more interesting loop-triggered, closed-loop version.
6. **Resolve the ablation confound up front.** GEPA-only needs a pipeline to optimize, and Ralph is what builds it, so the three arms aren't naturally symmetric. Pin them down: build the pipeline once with Ralph using *unoptimized* prompts and freeze that code. (a) *Ralph-only* = that frozen build, prompts as Ralph left them. (b) *GEPA-only* = same frozen build, prompts replaced by GEPA-optimized ones, no further Ralph iteration. (c) *Combined* = GEPA-optimized prompts plus any additional Ralph iterations the coupling triggers. All three run against the same eval harness and held-out split. State this protocol in the writeup; it's what makes the headline number trustworthy.

## Deliverables
- **The headline: a written-up coupling recipe.** The hand-off interface, the re-optimization trigger(s) you tried, the acceptance rule, how state persists across fresh-context iterations, and an honest account of what generalizes beyond Ralph+GEPA versus what's stack-specific.
- A runnable repo containing the coupling machinery plus a thin pipeline good enough to exercise it (it does not need to be a polished app).
- The Ralph spec/guardrails (`PROMPT.md`, TODO, contracts) used to drive the build.
- The GEPA harness (DSPy program + feedback metric) over the chosen LLM stage(s), as the concrete optimizer instance.
- An evaluation harness producing end-to-end accuracy, per-stage contract pass/fail, and cost/iterations, with a held-out split.
- A comparison table: build-loop-only vs. optimizer-only vs. coupled, with deltas and the protocol from approach step 6.

## Demo
Show the coupling working: run a command (or short script) that drives the build loop, hands off to the optimizer at the trigger point, writes an optimized prompt back, and shows the pipeline answering the fixed question set better afterward, with a side-by-side of the three arms. The demo is about the hand-off behaving as designed, not about a slick app.

A sensible build order (thin testbed first, then the coupling):
- First get a thin path end to end (ingestion → extraction → a single gold table → serving → one MCP tool → a CLI/chat app answering the fixed question set) so two LLM stages are wired and the end-to-end metric runs. This is enough testbed; don't gold-plate it.
- Then build the coupling on top: the hand-off interface, the re-optimization trigger, the acceptance rule, with GEPA optimizing the highest-leverage LLM stage(s).
- Run the three-way comparison using the protocol in approach step 6.

The end state is a demonstrated, documented coupling, not a production app.

## Success Criteria
- **A documented coupling recipe (primary bar):** a clear account of the hand-off interface, the re-optimization trigger(s), how prompts persist across iterations, and the acceptance rule, with an explicit take on what transfers to other loops/optimizers. The recipe is the result; the running system is its evidence.
- **The coupling demonstrably runs:** the build loop hands off to the optimizer and back, the optimized prompt survives subsequent iterations, and the pipeline answers the held-out question set end to end. The testbed only has to be real enough to show this.
- **Three-arm comparison (a key result):** build-loop-only vs. optimizer-only vs. coupled on held-out end-to-end accuracy and per-stage metrics, with a plain statement of whether coupling helped, hurt, or was neutral. A rigorously shown negative result counts as success. Keep the question set large enough that deltas mean something, and flag deltas small enough to be noise rather than over-reading them.
- **Per-stage contracts:** each implemented stage passes its contract (schema validation, SQL/dbt tests, MCP `tools/list`/`tools/call`). Report the extraction schema-pass rate.
- **Convergence cost:** iterations and tokens/$ for Ralph to reach a passing build, plus tokens/$ per GEPA run, logged so cost stays bounded. "Converged" = all implemented contracts pass; hitting the iteration ceiling without that is a recorded outcome.

## Keeping the Loops Bounded
Cost isn't a gating concern here, but two token-burning loops nested still need a stop condition so nothing runs away unattended. Keep things bounded without obsessing over exact numbers:
- Ralph: set an iteration ceiling via `--max-iterations` and log wall-clock and tokens per build, so a non-converging loop stops on its own rather than spinning forever.
- GEPA: give each invocation a finite rollout/eval budget and bound how often it can fire per build. Each run discards its output if it doesn't beat the incumbent on the held-out split.
- Keep the corpus and question set modest so a full ablation run is cheap enough to repeat freely.
- Treat hitting any ceiling as a data point to report, not something to quietly raise mid-experiment.

## Tech & Prerequisites
- **Languages/frameworks:** Python; DSPy (`dspy.GEPA`); DuckDB or SQLite; dbt (or hand-written SQL assertions for the slice); FastAPI (serving); the Python `mcp` SDK; Claude Code + the `ralph-loop` plugin.
- **Background:** LLM prompting/agents, basic data engineering (SQL, schemas, the medallion pattern), API basics, reading a research paper. Pair-friendly: one person owns build/orchestration, the other eval/optimization.

## Phases
- **Explore:** Read the starter refs. Run the `ralph-loop` plugin on a toy task and a `dspy.GEPA` tutorial standalone to understand each in isolation. Pick the anchor use case; write the fixed eval set, the held-out split, and the per-stage contracts.
- **Build the testbed (keep it thin):** Write the Ralph spec; get the thin end-to-end path building under the loop with two LLM stages and a working metric. Stop thickening once the coupling can be exercised. Stand up the GEPA harness on extraction, then a second stage.
- **Build the coupling (the main work):** design and try the hand-off interface, the re-optimization trigger(s), and the acceptance rule; this is where most of the time should go.
- **Evaluate:** Wire the eval harness; freeze the build; run the three-arm comparison per the step-6 protocol; log cost/iterations.
- **Write up:** Reduce the demo to one command; write up the coupling recipe, what generalizes, the comparison findings, and open questions.

## Stretch Goals
- Loop-triggered, closed-loop coupling: have Ralph invoke GEPA autonomously (e.g. when an LLM-stage contract fails) and measure whether self-triggered optimization beats a fixed-schedule run. This is the most interesting form of the coupling question.
- **Swap the optimizer** to test the recipe's generality directly: run the same coupling with a different prompt optimizer (or an RL-style baseline, as the GEPA paper does) and see how much of the hand-off design carries over. This is the strongest evidence the contribution isn't GEPA-specific.
- A second anchor use case to test whether the coupling pattern generalizes across tasks.
- A polished, non-CLI app UI over the same pipeline (explicitly a stretch, not part of the core deliverable).

## Starter References
- Ralph technique (blog, primary source): https://ghuntley.com/ralph/ and the follow-up https://ghuntley.com/loop/
- `ralph-loop` Claude Code plugin (the concrete driver): https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop
- GEPA paper (arXiv preprint, multi-module reflective optimization): https://arxiv.org/abs/2507.19457 ; repo: https://github.com/gepa-ai/gepa
- DSPy + `dspy.GEPA`: https://github.com/stanfordnlp/dspy and https://dspy.ai/api/optimizers/GEPA/overview/
- Compound AI systems framing (BAIR blog): https://bair.berkeley.edu/blog/2024/02/18/compound-ai-systems/
- MCP: https://modelcontextprotocol.io/ ; medallion architecture (Databricks vendor blog): https://www.databricks.com/blog/what-is-medallion-architecture

## Open Questions / Assumptions
- **The coupling design is the real research and the literature doesn't cover it.** Neither Ralph nor GEPA documents a way to combine them, and we're deliberately leaving the trigger logic open: when Ralph should hand off to GEPA, how state persists across the hand-off, and whether a file interface is even the right surface are the questions to answer empirically. Expect to try several strategies rather than land on one in advance.
- **Decided:** GEPA is the optimizer we instantiate the coupling with, but it's one option; the recipe is meant to read as loop-agnostic and optimizer-agnostic. Keep the write-up framed that way.
- **Decided (descope):** the deliverable is the coupling recipe plus evidence it works, not a polished full-stack production app. The pipeline is the testbed and may be thin or partly scaffolded wherever a stage doesn't bear on the hand-off. Resist the pull to finish three things (full build, nested loops, clean comparison) at once; the build is in service of the coupling.
- **Decided:** cost is not a gating concern; just keep the nested loops bounded so nothing runs unattended forever.
- **Assumption:** writing GEPA-optimized prompts back to files is a reasonable first hand-off, since Ralph carries state in files/git, not context. Validate early whether it composes cleanly, and be ready to abandon it if a better interface emerges.
- **Ablation symmetry:** the frozen-build protocol (step 6) is the proposed fix for the fact that GEPA-only still needs Ralph to build the pipeline. If there's a cleaner way to make the three arms comparable, take it.
- **Eval ground truth:** the question→answer set is the linchpin. Building a fair, non-trivial set with a real held-out split that GEPA can't fit to needs care; budget time for it.
- **Staffing:** Advanced. The coupling design plus a thin testbed is pair-sized; keeping the testbed thin is what keeps it pair-sized. Confirm the expectation that the pipeline stays a vehicle.
- **Versions:** `dspy.GEPA` and the MCP Python SDK move fast. Pin versions at project start; the APIs cited here may have shifted by the time work begins.
