# Design Document — Agentic AI Document Processing

## 1. Overview

This system processes a scanned artwork catalogue (approximately 50
pages, PDF) and produces structured, human-verified artwork records in
PostgreSQL. It is designed as an **agentic architecture**: a small set
of orchestrating agents coordinate the workflow, while the actual
computational work (OCR, extraction, matching, scoring) is done by
reusable, stateless **AI skills** that agents call as needed.

The central design decision throughout this document is the separation
between **orchestration** (deciding what to do, in what order, and how
to react to results — the agents) and **computation** (doing one
specific, well-defined task and returning a result — the skills).
Diagrams referenced below live in `diagrams/`:
[`architecture.md`](../diagrams/architecture.md),
[`workflow.md`](../diagrams/workflow.md),
[`data-model.md`](../diagrams/data-model.md).

## 2. System Architecture

The system has four layers (full diagram in
[`diagrams/architecture.md`](../diagrams/architecture.md)):

1. **Ingestion** — accepts the PDF, validates it, splits it into pages.
2. **Orchestration (agents)** — five agents coordinate the pipeline
   end to end.
3. **AI skills** — six stateless capabilities agents call into.
4. **Human-in-the-loop + storage** — review queue, reviewer UI,
   PostgreSQL, object storage, audit log.

Agents never perform OCR, run a model, or compute a similarity score
themselves — they call a skill for that and act on the structured
result. This means a skill can be tested, scaled, versioned, or
replaced independently of the workflow logic that calls it, and the
same skill can be reused by more than one agent (for example, the
confidence scoring skill is used by the validation agent per-record,
but could equally be called by the classification agent per-page).

## 3. Agent Responsibilities

| Agent | Responsibility | Calls into |
|---|---|---|
| **Coordinator agent** | Owns the overall run for one catalogue: tracks which pages are done, retries failed pages, and reports progress/completion. The only agent that knows about the *whole document*, not just one page. | Classification agent |
| **Classification agent** | For each page, determines page type (artwork entry vs. cover/index/ad/blank) using the layout detection skill. Routes entry pages to extraction; logs and skips non-entry pages. | Layout detection skill, Extraction agent |
| **Extraction agent** | For each artwork-entry page, orchestrates OCR, image extraction, and metadata extraction (in that order, since metadata extraction consumes OCR output), then hands the assembled candidate record to validation. | OCR skill, Image extraction skill, Metadata extraction skill, Validation agent |
| **Validation agent** | Scores the candidate record's confidence and decides how it should be routed for human review (light spot-check vs. full review). Does not decide correctness itself — it decides *how much human attention the record needs*. | Confidence scoring skill, Human review queue |
| **Persistence agent** | Writes an approved (human-confirmed) record to PostgreSQL and object storage, and writes the diff between the original extraction and the human-corrected version to the audit log. Only agent with write access to the system of record. | PostgreSQL, Object storage, Audit log |

**Why this split and not fewer/more agents:** each agent maps to a
distinct decision point in the workflow (what type of page is this? /
what does this page contain? / how much scrutiny does this need? /
is this safe to persist?). Collapsing agents (e.g. merging
classification and extraction) would mean a single agent making two
unrelated kinds of decisions, which becomes harder to test and reason
about independently. Splitting further (e.g. separate agents per skill
call within extraction) would add coordination overhead without a
corresponding decision point to justify it — those calls are a fixed
sequence, not a decision.

## 4. AI Skills and Their Interactions

