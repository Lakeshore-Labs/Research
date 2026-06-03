# When to Give an Agent Code vs. MCP vs. CLI: A Decision Guideline (and a Router That Applies It)

**One-liner:** Produce a usable decision guideline (*for this task type, on this kind of model, expose tools as code / MCP / CLI*) backed by a working harness that runs the same tasks all three ways across tool-calling-tuned and coding-tuned models; then optionally turn the guideline into a router that picks the interface per task.

**Area:** Research / ML / Infra   •   **Difficulty:** Intermediate–Advanced

## Motivation
There are three different ways to expose capabilities to an agent, and the industry disagrees on which is best. MCP gives the model direct structured tool calls. It is the de-facto standard, but it loads every tool definition into context and costs a model round-trip per call. Code-mode has the model write code against a generated API and execute it; Anthropic and Cloudflare are pushing it with large token-savings claims. CLI access (bash plus the existing Unix tooling) is how coding agents already get most real work done.

The vendor numbers are striking but worth reading carefully. Anthropic reports one workflow dropping from ~150K to ~2K tokens (~98.7%); Cloudflare reports ~1.17M to ~1K (~99.9%) across 2,500+ endpoints. Both are vendor-published, measured on hand-picked workflows, with different backbone models and no controlled MCP baseline. They are motivation, not numbers we expect to reproduce.

What a practitioner actually needs is not another comparison table but a *rule they can apply*: given the task in front of them and the model they're running, which interface should they reach for? The headline deliverable here is that guideline, a decision rule keyed on task type and model class, and the comparison harness exists to earn it. The interesting wrinkle that makes the guideline non-trivial is the model itself: some models are tuned hard for structured tool-calling, others for writing code, and the right interface may flip depending on which you use. So the rule has to be conditioned on both task type *and* model class, and once you have it you can encode it as a router that picks the interface automatically. That is buildable in the scope of one project.

## Research Question(s)
The first two questions are the payload: the guideline and the capability. The rest are the evidence that supports them.
1. **The guideline.** Can we state a defensible decision rule (*for task type T on model class M, prefer interface I*) that a practitioner could apply without rerunning the experiment? What does that rule look like, and where is it confident vs. fuzzy?
2. **The capability.** Can a router that applies the rule (picking the interface per task, optionally per model) beat always-pick-one-interface on success/cost? This turns the insight into something usable.
3. Which task types favor which interface, and why? Working hypotheses feeding the rule: MCP wins on single simple calls; code-mode wins on multi-step composition and data wrangling and when many tools are loaded; CLI wins on file and system tasks. These are the claims to confirm or break.
4. Does the ranking flip by model class? Comparing tool-calling-tuned against coding-tuned models is the core evidence that the guideline has to be conditioned on the model, not just the task. Without this, the rule is incomplete.
5. Underneath all of it: how do the three trade off on success, token usage (ideally split into tool-definition overhead, conversation, and intermediate results), round-trips/turns, latency, cost, and reliability under repetition? This is the measured substrate the rule is read off of.

## Scope
In scope:
- A working harness that runs the same logical tasks under three adapters, sharing one set of underlying operations and swappable across models. Getting this running end-to-end and easy to iterate on is the priority.
- MCP adapter: operations exposed as MCP tools, called one at a time as structured tool calls, with all tool defs loaded into context.
- Code-mode adapter: the same operations exposed as a typed code API (TypeScript or Python). The model gets a single "write and run code" affordance and executes in a sandbox. Intermediate results stay in the sandbox unless the code returns them.
- CLI adapter: the same operations exposed as command-line tools, run through a sandboxed bash tool, with the model parsing stdout/stderr.
- A handful of tasks with deterministic ground-truth checks, spanning four families: single simple call, multi-step composition, many-tools, file/system. Start small and grow the suite as the harness stabilizes.
- A pluggable model layer so the same tasks can run on more than one model, including at least one tuned for tool-calling and one tuned for coding. This is core evidence, not a nice-to-have: the guideline is conditioned on model class.
- A metrics logger and a comparison view as supporting evidence: the raw substrate the guideline is read off of, not the headline.
- **The guideline itself:** a written decision rule, in plain terms a practitioner can apply, mapping (task type × model class) → preferred interface, with the confidence and caveats on each cell.
- **Optionally, the router** (stretch-ready but in scope to attempt): a small component that, given a task (and model), applies the rule to pick an interface, evaluated against always-pick-one baselines.

Out of scope:
- New tool-calling protocols or any model fine-tuning.
- Production hardening, multi-agent setups, and very large benchmarks. The aim is an iterable comparison, not an exhaustive one.
- Reproducing the 98.7%/99.9% vendor numbers. They are upper bounds on hand-picked workflows, not a target.

Aim for a working harness you can run and iterate on, not a perfectly controlled one-shot benchmark. Get one task flowing through all three interfaces early, then widen.

