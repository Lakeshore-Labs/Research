# A Cross-Domain Canonicalization Layer: Turning Messy Surface Forms into Clean, Queryable Keys

**One-liner:** Build a cross-domain canonicalization layer that turns messy engineering, scientific, and enterprise surface forms into clean SQL-queryable canonical keys, and, in doing so, work out the transferable recipe for *when* cheap candidate generation (embeddings/blocking) plus LLM adjudication actually beats classical entity resolution, and how to keep false merges down.

**The insight we're after:** Hard entity resolution lives in a few percent of genuinely ambiguous pairs. The generalizable result is a recipe: cheap recall-oriented candidate generation to prune the O(n²) space, then LLM adjudication on only the hard residue, plus a clear account of *where the LLM earns its cost and where classical methods already suffice*. That recipe transfers to any new domain; the corpus and metrics below are how we earn the right to claim it.

**Area:** ML / Research   •   **Difficulty:** Intermediate–Advanced

## Motivation
Real corpora refer to the same thing in many ways, and they do it across very different domains at once: engineering content (part numbers, spec variants, supplier SKUs), scientific terminology (gene/protein/compound names, instrument models), and enterprise content (acronyms, project code-names, team nicknames, "Acct" / "account" / "A/C"). When that text becomes SQL-queryable data, the surface forms have to be canonicalized (mapped to one identifier) and disambiguated (the same string can mean different things in different contexts). Skip this and `WHERE part = 'A-1024'` silently misses rows stored as "A1024 rev B", while a join on an ambiguous acronym merges two distinct things into one. This project builds the canonicalization / entity-resolution layer that sits between extracted entities and the structured store. The goal is a layer someone can actually run a corpus through and get usable keys out of.

## Research Question(s)
1. Can a hybrid pipeline (cheap embedding/blocking candidate generation + LLM judging the hard pairs) canonicalize messy cross-domain mentions more accurately than embeddings-only or LLM-only baselines, and at what cost?
2. Can context-aware disambiguation separate identical surface strings that map to different canonical IDs without over-merging distinct entities, and how does the false-merge behaviour move as you trade off recall?

## Scope
This is deliberately a hard problem: the corpus is cross-domain (engineering + science + enterprise), with all the alias and ambiguity patterns those bring together. Get the full pipeline and evaluation working end-to-end on a real, messy slice before reaching for the stretch directions. The point is a usable canonicalization layer *and* a transferable recipe for hybrid ER, not a benchmark score, and not a one-corpus result.

- In scope:
  - Canonicalization over a cross-domain corpus of already-extracted mentions: part numbers/specs, scientific terms, enterprise acronyms and code-names mixed together.
  - A hybrid pipeline: blocking/embedding candidate generation → pairwise or cluster-level LLM adjudication → clustering into canonical entities.
  - Acronym/abbreviation expansion (which is where much of the cross-domain pain lives, since the same acronym means different things in different domains).
  - Structured output: `canonical_entities` + `surface_forms` (alias) tables with foreign keys, queryable by SQL: the actual deliverable someone uses.
  - A gold canonicalization set and a metric harness (see Deliverables) covering cluster quality, false merges, cost/latency, and end-to-end SQL-query correctness.