| Skill | Input | Output | Notes |
|---|---|---|---|
| **Layout detection** | Page image | Page type (entry/non-entry), bounding regions (image area, text blocks) | Cheap relative to full extraction — this is why it runs first, on every page, before deciding whether the expensive skills are worth calling |
| **OCR** | Page image / text region | Raw text | Off-the-shelf OCR engine or cloud OCR API; swappable without touching any agent |
| **Image extraction** | Page image + bounding region from layout detection | Cropped artwork image, written to object storage | Independent of OCR — image and text extraction can run in parallel |
| **Metadata extraction** | Raw OCR text | Structured fields: artist name, title, medium, dimensions, estimate | The one skill most likely to use an LLM with a structured-output schema, since field boundaries in catalogue text are inconsistent across publishers |
| **Entity matching** | Extracted artist name (raw) | Resolved `artist_id` from the artist registry, or null with the raw name preserved | Fuzzy-matching against a reference table (see [entity matching considerations](#5-entity-matching-considerations) below) — deliberately a separate skill from metadata extraction, since matching logic (and its reference data) evolves independently of extraction logic |
| **Confidence scoring** | Candidate record + intermediate signals (OCR quality, field completeness, entity-match strength) | A single confidence score (or per-field scores) | Combines signals from multiple upstream skills rather than being computed inside any one of them, since it needs visibility across the whole candidate record |

**Interaction pattern:** skills are called synchronously within a
page's processing but are independent of each other's internal
implementation — the extraction agent doesn't know or care whether OCR
uses Tesseract or a cloud API, only that it receives text back in an
agreed format. This is what makes skills swappable and independently
scalable (e.g. OCR workers can scale separately from LLM-based metadata
extraction workers, since their cost/latency profiles are very
different).

## 5. Entity Matching Considerations

Artist names in scanned catalogues are inconsistently formatted
("Picasso, Pablo" vs. "Pablo Picasso" vs. "P. Picasso") and OCR
introduces additional noise (misread characters, especially in stylized
catalogue fonts). The entity matching skill is expected to combine:

- Exact/normalized lookup first (cheap, catches the common case)
- Fuzzy string matching (e.g. token-based similarity) against a
  canonical artist registry for near-misses
- A confidence-bearing result: an exact match is high confidence, a
  fuzzy match above some threshold is medium confidence, and no
  reasonable match returns null rather than a guess

An unresolved artist name is not treated as a failure — it's preserved
as `artist_name_raw` and surfaced clearly to the human reviewer, since
guessing wrong on artist attribution is a worse outcome than asking a
human to confirm.

## 6. Human-in-the-Loop Workflow

Every extracted record passes through human review before being
persisted — there is no automatic write path directly from extraction
to the database, given the cost of an error (incorrect artist
attribution or estimate in an auction record is a meaningful business
risk). What confidence *does* determine is review depth, not whether
review happens:

- **High confidence** (clean OCR, exact artist match, all fields
  populated): routed to a lightweight spot-check queue — reviewer sees
  the extracted record side-by-side with the source image and can
  approve with a single action if it looks correct.
- **Low confidence** (poor scan quality, ambiguous or unmatched artist
  name, missing fields): routed to a full review queue — reviewer sees
  the same view but with low-confidence fields visually flagged, and is
  expected to manually verify each flagged field against the source
  image before approving.

Every reviewer correction — not just the final approved value — is
logged in `review_corrections` (see
[`diagrams/data-model.md`](../diagrams/data-model.md)). This produces a
running dataset of "extraction produced X, human corrected to Y" pairs
per field, which is the natural input for periodically evaluating the
metadata extraction skill's accuracy and deciding where to invest in
improving it (e.g. if "dimensions" is corrected far more often than
"title", that's a clear signal about where the extraction prompt or
approach is weakest).

## 7. Data Flow into PostgreSQL

Only the persistence agent has write access to PostgreSQL — this is a
deliberate chokepoint. It's triggered exactly once, when a human
reviewer approves a record, and does three things atomically:

1. Writes/updates the `artworks` row (with the human-approved field
   values, not the raw extraction).
2. Confirms the associated image is durably stored in object storage
   and the `image_uri` is correct.
3. Writes one row per corrected field into `review_corrections`,
   capturing what the extraction originally produced vs. what the
   human changed it to (if anything).

Centralizing writes through one agent — rather than letting the
extraction or validation agents write directly — means the "what
actually got persisted and when" question always has one answer path
to trace, which matters for auditability in a system handling
potentially valuable, provenance-sensitive records.

## 8. Assumptions

- The PDF is a scanned or image-based catalogue (i.e. OCR is actually
  needed) rather than a fully digital-native PDF with a clean text
  layer; the design still benefits a digital-native PDF (OCR just
  becomes higher-confidence/faster) but is not built assuming one.
- One catalogue page maps to zero or one artwork entries for the
  common case. Catalogues with multiple entries per page are handled
  by the layout detection skill returning multiple regions instead of
  one, without changing the agent workflow.
- An artist reference registry (`ARTISTS` table) exists or can be
  bootstrapped; entity matching quality is bounded by this registry's
  coverage, and a genuinely new/obscure artist will correctly resolve
  to "no match" rather than a wrong match.
- Human reviewers are available in a reasonable turnaround window;
  the system is not designed for a fully unattended/autonomous mode,
  by design, given the business risk of incorrect auction records.

## 9. Trade-offs

**Always-HITL vs. confidence-based auto-approval.** This design
requires human review of every record rather than auto-approving
high-confidence ones outright. This is slower per-catalogue but avoids
the risk of a wrong high-confidence extraction (e.g. two similarly
named artists, or a plausible-looking but incorrect estimate) reaching
the database unseen. A natural evolution once the system has an
established track record would be to allow auto-approval above a very
high, empirically-validated confidence threshold, with a sampling audit
to catch drift.

**Synchronous per-page skill calls vs. a fully async pipeline.** The
workflow diagram shows a straightforward sequential-per-page flow. In
practice, pages are independent of each other and can be processed in
parallel (a worker pool processing multiple pages of the same
catalogue concurrently), which the agent design supports without
change — the coordinator agent's job of tracking per-page completion
is exactly what makes this safe to parallelize. The design doc presents
the per-page logic sequentially for clarity; the implementation would
run many of these page-level workflows concurrently.

**Separate entity matching skill vs. folding it into metadata
extraction.** Keeping matching separate from extraction means the
artist registry and matching logic (which will need tuning over time
as more catalogues are processed) can evolve independently of the
extraction model/prompt. The cost is one extra hop per record; given
that hop is cheap (a lookup/fuzzy-match against a reference table, not
another model call), this trade-off favors separation.

## 10. Technology Choices

These are illustrative choices consistent with the architecture, not
claimed to be the only valid ones:

| Component | Suggested technology | Why |
|---|---|---|
| Agent orchestration | A workflow/state-machine framework (e.g. an agent-orchestration library, or a durable-execution engine like Temporal) | Needs retry, per-page state tracking, and observability across a multi-step, partially-parallel pipeline — hand-rolled orchestration reinvents this poorly |
| OCR | Cloud OCR API (e.g. AWS Textract / Google Document AI) or Tesseract for a self-hosted option | Mature, well-tested for scanned documents; swappable behind the OCR skill interface |
| Metadata extraction | LLM with structured/JSON-schema output | Catalogue text formatting varies enough across publishers that rule-based field extraction would need constant per-catalogue tuning; an LLM with a fixed output schema generalizes better |
| Entity matching | Fuzzy string matching (e.g. token-based similarity) against a PostgreSQL-backed artist registry | Lightweight, explainable, and fast; a full embedding-based approach is plausible future work but adds complexity not clearly justified at this scale |
| Object storage | S3-compatible storage | Standard, decouples image storage from the relational database |
| Database | PostgreSQL | Structured, relational data with clear referential integrity needs (artworks → artists, artworks → pages) |
| Reviewer UI | Lightweight internal web app | Only needs to show an extracted record + source image + edit form; doesn't warrant a heavy framework |

## 11. Scalability and Maintainability

- **Skills scale independently of agents and of each other.** OCR and
  LLM-based metadata extraction have very different cost/latency
  profiles; keeping them as separate skills means their worker pools
  can be sized independently rather than coupled to one monolithic
  "extraction service."
- **Pages within a catalogue parallelize naturally** (see trade-offs,
  above), and catalogues themselves parallelize across each other with
  no shared state beyond the database.
- **Skills are independently testable and versionable.** Because a
  skill is a pure function (input in, structured output out, no
  workflow state), it can have its own test suite and be upgraded
  (e.g. swap OCR engines, tune the entity-matching threshold) without
  touching agent code.
- **The audit log is the feedback loop for maintainability.** Rather
  than maintainability depending on manually noticing extraction
  quality problems, `review_corrections` gives a structured, queryable
  record of exactly where and how often extraction needs human
  correction, per field — the natural basis for prioritizing future
  improvement work.

## 12. Future Improvements

- Confidence-threshold auto-approval once track record supports it,
  with ongoing sampling audit.
- Active-learning loop: periodically retrain/re-prompt the metadata
  extraction skill using accumulated `review_corrections` data.
- Batch-level analytics dashboard (per-catalogue completion rate,
  average confidence, most-corrected fields) built directly on the
  audit log and artworks tables.
- Multi-language OCR/extraction support for non-English catalogues.

## 13. Open Questions / Assumptions to Validate

The design above makes several assumptions that would need validating
against the actual business context before implementation:

- **Scale ceiling.** Is ~50 pages the typical catalogue size, or could
  it occasionally be 500+? This affects whether per-page parallelism
  alone is sufficient or whether the ingestion layer needs chunked/
  streaming processing for very large documents.
- **Review capacity and timing.** How many reviewers are available, and
  is review expected in real-time as pages complete, or as a batch
  after the catalogue finishes processing? This affects whether the
  review queue needs to support live partial results.
- **Rejected record handling.** Should a rejected record be discarded,
  or retained for reprocessing / audit reference?
- **Artist registry provenance.** Does a reference artist database
  already exist (internal, or an external source like ULAN), or does
  this system need to bootstrap one from scratch? Registry coverage
  directly bounds entity matching quality.
- **Format and language variability.** Do catalogues follow a broadly
  consistent layout, or do they vary significantly by publisher? Is
  English-only OCR/extraction sufficient, or is multi-language support
  needed from day one?
- **Source document type.** Always scanned/image-based PDFs, or
  sometimes digital-native PDFs with a clean text layer already
  present (which would let OCR be skipped or used only as a fallback)?
- **Data retention and rights.** Are there copyright/licensing
  constraints on storing extracted artwork images long-term, and are
  there retention requirements (e.g. auction records kept for N years)?
- **Downstream integration.** Is PostgreSQL the final destination, or
  does the approved-records data need to feed an existing inventory/
  cataloguing system?
