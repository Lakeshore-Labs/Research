# PDF-to-SQL: Querying Technical PDFs as Relational Data

**One-liner:** Drop a folder of messy technical PDFs from any domain (multi-page tables, merged cells, charts, units, footnotes) and get back a clean, populated SQL database you can actually query, and, along the way, produce a transferable map of *where document-AI extraction actually breaks* across domains and document types.

**Area:** ML / Research   •   **Difficulty:** Intermediate–Advanced

## Motivation
Engineers and scientists spend hours re-keying numbers out of datasheets, lab reports, and financial filings because the data is locked inside PDF tables. OCR gives you text, not data you can join and aggregate. The headline here is two things at once. First, the **capability**: a tool someone would actually reach for. Drop in PDFs from whatever domain, get back a queryable database. Second, a **generalizable insight** that outlasts the tool: a transferable map of *where document-AI extraction breaks* (spanning cells, footnoted units, multi-page tables, borderless layouts, rotated tables) and which failures are domain-specific versus universal. Anyone building a document→data pipeline hits these same walls; a clear-eyed account of which extractor fails on what, and why, is useful well beyond our gold set. (On hard tables, specialized table-structure models still tend to beat general vision-LLMs; see the benchmark pointers below, and verify them on our own data.)

## Research Question(s)
1. **(Capability)** Can we build a domain-agnostic tool that takes unseen technical PDFs and returns a sensible, normalized, queryable SQL database (types, units, keys, loaded data) good enough that analytical questions are actually answerable?
2. **(Insight)** *Where does document-AI extraction break, and is it predictable?* Across extractors and across domains/document types, which failure modes recur (spanning cells, multi-row headers, multi-page tables, footnoted units, borderless cells, rotated tables), which are universal versus domain-specific, and what is the transferable map a pipeline builder should plan around?
3. End-to-end, on a gold set spanning multiple domains, how often does the pipeline answer analytical questions correctly, and, just as important, where does it break (extraction vs. schema vs. SQL)?

## Scope
- **In scope (core):** PDF → table extraction → schema synthesis → load into SQLite/DuckDB → run SQL, delivered as a tool that runs end to end on PDFs it has not seen. Domain-agnostic by design: it should handle technical, scientific, engineering, and financial PDFs without being tuned to one document type. Focus on born-digital and clean-scanned PDFs with tables. Build an evaluation harness with a gold set drawn from several domains.
- **In scope (lighter):** Unit normalization and type inference; a thin NL-to-SQL layer for the demo.
- **Out of scope:** Training/fine-tuning new extraction models from scratch; production-grade ingestion at scale; arbitrary handwriting; equation *semantics* (capture equations as text/LaTeX only, do not solve them); building a polished UI beyond a minimal query console.
- **Stretch (optional):** Charts/figures → data. This is the weakest link; keep it a separate, bounded experiment and do not let it into the core.

## Suggested Approach
A starting path, not prescriptive. Get a working tool (tables → schema → SQL) running end to end first; add charts only if time remains.

1. **Extraction (survey, then pick a primary).** Compare a learning-based document parser, a rule-based extractor, and a vision-LLM on the gold set:
   - Docling (IBM, MIT-licensed): layout plus the TableFormer table-structure model; handles complex tables and exports DataFrame/CSV/structured JSON. Reasonable default primary.
   - Table Transformer (TATR) on the PubTables-1M lineage: a transparent table-structure baseline, useful for the eval comparison.
   - Rule-based fallback: Camelot (lattice/stream) or pdfplumber for bordered, well-structured tables.
   - **Vision-LLM** (a multimodal model reading the page image, via a cloud API). Flexible on odd layouts but expect it to lag specialized models on hard tables and to invent cells. Use it as a comparison point and for pages the others choke on. Third-party leaderboards have reported general vision-LLMs trailing specialized table models on RD-TableBench by a wide margin; treat those as pointers to check, not facts to quote, since we have not verified the methodology or run them ourselves.
   - Record where each tool fails: spanning/merged cells, multi-row headers, rotated tables, tables split across pages, borderless (color-delimited) cells, footnote-attached units. Run each extractor twice on the same page to confirm output is deterministic; if a tool varies run-to-run, pin its config or note it as a reliability cost.