1. Share the operations across adapters. The core idea: implement the underlying operations once as plain functions (say CRUD over a mock CRM with customers, orders, and files) and make each adapter a thin shell over those same functions, exposing the same operations. Keep the comparison as fair as you reasonably can: aim for operation parity (each adapter offers the same operation set), keep real work like filtering, looping, joining, and aggregation as something the model drives through the interface rather than baking it into one operation, and hold run conditions (model, temperature, turn limits, system-prompt scaffold) the same except for the paragraph explaining the interface. Some asymmetries are structural and are exactly what you are measuring: CLI gets pipes and grep, code-mode gets loops and variables, MCP gets typed schemas; treat those as properties of the interface, not bugs to equalize away. Other asymmetries are accidental (one adapter happens to return prettier JSON); fix the easy ones and write down the rest. The goal is documenting confounds honestly, not a contract you have to police.
2. Build a few tasks with deterministic checks. Each task is (prompt, setup state, verifier). Prefer scoring by final-state diff (DB plus filesystem state vs. goal state, the τ-bench approach) over judging the transcript; for read-only tasks, check the returned answer against an expected value. Cover four families:
   - Single simple call, e.g. "look up customer X's email."
   - Multi-step composition / data wrangling, e.g. "fetch all orders, keep those over $100, report totals per region."
   - Many-tools: a large pool of operations loaded where only two or three are needed. This stresses MCP's context cost and code-mode/CLI's discovery cost. Tune the pool size to the model's context window.
   - File/system, e.g. "find every .csv, merge, dedupe, count rows."
   Start with one or two tasks per family and add more once the harness is stable.
3. Make the model swappable, and run several. Put model choice behind one config point so the same tasks run across models with minimal effort. Include at least one model tuned for tool-calling and one tuned for coding. Seeing whether the interface ranking flips by model is a main result, so build for it from the start rather than bolting it on.
4. Sandbox both code-mode and CLI. Arbitrary code and shell execution is a real exposure (see Open Questions). Use an isolated executor: a container (Docker), a microVM, or a JS isolate (`isolated-vm` / Workers, Cloudflare-style) for code-mode, and a locked-down bash tool with no network and a scratch filesystem for CLI. Reset state between runs.
5. Instrument every run. Log input and output tokens, ideally separating tool-definition overhead from conversation from returned intermediate results, since that split is where the interesting differences show up; turns/round-trips; latency; cost from published per-token prices; and a rough error taxonomy (bad call, parse failure, runtime error, recovered after retry, gave up).
6. Replicate and iterate. Agent runs are stochastic, so run each task × interface × model a few times and report mean success alongside pass^k (all k attempts pass), the τ-bench reliability metric. Then look at the results by family and by model, see which RQ3 hypotheses hold, and feed what you learn back into the tasks and harness.
7. Distill the guideline, then (optionally) wire the router. Once the by-family-by-model results stabilize, write them up as a decision rule: a (task type × model class) → interface table with a confidence note per cell, plus the one-line "if X, reach for Y" version a practitioner would actually use. If time allows, encode that rule as a router (even a simple classifier or set of heuristics over task features) and measure it against always-MCP / always-code / always-CLI baselines on a held-out slice of tasks. The router is the proof the guideline is real and usable; the table is the proof it generalizes.

## Deliverables
- **The headline: a decision guideline.** A (task type × model class) → preferred-interface table with a confidence note per cell, plus the plain-language "if your task looks like X and your model is Y, reach for Z" rules a practitioner can apply without rerunning anything.
- **Optionally, a router** that applies the guideline to pick an interface per task (and model), with a measured comparison against always-pick-one baselines.
- A runnable, iterable harness in one repo (the evidence engine): three adapters over a shared operation layer, with model choice as a config point.
- A task suite (a handful of tasks across the four families) with deterministic verifiers and seed/reset scripts, easy to extend.
- Supporting evidence: a metrics logger and a comparison view (success, tokens, turns, latency, cost, pass^k per interface, per family, and per model): the substrate the guideline is read off of, not the punchline.
- A short report that leads with the guideline (and router result if built), then backs it with the trade-offs seen, how they shift by model, and the confounds hit.

## Demo
Lead with the guideline: show the (task type × model class) → interface table and read off a couple of its rules out loud. If the router exists, hand it a fresh task, watch it pick an interface, and show it beating the always-pick-one baselines. Then back it up with the evidence: one command runs the suite across all three interfaces (and, ideally, a couple of models) and renders the comparison side by side: success per interface and per family, total tokens with tool-def overhead broken out where possible, round-trips, latency, cost, plus one or two walked-through tasks showing how each interface attacked the same problem.

