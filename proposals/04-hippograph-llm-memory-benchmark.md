# A Pluggable Long-Term Memory Layer for LLM Agents (HippoRAG 2 / Zep / Mem0)

**One-liner:** Build a pluggable long-term memory layer that measurably improves an assistant or agent (swap HippoRAG 2, Zep, or Mem0 behind one interface and actually use it), and from running it, derive a transferable design guideline: which memory architecture wins for which kind of recall (graph memory for multi-hop/associative vs. flat vector for simple lookup).

**Area:** Research / ML   •   **Difficulty:** Intermediate–Advanced

> Naming note: this proposal originally used "Hippograph." That's resolved to **HippoRAG 2**, the backend used here. A separate, similarly-named literal project (HippoGraph Pro) exists but isn't in use; see Open Questions.

## Motivation
LLM agents forget everything outside their context window, so a real assistant needs a memory layer that stores facts from past sessions and retrieves them on demand. The leading approaches disagree on architecture: vector stores with LLM fact extraction (Mem0), temporal knowledge graphs (Zep/Graphiti), and knowledge graphs with Personalized PageRank retrieval (HippoRAG). Each vendor reports wins on a benchmark of its own choosing, often with a different backbone LLM, so the published numbers don't compare cleanly, and none of them tells you what it's actually like to wire that backend into your own agent. This project delivers two things a practitioner can use: first, the memory layer you'd want to ship, which is one interface, swappable backends, plugged into a working agent; and second, a design guideline that outlives any leaderboard. Given a kind of recall (simple lookup, multi-hop/associative, temporal/knowledge-update), which memory architecture should you reach for? The measurement here is in service of that guideline, not the headline.

## Goals / Questions
Two headline deliverables: a working, reusable memory layer, and a transferable design insight about memory architectures. The comparison isn't a leaderboard; it's the evidence behind the guideline.
1. Capability: can the same agent use HippoRAG 2, Zep, or Mem0 as its memory with no code changes above the interface, and does plugging memory in measurably improve what the agent can recall versus no memory? That's the system you ship.
2. Insight: which memory architecture wins for which kind of recall? Does graph memory earn its complexity on multi-hop/associative recall while flat vector suffices for simple lookup, and do temporal graphs (Zep) win on knowledge-update and temporal questions? The output is a design guideline ("for recall type X, reach for architecture Y"), not a ranking of three products.
3. Practicality: what does each architecture cost in practice (indexing work, retrieval latency, answer tokens), so the guideline accounts for what it actually takes to run, not just accuracy.

## Scope
- In scope:
  - A common memory interface that an agent talks to, with three interchangeable backends behind it: HippoRAG 2 (graph + Personalized PageRank, MIT-licensed, `pip install hipporag`); Zep (OSS Graphiti stack or hosted API); Mem0 (OSS, vector mode and graph "Mem0g" mode).
  - A small demo agent that uses the layer, so the system is actually usable and not just a scoring script.
  - A no-memory sanity baseline (stuff the history into the prompt where it fits, truncate otherwise) as a floor, so you can tell whether memory is helping at all on a given query type. Worth having; not a hard requirement.
  - Evaluation of the working system on a realistic long-memory task, holding the backbone LLM, embedding model, and judge model fixed across backends wherever each one lets you set them.
  - Looking at accuracy by query type (single-hop, multi-hop, temporal, knowledge-update/open-domain) alongside indexing cost, retrieval latency, and answer token cost.
- Out of scope:
  - Inventing a new memory algorithm or modifying any backend's internals.
  - Multi-modal memory, multi-agent memory sharing, production SLAs.
  - Turning this into a leaderboard or reproducing every vendor's full benchmark suite.