2. **Schema synthesis.** Infer a relational schema from the extracted tables. A reasonable recipe: per-table type inference (int/float/date/text/categorical) on cell values; unit normalization (parse "kW", "MPa", "°C", and footnote-defined units into a canonical unit plus numeric value); an LLM that proposes table/column names, primary keys, and cross-table relationships from headers and sample rows; then a programmatic check on that proposal (does it load, do FKs resolve, are types consistent?) with a repair loop. SI-LLM is a reference point for header+value schema inference. De-duplicate tables that recur across pages into one. Run the LLM at temperature 0 and cache its output keyed on the input, so re-running the pipeline does not silently change the schema and corrupt the evaluation.
3. **Load & query.** Land data in **DuckDB** (columnar, fast for analytics, zero-copy from DataFrames) or **SQLite** (simpler, ubiquitous). Expose raw SQL plus a thin NL-to-SQL layer (LLM given the synthesized schema). NL-to-SQL is itself unreliable: published systems report well short of perfect execution accuracy on the BIRD benchmark, so raw SQL stays the source of truth.
4. **Evaluation (the crux; it produces the failure-mode map).** The numbers below are the *means* to the insight, not the deliverable: TEDS/GriTS, end-to-end QA, and error attribution exist to populate the transferable map of where extraction breaks across extractors, domains, and document types. Two layers, on two different document populations. Say so plainly, because the cell metric and the QA metric do not run on the same PDFs:
   - *Cell-level extraction* vs. ground truth using **TEDS** (tree-edit-distance similarity, de facto standard) and **GriTS** (grid table similarity, from the TATR authors). Run these on a slice of **PubTables-1M** / **FinTabNet** / **SciTSR**, which have public cell labels. These cover scientific and financial tables; the second layer exists precisely because public labels won't span every domain the tool should handle.
   - *End-to-end QA* on a hand-built gold set: analytical questions over a handful of source PDFs drawn from several domains, scored on answer correctness, with each wrong answer attributed to extraction, schema, or SQL. Write the gold answers by reading the source PDF directly, independent of the pipeline output, so the ground truth is not contaminated by the system being tested. The aim is to learn where the tool breaks, not to hit a target score.

## Deliverables
- **The capability:** a usable tool. Point it at one or more PDFs from any domain and get back a populated DuckDB/SQLite database plus the synthesized schema (DDL), ready to query.
- **The insight:** a transferable failure-mode map, which extractors break on which document features (spanning cells, footnoted units, multi-page tables, borderless/rotated layouts, ...), tagged as universal vs. domain-specific, so a pipeline builder in any domain can predict where they'll get burned. Backed by the tool comparison (Docling vs. TATR vs. rule-based vs. vision-LLM) with TEDS/GriTS numbers.
- A schema-synthesis module: type inference, unit normalization, and an LLM-proposed schema that is validated programmatically.
- An evaluation harness and a multi-domain gold set, reporting end-to-end accuracy with errors attributed to extraction, schema, or SQL, feeding the failure-mode map above.
- A short write-up: the failure-mode map, what works, what doesn't, and a recommended default stack.

## Demo
Drop a technical PDF (e.g. an equipment datasheet or a financial filing) into the tool. It prints the synthesized schema (CREATE TABLE … DDL with inferred types/units), loads the data, and then:
1. Runs 3–5 prewritten example SQL queries (e.g. "average value of column X grouped by category Y", a cross-table join).
2. Accepts one free-text question, shows the generated SQL and the answer.
3. Shows the matching cells highlighted on the source page for one query (provenance), so a reviewer can spot-check correctness.

A good outcome is a tool that extracts tables accurately, synthesizes a sensible queryable schema, and reports honestly where it breaks. Concrete goals, not hard gates:
- **Extraction:** report TEDS and GriTS on the public ground-truth slice, broken out for the hard subset (spanning cells, multi-row headers, multi-page). The chosen primary extractor should clearly outperform the rule-based baseline on that hard subset; measure the margin rather than predeclare it.
- **Schema:** most synthesized tables should load and query without hand-editing, with types and units largely correct on a manually spot-checked sample. For schema quality, use a simple rubric (keys present, no obviously wrong FKs, no duplicate tables) and, if two reviewers score it, report their agreement.
- **End-to-end:** report the fraction of gold questions answered correctly, with every error attributed to extraction, schema, or SQL. The credible breakdown of where it fails matters more than the headline number: don't tune toward a target, report what you actually get.