## Success Criteria
- Done: the same tasks run end-to-end under all three interfaces with deterministic scoring, model choice is swappable, the comparison view renders the metrics per interface and per family, and you can state a first-cut decision guideline from the results. The system is something you can rerun and extend.
- Good: it runs across at least two models (one tool-calling-tuned, one coding-tuned), confounds are documented honestly, results are reasonably stable across re-runs, and the guideline is defensible and conditioned on both task type and model class: a rule a practitioner could actually apply, with at least one non-obvious cell (e.g. a measured size for MCP's context penalty on the many-tools family, code-mode's round-trip savings on multi-step tasks, or a ranking that flips between the two model classes).
- Reach: the router is built and beats every always-pick-one baseline on a held-out slice, turning the guideline into a working capability.

## Tech & Prerequisites
- Languages: Python and/or TypeScript. The code-mode API is cleanest in one of these; the harness can be Python.
- Tools: an LLM API with tool calling (Anthropic or OpenAI); the MCP SDK; a sandbox (Docker, microVM, or `isolated-vm`/Workers); a simple dashboard (Streamlit or a static HTML report).
- Background: LLM tool/function calling, basic agent loops, async and subprocess handling, and experiment hygiene (seeds, replication, controlling confounds). τ-bench-style state-diff evaluation helps.

## Phases
- Explore: read the starter refs; pick the sandbox tech and an initial set of models (at least one tool-calling-tuned, one coding-tuned); sketch how the adapters share operations; spike one task end-to-end across all three interfaces.
- Build: the shared operation layer, the three adapters, the swappable model layer, the sandbox, the metrics logger, and a starter set of tasks with verifiers.
- Evaluate: run a few replications across interfaces and models, collect metrics, build the comparison view, and look at results by family and by model.
- Distill: turn the stable results into the decision guideline (the table plus the plain-language rules); if time allows, encode it as a router and test it against always-pick-one baselines.
- Polish: write the report leading with the guideline (and router result), get the repo to one-command repro, prep the demo. Iterate on tasks and the harness as findings come in.

## Stretch Goals
- Widen the model set further, or add a model from a different vendor, to test how general the guideline is and whether the router transfers.
- A per-step hybrid: let the model (or the router) pick interface step-by-step within one task rather than once up front, as a finer-grained capability.
- Measure progressive disclosure: code-mode with on-demand tool discovery vs. all-defs-loaded, to separate how much of code-mode's win is just context savings.
- A security probe: attempt prompt injection and sandbox escape on the code-mode and CLI adapters and report containment.

## Starter References
- Anthropic, *Code execution with MCP: building more efficient AI agents* (the code-mode-over-MCP argument and the token-savings claims): https://www.anthropic.com/engineering/code-execution-with-mcp
- Cloudflare, *Code Mode: the better way to use MCP* (MCP tools turned into a TypeScript API the model writes code against; V8-isolate sandbox): https://blog.cloudflare.com/code-mode/
- Model Context Protocol specification (how tools/servers are exposed; JSON-RPC; tool annotations): https://modelcontextprotocol.io/specification/2025-06-18
- Wang et al., *Executable Code Actions Elicit Better LLM Agents* (CodeAct; the paper reports up to +20% success and ~30% fewer steps vs. JSON tool calls, their result, worth re-checking against this suite): https://arxiv.org/abs/2402.01030. Code: https://github.com/xingyaoww/code-act
- Yao et al., *τ-bench: A Benchmark for Tool-Agent-User Interaction* (state-diff evaluation and the pass^k reliability metric this proposal borrows): https://arxiv.org/abs/2406.12045. Code: https://github.com/sierra-research/tau-bench

## Open Questions / Assumptions
- Fairness is a judgment call, not a checklist. Sharing the underlying operations across three structurally different interfaces gets you most of the way, but the line between "structural property under test" and "accidental confound" stays fuzzy. The aim is to keep it as fair as you reasonably can and document the confounds, not to enforce a rigid contract. Sanity-check the setup with someone before drawing strong conclusions.
- Model choice drives the result, which is why the guideline is conditioned on it. A model trained heavily on one interface (e.g. Anthropic models with MCP) may favor it. Comparing tool-calling-tuned against coding-tuned models is what lets the rule say "on model class M, prefer I"; how many models and which ones is open and can grow, and a rule built on two model classes is a starting point, not the final word.
- The Anthropic ~98.7% and Cloudflare ~99.9% figures are vendor claims on hand-picked workflows, not validated general results. They are motivation and rough upper bounds, not targets. Assumption: they will not hold on a neutral suite.
- How many tasks, how many replications, and the many-tools pool size are all left loose on purpose. Start small, get the harness running, and tune as you go; tie the many-tools pool to the model's context window.
- Cost numbers depend on published per-token prices, which change. Record the price sheet and date used so runs stay comparable.
- Sandboxing is a hard requirement for both code-mode and CLI because both execute arbitrary code/shell. The scope assumes a single-machine, no-network sandbox is enough; production-grade isolation is out of scope.