## Suggested Approach
1. Design the memory interface first. This is the heart of the project. Something like `ingest(session)`, `retrieve(query) -> context`, `answer(query) -> str`, plus whatever an agent actually needs (writing new memories mid-conversation, scoping by user/session). Wrap each backend to it so the agent above never knows which one is plugged in.
2. Get the three backends running behind the interface and wire up a small demo agent that uses them, so you can actually swap memory at runtime and watch it work. The no-memory baseline slots in as one more backend.
3. Pick a realistic long-memory task to evaluate on. LongMemEval (multiple question types, long chat histories, built for assistant memory) and LoCoMo (multi-session dialogues; single/multi-hop/temporal/open-domain) are both reasonable, and both are reported on by Zep and Mem0, which helps. Either is fine; pick one to start and check the current dataset release for question counts and history lengths rather than trusting paper figures.
4. Keep the comparison honest about home turf. No task is native to all three: Zep reports on DMR + LongMemEval, Mem0 on LoCoMo, HippoRAG on multi-hop QA (MuSiQue/2Wiki/HotpotQA). Whatever you evaluate on, at least one backend is off its home turf, so note which backend the task favors by design and read any single-task ranking as suggestive, not a verdict. A second task is a good cross-check if you have time.
5. Hold the backbone and judge fixed. Same generation LLM, same embedding model, same LLM-as-judge prompt for every backend where each is configurable. Where a backend forces its own model or prompt (e.g. a hosted API), record what it uses and flag that comparison as confounded.
6. Run the working system on the task and look at it by query type and by cost. Capture answers, retrieved context, token counts, and latency; score with the task's own metric, and log retrieval recall where the dataset gives evidence spans. Pull it into one view: architecture × question-type accuracy, plus indexing cost, retrieval latency, and answer tokens. The point isn't who has the highest overall number; it's to write down the design guideline the data supports: for each recall type (simple lookup, multi-hop/associative, temporal/knowledge-update), which memory architecture a practitioner should reach for, and at what cost. Read each row as evidence about an architecture (graph-with-PageRank, temporal graph, vector-with-extraction), not just about a vendor.

## Deliverables
- A usable memory layer (repo): the common interface plus working HippoRAG 2, Zep, and Mem0 backends, swappable behind it. This is the main thing.
- A small demo agent that uses the layer, so the system can actually be run and used.
- An evaluation of the working system on a realistic long-memory task: accuracy by question type, retrieval recall where measurable, latency, and token cost.
- A design guideline practitioners can apply: which memory architecture to reach for given a kind of recall, and what it costs, with honest caveats about which comparisons are confounded and why.

## Demo
Run the demo agent and swap its memory backend on the fly (`--memory {hipporag,zep,mem0,none}`), showing it recall facts from earlier sessions through whichever backend is plugged in. Plus a notebook or simple dashboard that renders the comparison across backends and question types.

## Success Criteria
Roughly in order of importance:
1. The capability works: the demo agent runs against HippoRAG 2, Zep, and Mem0 through one interface, with no changes above that interface to switch backends, and memory measurably beats the no-memory floor on at least some query types.
2. You can state a design guideline (for which recall types graph memory earns its complexity, where flat vector suffices, where temporal graphs win) with enough runs behind it that you're not calling noise a win, and flag close results as ties.
3. The system is evaluated end-to-end on the chosen task with a shared backbone, embedding, and judge model wherever each backend allows it, and every place a model or prompt was forced is written down.
4. The guideline is stated honestly as conditioned on the task and its home-turf bias, ideally cross-checked on a second task.
- Nice to have: where a result confirms or contradicts a vendor's published number, note both numbers and the likely reason for the gap (different backbone, metric, or split).

## Tech & Prerequisites
- Languages/frameworks: Python 3.10+. HippoRAG (`pip install hipporag`), Mem0 OSS, Zep / Graphiti, sentence-transformers, an LLM API (OpenAI-compatible) or a local model via vLLM.
- Background: comfortable with RAG basics (chunking, embeddings, vector search), knowledge graphs at a conceptual level, and writing evaluation harnesses. Familiarity with LLM-as-judge scoring. No graph theory required, though knowing what Personalized PageRank does helps.