- Out of scope (explicit non-goals):
  - PDF/table extraction and schema synthesis (proposal #6; assume clean extracted text/entities as input).
  - Cross-document coreference and ontology/taxonomy induction (stretch only).
  - Multilingual generalization (stretch only).
  - Production-scale (millions of records) infra. Target a realistic working corpus, not a stress test.

## Suggested Approach
A starting path, not prescriptive:

1. Frame the task. Entity resolution = record linkage = deduplication: decide which mentions refer to the same real-world entity, then give each cluster one canonical key. Read the classic blocking + similarity + Fellegi-Sunter framing, then the recent LLM entity-matching work. The cross-domain twist: a single similarity threshold or blocking key rarely works across part numbers, scientific terms, and acronyms at once, so expect to handle them with different signals.
2. Baselines first. Two of them:
   - Classical: normalize and block with `dedupe`, `recordlinkage`, or Splink (Fellegi-Sunter). This is the cheap floor to beat.
   - Embeddings-only: embed each surface form (sentence-transformer or an embedding API) and cluster (agglomerative / HDBSCAN / connected components over a similarity threshold).
3. Hybrid pipeline (the core contribution).
   - Candidate generation (cheap): blocking + embedding nearest-neighbours propose candidate pairs/clusters. Tune this for recall; it exists to avoid the O(n²) all-pairs LLM cost. Measure candidate recall against the gold set so you know what the LLM step can no longer fix.
   - Adjudication (precise): the LLM judges only the hard candidates ("do these refer to the same entity? expand any acronyms; use the surrounding context"). Compare the match / compare / select strategies from the COLING 2025 study. Serialize records in a Ditto-style text format.
   - Clustering: turn pairwise/group decisions into transitively consistent clusters (connected components, or correlation clustering if pairwise decisions conflict). Pick a canonical name per cluster, favouring the official or most-frequent form.
4. Disambiguation with context. When one string maps to different entities, give the LLM the mention's surrounding text. This is the entity-linking setup: candidate generation against the catalog, then context-based disambiguation, à la BLINK's bi-encoder retrieve + cross-encoder rank.
5. Bootstrapping the canonical vocabulary. Have the LLM propose canonical names and alias lists from the corpus. Route low-confidence merges to a human review queue. Persist accepted entries so new docs match against the existing catalog (incremental linking) instead of re-clustering from scratch.
6. Structured output. Emit `canonical_entities(id, canonical_name, type, …)` and `surface_forms(surface, canonical_id, context_hint, confidence)`. Fact rows carry `canonical_id` foreign keys so SQL hits one key.

Tools to consider: Splink / dedupe / recordlinkage / py_stringmatching, a sentence-transformer or embedding API, scikit-learn clustering, DuckDB or SQLite for the SQL store, and any frontier LLM for adjudication.

## Deliverables
- A runnable, usable hybrid canonicalization layer (corpus of mentions in → `canonical_entities` + `surface_forms` tables out) that someone can point at the corpus and get clean SQL-queryable keys from.
- Classical, embeddings-only, and LLM-only baselines for comparison.
- A hand-labelled gold canonicalization set drawn from the cross-domain corpus, large enough to be meaningful and deliberately seeded with the hard cases that matter (abbreviations, polysemous acronyms, code-name/official-name pairs, near-identical part numbers). Label by clustering, not by exhaustive pairwise judgement, and write down what "the same entity" means well enough that the labels are consistent. A second pass over a slice to check agreement is a good idea where disagreement is likely; treat it as a quality habit, not a mandatory protocol.
- An evaluation harness that reports cluster quality (pairwise P/R/F1, B-cubed, ARI), surfaces false merges as their own number, and measures cost/latency and end-to-end SQL-query correctness vs. a gold-canonicalized dataset.
- The recipe itself, written down as the headline deliverable: when to reach for LLM adjudication vs. when classical/embedding methods already suffice, how to set candidate-generation recall so the LLM only sees the hard residue, which adjudication strategy (match/compare/select) holds up across domains, and how to keep false merges low. State it as guidance a practitioner could apply to a new domain, with the evaluation below as the evidence behind each claim.

## Demo
Feed in a corpus seeded with messy cross-domain aliases (e.g. a part number written three ways across spec sheets; a scientific term and its abbreviation; "Acct", "account", "A/C"; a project code-name and its official name; a polysemous acronym that means two different things in two domains). Run the pipeline to produce a `canonical_entities` table + alias map. Then run two contrasting SQL queries live:
- a query that previously **missed** rows now correctly aggregates all surface forms under one canonical key;
- a query over the polysemous acronym shows the two contexts correctly resolved to **two different** canonical IDs.
Side-by-side: raw-string query results vs. canonicalized-key query results.

## Success Criteria
The guiding principle: **over-merging is worse than missing.** A false merge (two mentions that belong to different gold entities ending up in the same cluster) corrupts downstream SQL silently (a join returns rows that shouldn't be there), whereas a miss just leaves some rows un-aggregated and visible. So surface false merges as their own number (it's hidden inside F1 otherwise) and tune the pipeline to keep them low even at some cost to recall. Report the precision/recall trade-off curve so a reviewer can pick the operating point rather than reading off one number.

- Good: the hybrid pipeline beats both the embeddings-only and LLM-only baselines on cluster quality (pairwise and B-cubed F1) on the gold set, with no worse a false-merge rate at comparable recall. How tight the false-merge tolerance needs to be is a data-quality call, made against the real corpus. Don't hard-code a number.
- Done: end-to-end demo runs; cost/latency is measured; and SQL-query correctness is reported on a handful of representative queries written against the gold-canonicalized data, scored by result-set precision/recall against the gold answer (does the canonicalized query return the rows the gold key would), not by eyeballing.
- A result on a public ER benchmark (see references) reported in the same metrics, so the in-house numbers can be sanity-checked against published systems. No specific target score is claimed in advance; measure and report.

## Tech & Prerequisites
- Languages/frameworks: Python; SQL (DuckDB/SQLite); scikit-learn; an embedding model (sentence-transformers or API); at least one ER library (Splink/dedupe/recordlinkage); an LLM API.
- Background assumed: comfort with embeddings and basic clustering; precision/recall/F1; SQL and joins; reading ML papers. No prior entity-resolution experience needed; the references cover the on-ramp.

## Phases
- Explore: read the core references; get the cross-domain corpus; build the gold canonicalization set; characterize the alias/ambiguity patterns you actually see across the domains.
- Build: classical, embeddings-only, and LLM-only baselines, then the hybrid candidate-gen → LLM-adjudicate → cluster pipeline; emit the SQL tables someone can query.
- Evaluate: run the metric harness (cluster quality, false merges, cost) on gold and on a public benchmark; sweep thresholds and plot the precision/recall trade-off; score end-to-end SQL-query correctness.
- Polish: human review queue, incremental linking for new docs, demo, write-up.

## Stretch Goals
- Cross-document coreference (resolve mentions that share no string overlap, e.g. nickname ↔ official name only inferrable from context).
- Bootstrap a small controlled vocabulary / lightweight ontology from the corpus (LLM-proposed canonical names + alias hierarchies, validated by a human).
- Incremental online linking as new documents stream in, with catalog-drift detection.
- Report how cleanly performance transfers across the domains in the corpus (does engineering-tuned blocking help or hurt on scientific terms?).
- Active-learning loop to minimize human-review labels.

## Starter References
- **LLM entity matching (recent):** "Match, Compare, or Select? An Investigation of Large Language Models for Entity Matching," COLING 2025: https://aclanthology.org/2025.coling-main.8/ ; and Peeters & Bizer, "Entity Matching using Large Language Models": https://arxiv.org/abs/2310.11244
- **Classic LM-based entity matching (baseline + serialization):** Ditto, "Deep Entity Matching with Pre-Trained Language Models," VLDB 2021: https://arxiv.org/abs/2004.00584 ; code: https://github.com/megagonlabs/ditto
- **Entity linking / disambiguation:** BLINK (bi-encoder retrieve + cross-encoder rank against a KB): https://arxiv.org/abs/1911.03814 ; code: https://github.com/facebookresearch/BLINK ; survey: "Entity linking for English and other languages: a survey": https://link.springer.com/article/10.1007/s10115-023-02059-2
- **Acronym disambiguation:** AcroBERT: https://huggingface.co/Lihuchen/AcroBERT ; "Automated Extraction of Acronym-Expansion Pairs from Scientific Papers": https://arxiv.org/abs/2412.01093
- **ER tooling, benchmarks & metrics:** Awesome-Entity-Resolution (Splink, dedupe, recordlinkage, PyJedAI): https://github.com/OlivierBinette/Awesome-Entity-Resolution ; Leipzig benchmark datasets: https://dbs.uni-leipzig.de/research/projects/benchmark-datasets-for-entity-resolution ; "A Practitioner's Guide to Evaluating Entity Resolution Results" (pairwise vs. B-cubed): https://arxiv.org/abs/1509.04238

## Open Questions / Assumptions
- Decided: the corpus is cross-domain and deliberately hard: engineering (part numbers/specs), science (terminology), and enterprise (acronyms/code-names) together. We're not picking a single "easy" entity type to start with; the difficulty is the point.
- Decided: a corpus will be provided ("we'll find a corpus"), so don't block on sourcing it. Design for hard cross-domain canonicalization and slot the real data in when it arrives; if it shows up with a partial canonical catalog, use it as a head start for the gold set rather than labelling from scratch.
- Decided: over-merging is worse than missing. How tight the false-merge tolerance must be is a data-quality call made against the real corpus, not a fixed budget set in advance.
- Assumption: extracted entity mentions, with surrounding context, are available as input. Extraction and schema synthesis are out of scope (proposal #6).
- Open: budget and latency limits for LLM adjudication at the working scale. These set how hard candidate generation must prune before the LLM step.
- Unverified: no false-merge rates or per-mention costs are claimed from a citation here. Measure them on the target corpus rather than assume published numbers transfer to messy cross-domain text.
