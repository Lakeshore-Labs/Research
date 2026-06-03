# Agent-Driven Customizable UIs on a Non-Breaking Declarative Substrate

**One-liner:** An LLM agent that composes multiple disparate APIs (Facebook / Instagram / LinkedIn) into a single, user-customizable app on the fly, and that app cannot white-screen, by construction. The capability is multi-API composition you can drive and customize in natural language; the guarantee is a declarative, Elm-Architecture / Tar-Pit-inspired substrate on which no agent output can crash the page.

**Area:** Research (with heavy SW build)   •   **Difficulty:** Advanced

## Motivation
The capability we want: tell an agent "merge my three social feeds into one app I can re-sort and re-layout," and get a real, usable interface composed from disparate APIs, with no hand-written UI. An agent that writes raw React/JS to do this is flexible but brittle. It hallucinates components, renames fields, and crashes at runtime, which kills the capability the moment a user trusts it. The way out is to stop letting the agent emit arbitrary code and instead have it author against a declarative substrate that carries a hard guarantee: no agent output can white-screen the app. The project pairs the two (the composition capability and the non-breaking substrate that makes it safe to ship), and the central claim is that the guarantee holds *by construction*, not by after-the-fact luck.

The guarantee, stated precisely so it can be falsified, is in the next section; the rest of this section motivates the substrate that delivers it.

Two ideas ground the design. The Elm Architecture (Model-View-Update with pure views) is the canonical example of a UI runtime that, in practice, does not throw at runtime. "Out of the Tar Pit" (Moseley & Marks, 2006) blames mutable state for most accidental complexity and proposes Functional Relational Programming: essential state as relations, essential logic as pure functions, accidental state and control pushed to the edges. Together they point at a substrate where the view is a pure function of relational state, the agent authors against a constrained vocabulary, and the only impurity is effects at the edge.

The point of the project is a working artifact: a substrate plus an actual app a person can open, drive with natural language, and customize live, not a paper study. Multi-API composition is the concrete thing it has to do well.

## The Guarantee (what "cannot break by construction" means)
The reviewer flagged this as soft, so here it is pinned down to something falsifiable.

The agent never emits executable code. It emits a **view spec**: a value in a typed, closed UI DSL, a fixed set of component and effect node types, each with a typed schema. The runtime is a fixed, audited interpreter that renders only those node types. The guarantee is:

> **G:** For every value `s` the agent can emit, `render(s)` either (a) produces a valid UI, or (b) renders a contained error region in place of the offending subtree while every sibling region keeps rendering. There is no `s` for which `render(s)` unmounts the root or leaves a blank page.