## Tech & Prerequisites
- **Language:** Python.
- **Libraries:** Docling, Table Transformer / PubTables-1M tooling, Camelot/pdfplumber, DuckDB and/or SQLite, pandas/polars, an LLM API for schema synthesis + NL-to-SQL.
- **Background:** comfortable with SQL and relational modeling (normalization, keys); basic ML/CV familiarity (object detection, OCR concepts) and LLM prompting; ability to read a research paper. No model training required for the core.

## Phases
- **Explore:** Collect a handful of representative PDFs spanning several domains + label the gold set. Run Docling, TATR, Camelot/pdfplumber, and a vision-LLM on them; record TEDS/GriTS and a failure-mode table. Pick a primary extractor.
- **Build:** Extraction → type inference → unit normalization → LLM schema proposal → programmatic validation/repair → load into DuckDB/SQLite. Add raw SQL + thin NL-to-SQL.
- **Evaluate:** Run the end-to-end QA harness; attribute errors; tighten the weakest stage.
- **Polish:** Demo, provenance highlighting, write-up, recommended stack.

## Stretch Goals
- Charts/figures → data: DePlot-style plot-to-table or vision-LLM chart reading, measured with RMS/RNSS. Chart extraction is much harder than tables; report the numbers honestly.
- Multi-PDF corpus mode: merge heterogeneous tables across documents into a shared schema.
- Equation capture as LaTeX with table-cell linkage.
- Per-cell/column confidence scores to flag low-trust data for human review.

## Starter References
- **Docling technical report + repo** (IBM, doc parser w/ TableFormer): https://arxiv.org/abs/2501.17887, repo: https://github.com/docling-project/docling
- **Table Transformer + PubTables-1M** (table-structure dataset/model + GriTS metric): https://github.com/microsoft/table-transformer, paper: https://arxiv.org/abs/2110.00061
- **DePlot** (chart-to-table translation, for the charts stretch goal): https://arxiv.org/abs/2212.10505
- **GriTS metric** (table-structure evaluation): https://arxiv.org/abs/2203.12555
- **SI-LLM: Schema Inference for Tabular Data Repositories Using Large Language Models** (schema synthesis): https://arxiv.org/abs/2509.04632
- **A Comparative Study of PDF Parsing Tools Across Document Categories** (tool-selection grounding): https://arxiv.org/abs/2410.09871

## Open Questions / Assumptions
- **Domain-agnostic is the requirement, not an open question.** The tool should handle technical/scientific/engineering/financial PDFs generally rather than being tuned to one document type. The gold/eval set draws from several domains so the breadth is actually exercised.
- **A cloud LLM API is assumed and allowed** for schema synthesis and NL-to-SQL (and as the vision-LLM comparison point). Record the model and date used.
- Gold-set ground truth is the largest setup cost. PubTables-1M/FinTabNet/SciTSR give cell-level labels but are scientific/financial, so they only cover part of the domain range; the multi-domain end-to-end gold set has to be hand-labeled.
- The cell-level metric (public datasets) and the end-to-end metric (hand-built gold set) run on different document populations. Strong TEDS/GriTS on public tables does not guarantee end-to-end accuracy across domains. Both are reported, and the second is the one that matters for the use case.
- LLM stages are nondeterministic by default. Pin temperature 0 and cache outputs keyed on the input so the schema and SQL don't drift between runs and corrupt the evaluation. Even pinned, identical reproducibility across model versions is not guaranteed.
- Schema inference from headers and sample rows is ambiguous when headers are terse or units live only in footnotes; the LLM proposal will sometimes pick wrong keys or merge tables it shouldn't. Programmatic validation catches load/FK/type failures but not semantic mistakes, so the manual schema check still matters.
- Charts are unreliable. DePlot's headline numbers are on benchmark plots; real charts (dual axes, stacked, legends, noise) are much harder. Charts stay a stretch goal to keep scope from blowing up.
- Any benchmark figures encountered for these tools (Docling complex-table accuracy, RD-TableBench, BIRD) are third-party/vendor numbers. Treat them as directional and reproduce on our own data before quoting any of them.
