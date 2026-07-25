# Agentic AI Document Processing — Artwork Catalogues

A beginner-friendly system design for processing an artwork catalogue
PDF (~50 pages) into structured, human-reviewed records stored in
PostgreSQL.

**This is a system design exercise — no code is included.**

## Project Overview

An artwork catalogue is uploaded as a PDF. The system splits it into
pages, figures out which pages contain artwork entries, extracts the
artwork image and metadata (artist, title, medium, dimensions,
estimate, etc.), sends everything to a human reviewer for
correction/approval, and only then saves the approved record in
PostgreSQL.

The design uses four simple **Agents** to coordinate the workflow and
about six or seven reusable **AI Skills** to do the actual work (OCR,
classification, extraction, scoring). Every record is reviewed by a
human before it's final — nothing is auto-approved in this version.

## Main System Flow

```text
User uploads PDF
        ↓
System stores the PDF
        ↓
PDF is divided into pages
        ↓
Each page is checked
        ↓
Artwork pages are identified
        ↓
Text and artwork images are extracted
        ↓
Metadata is created
        ↓
Confidence score is calculated
        ↓
Human reviews and corrects the data
        ↓
Approved record is saved in PostgreSQL
```

## Contents

- [`docs/design-document.md`](docs/design-document.md) — the full
  design document: architecture, agent responsibilities, AI Skills,
  human-in-the-loop review, data flow into PostgreSQL, scalability,
  and technology choices.
- [`docs/assumptions.md`](docs/assumptions.md) — assumptions,
  limitations, and open questions.
- [`docs/technology-choices.md`](docs/technology-choices.md) —
  chosen technologies and why, with alternatives and trade-offs.
- [`diagrams/architecture-diagram.md`](diagrams/architecture-diagram.md)
  — high-level architecture diagram.
- [`diagrams/workflow-diagram.md`](diagrams/workflow-diagram.md) —
  end-to-end workflow diagram.
- [`examples/sample-artwork-record.json`](examples/sample-artwork-record.json)
  — an example of a completed, approved artwork record.

All diagrams are written in Mermaid and render automatically when
viewed on GitHub.

## Agents (Coordinate the Workflow)

- **Orchestrator Agent** — splits the PDF into pages, tracks progress,
  marks the catalogue complete.
- **Page Processing Agent** — processes one page at a time: classifies
  it, runs OCR if needed, extracts the image, gets the metadata.
- **Validation Agent** — checks confidence and basic rules, sends the
  record for human review.
- **Storage Agent** — saves the approved record to PostgreSQL.

## AI Skills (Do the Actual Work)

PDF Page Splitter, Page Classification, OCR, Artwork Image Extraction,
Metadata Extraction, Confidence Scoring, Basic Validation.

Agents decide *what happens next*; Skills each do *one specific task*
and return a result. That separation is the core idea behind this
design.

## Keeping It Simple

This is intentionally a beginner-level design. It does **not** include
Kubernetes, multiple microservices, automatic model training, artist
knowledge graphs, multilingual support, fully automatic approval, or
advanced analytics — all of that is listed as future improvement, not
part of this first version.