This is enforced by three mechanisms, in order:
1. **Closed vocabulary.** The DSL admits only interpreter-known node types; there is no "raw HTML/JS" escape hatch. An unknown node type is a parse failure, not a render.
2. **Validate-before-render.** Every spec is checked against the DSL schema before it reaches the interpreter. A spec that fails validation never renders; it is returned to the agent as a typed error to retry against (the validate-and-repair loop). The user-facing app only ever renders specs that already passed validation.
3. **Region isolation at render.** Each rendered region is wrapped (React error boundary plus a typed fallback) so that a runtime fault inside one region (e.g. an effect returning malformed data the view didn't expect) degrades that region to a contained error state and leaves the root and siblings mounted.

**Scope: what G covers and does not.** In scope: agent-authored specs (correct, wrong, or adversarial) and malformed/perturbed API responses flowing through the substrate; the claim is total over these. Explicitly out of scope of the formal guarantee: bugs in the fixed interpreter itself, the validator, or third-party libraries; browser/runtime OOM or crashes below the substrate. Those are treated as a small, audited trusted base (the analog of Elm's compiler being trusted), not as part of the surface the agent or APIs can reach. The honest version of the claim is therefore: *no agent output and no API response can break the app, given a correct interpreter.* That is the strong, definable property; we commit to it and do not weaken it to "validate and repair usually catches it." Validate-and-repair is how we improve *spec acceptance rate*, not how we achieve non-breakage. Non-breakage comes from the closed vocabulary plus region isolation, which hold even when repair gives up.

The empirical test of G is direct: enumerate and fuzz specs (including type-valid-but-semantically-hostile ones) and API payloads, and show no input drives the root to unmount. A single white-screen on an in-scope input falsifies G.

## Research Question(s)
1. Can an LLM agent compose 2–3 disparate read-only APIs into one normalized data model and a unified, customizable UI from a single natural-language request?
2. Can the substrate be designed to satisfy guarantee **G** (above), non-breakage by construction over all agent specs and API responses, rather than catching breakage after the fact, and how expressive can the DSL be while G still holds?
3. How wide a range of *user* customizations (layout, sort/filter, behavior) can the substrate absorb without regenerating or breaking the running app?

## Scope
In scope:
- A small declarative UI substrate: relational state, derived (pure) values, an MVU update loop, and a validated view spec the agent emits (a typed schema/DSL, not arbitrary JS).
- Schema-constrained generation plus a validate-and-repair loop, so invalid specs come back to the agent as errors it retries against.
- A multi-API composition demo: the agent maps 2–3 mock/read-only social-style feeds into one normalized model and composes a unified UI.
- User-side customization (declarative edits to layout, sort, filter, behavior) without crashing.
- A way to exercise the substrate against malformed and adversarial inputs to show that totality holds.

Out of scope:
- Live OAuth against Facebook/Instagram/LinkedIn (mock/read-only fixtures instead; see Open Questions). Any write actions to those APIs.
- Multi-user/realtime collaboration, production auth, rate-limit infrastructure.
- New model architectures. This uses off-the-shelf LLMs with structured output and tool calling.

## Suggested Approach
1. Design the substrate around the non-breaking goal. Build the doc model from first principles: `state` (relational tables), `derived` (pure signals), `view` (pure function of state + derived), and event-sourced messages + handlers. The Elm Architecture and the Tar Pit paper are the references for how to shape this.
2. Make the view total by construction. This is the central design decision. Instead of letting the agent emit arbitrary JSX, give it a typed, validated component DSL: a fixed vocabulary of components and effects, validated against a schema before render, with every rendered region wrapped so a bad subtree degrades to a contained error state rather than unmounting the app. The target is that no agent output, however wrong, can produce a white-screen crash. The Portal UX Agent's approach (typed composition JSON validated against a component schema, a renderer that instantiates only vetted components) is a working precedent for this style.
3. Keep effects the only impurity. Network access stays behind a declarative HTTP effect source, so all external I/O lives in the imperative shell and the view stays pure.
4. Ingest multiple APIs. Define 2–3 mock APIs (JSON fixtures or a tiny local server) shaped like social feeds. The agent declares a normalized `posts` table (author, source, text, ts, media) plus per-source relations, then declares effects to fetch and populate state.
5. Compose, then customize. The agent emits the unified view. User customizations dispatch as messages into `state.ui` (sort/filter/layout), which keeps the view referentially transparent.
6. Stress it and ship it. Exercise the substrate with malformed and perturbed specs to confirm breakage stays impossible, then get the app to a state a person can actually use.

## Deliverables
- A runnable, usable app where a natural-language request produces a working combined app the user can drive and customize.
- The declarative substrate itself: relational state, pure derived values, MVU loop, and the typed view-spec/DSL with its validate-and-repair layer.
- 2–3 mock/read-only "social feed" APIs and a normalized data model.
- A way to demonstrate non-breakage under bad/adversarial input.
- A short write-up of how the substrate achieves totality and what range of customization it supports, grounded in Elm and the Tar Pit paper.

## Demo
One natural-language request, e.g. *"Make me one feed that merges my Facebook, Instagram, and LinkedIn posts, newest first, with a toggle to filter by source and a card layout."* The agent declares a normalized table, wires effects to the three mock APIs, populates state, and emits a view. The combined app renders. The user then live-customizes it ("group by source", "switch to compact list", "hide LinkedIn") through UI controls or prompts, and the running app updates without regenerating or breaking. To close, feed the substrate deliberately broken specs and bad API responses and show the app staying alive.

## Success Criteria
The design target is guarantee **G**: over all agent specs and API responses, the app either renders a valid UI or shows a contained error state for the failed region while the rest keeps working, and the root never unmounts. This is enforced by closed vocabulary plus region isolation (non-breakage) with validate-and-repair layered on top to raise acceptance, not to provide the guarantee. The other goals are qualitative and demonstrated rather than scored to a fixed bar:

- Non-breakage (G): under malformed, empty, oversized, wrong-shape, adversarial, and perturbed inputs (agent specs and API payloads alike), the root stays mounted; no white-screen. This is the headline claim, falsified by a single in-scope white-screen, and the thing to demonstrate convincingly.
- Multi-API composition: a single natural-language request spanning 2–3 APIs produces a working, sensible combined app.
- Customization range: show the breadth of user edits (sort, filter, layout, behavior) the substrate absorbs live, and be honest about which classes it cannot express.

## Tech & Prerequisites
- A reasonable stack: a fast JS runtime (e.g. Bun), Vite + React, a fine-grained reactivity library (e.g. signals) for derived state, a relational/dataframe library (e.g. arquero) for the relational ops, and an LLM chassis for the agent/chat loop with structured output and tool calling. None of this is load-bearing. Pick what gets a usable system up fastest.
- Background: TypeScript and React; LLM tool-calling and structured (schema-constrained) output; functional-programming and relational/data-model intuition; familiarity with MVU.

## Phases
- Explore: read the Tar Pit paper and the Elm guide; sketch the doc model and the view DSL with the non-breaking goal driving every decision.
- Build: the substrate (relational state, derived values, MVU loop); the typed view-spec and validate-and-repair layer; 2–3 mock APIs and the normalized model; agent prompts and tools for multi-API composition; the user-customization path through `state.ui`.
- Harden: drive malformed and perturbed inputs through the substrate to confirm totality holds; fix any path that can still take down the page.
- Polish: get the app to genuinely usable, then write up how totality is achieved and what customization the substrate supports.

## Stretch Goals
- One real read-only API behind OAuth (resolve auth realities first).
- Constrained *decoding* (grammar or JSON-schema-masked generation) so invalid specs are impossible to emit, not just caught after the fact.
- Time-travel/undo over the event-sourced message log as a customization safety net.
- Agent-driven layout search: propose several layouts, user picks.
- Capability-based effect permissions per API.

## Starter References
- Out of the Tar Pit (Moseley & Marks, 2006), canonical PDF: https://curtclifton.net/papers/MoseleyMarks06a.pdf
- The Elm Architecture, official guide (Model/View/Update, pure views): https://guide.elm-lang.org/architecture/. The "no runtime errors in practice" claim is on the guide home page: https://guide.elm-lang.org/
- Vercel AI SDK, Generative User Interfaces (tool-call-driven component rendering): https://ai-sdk.dev/docs/ai-sdk-ui/generative-user-interfaces. Background announcement: https://vercel.com/blog/ai-sdk-3-generative-ui
- Portal UX Agent: A Plug-and-Play Engine for Rendering UIs from Natural Language Specifications (Li, Jiang & Selvaraj, 2025); typed composition validated against a component schema, deterministic renderer: https://arxiv.org/abs/2511.00843

## Open Questions / Assumptions
- Totality by construction: guarantee G is the bet, scoped to agent specs and API responses given a correct interpreter (see The Guarantee). The open question is not whether G can be stated (it is stated), but how expressive the DSL can be while G still holds: every richer node type widens the surface the region-isolation fallback has to cover, and the expressiveness/airtightness tradeoff is part of what the project explores. Where it lands is a finding, not a foregone conclusion.
- API-auth realities: Facebook, Instagram, and LinkedIn all require OAuth and app review and enforce strict rate limits. Live multi-account access is impractical for a student project and likely breaks ToS for arbitrary recomposition. Assumption: the tractable core uses mock or read-only fixtures shaped like these feeds. One real read-only API is a stretch goal only.
- Normalization fidelity: the feeds have different schemas and semantics, so collapsing them into one `posts` table loses information. How much loss is acceptable is itself a design question to surface as the app takes shape.