## Phases
Sized for one term but tight; if time runs short, do one task and the three core backends and cut the stretch goals first.
- Explore: Read the HippoRAG, Zep, and Mem0 papers and a task paper. Get each backend running on a single toy conversation. Decide the task and the fixed models. Budget for installation pain: Zep/Graphiti needs a graph DB, and getting all backends onto one backbone is usually the slowest part.
- Build: Design and implement the memory interface and the backends behind it; wire up the demo agent and ingestion.
- Evaluate: Run the working system on the task, collect accuracy/recall/latency/cost, slice by question type, and write down every confound as you hit it.
- Polish: Write up the findings, build the dashboard/notebook, clean up the repo so someone else can run it.

## Stretch Goals
- Add a second task (LoCoMo if you started with LongMemEval, or the reverse) to see whether the picture holds across datasets, the main check on home-turf bias.
- Run Mem0's vector-only and graph ("Mem0g") modes as separate backends to isolate what the graph alone buys.
- Vary retrieval `top_k` and backbone size to see how stable the picture is.
- Add another backend (GraphRAG or RAPTOR) behind the same interface.

## Starter References
- HippoRAG paper (NeurIPS'24), "Neurobiologically Inspired Long-Term Memory for LLMs": https://arxiv.org/abs/2405.14831
- HippoRAG 2 paper (ICML'25), "From RAG to Memory: Non-Parametric Continual Learning for LLMs": https://arxiv.org/abs/2502.14802
- HippoRAG implementation (HippoRAG + HippoRAG 2, MIT): https://github.com/OSU-NLP-Group/HippoRAG
- Zep, "A Temporal Knowledge Graph Architecture for Agent Memory" (reports its own DMR + LongMemEval numbers): https://arxiv.org/abs/2501.13956
- Graphiti, Zep's OSS temporal knowledge graph engine (Apache 2.0): https://github.com/getzep/graphiti
- Mem0, "Building Production-Ready AI Agents with Scalable Long-Term Memory" (reports its own LoCoMo numbers, Mem0 vs Mem0g): https://arxiv.org/abs/2504.19413
- Mem0 implementation (OSS, vector + graph "Mem0g" modes, Apache 2.0): https://github.com/mem0ai/mem0
- LongMemEval benchmark (paper + repo): https://arxiv.org/abs/2410.10813 and https://github.com/xiaowu0162/LongMemEval
- LoCoMo benchmark (very long-term conversational memory): https://snap-research.github.io/locomo/
- The literal "Hippograph" project (see Open Questions): https://github.com/artemMprokhorov/hippograph-pro

## Open Questions / Assumptions
- Naming, resolved. "Hippograph" here means HippoRAG 2 (OSU-NLP-Group, NeurIPS'24): an LLM-built knowledge graph as a "hippocampal index" with Personalized PageRank retrieval; mature, MIT-licensed, benchmark-backed. There is also a separate, literal project "Hippograph / HippoGraph Pro" (github.com/artemMprokhorov/hippograph-pro), a self-hosted graph-associative memory using spreading activation, local BGE-M3 embeddings, and GLiNER entity extraction. Its README claims roughly 90.8% LoCoMo retrieval, which is self-reported and unverified. It looks early-stage and we're not using it; if someone later wants it in, it slots in behind the interface like any other backend.
- Which task to evaluate on is open. The user is genuinely undecided here, and that's fine. LongMemEval and LoCoMo are both reasonable starting points, and the layer doesn't depend on the choice. Just be aware no task is native to all three backends (Zep: DMR + LongMemEval; Mem0: LoCoMo; HippoRAG: multi-hop QA), so whatever you pick, someone is off home turf; read the results with that in mind.
- Vendor numbers are claims, not ground truth. The accuracy figures in the source papers were each produced by the vendor under its own setup. If you compare, report your own numbers and don't quote theirs as established.
- Forced-model confound. Zep's hosted offering and possibly others pin internal models or prompts, so a perfectly uniform backbone may not be reachable. Where it isn't, record what each backend actually used and flag that comparison.
- Cost and access. Hosted APIs (Zep cloud, OpenAI judge) cost money; line up a budget or use OSS/local equivalents, and keep the judge fixed so cost doesn't push you to swap it mid-run. Check dataset licenses before redistributing anything.
