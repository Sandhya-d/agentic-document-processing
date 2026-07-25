# Design Document — Agentic AI Document Processing for Artwork Catalogues

## 1. Executive Summary

This system processes an artwork catalogue PDF (about 50 pages) and
turns each artwork entry into a structured record. Simple **Agents**
coordinate the workflow, and reusable **AI Skills** do the actual work
— OCR, page classification, image extraction, metadata extraction, and
confidence scoring.

Every extracted record is shown to a human reviewer before it counts as
final. The reviewer checks it against the original page, corrects
anything wrong, and approves it. Only then is it saved in PostgreSQL.
PDFs and artwork images are kept in separate file/object storage, not
in the database itself.

The first version is kept intentionally simple: one backend service,
four agents, and about six or seven Skills. No Kubernetes, no
microservices, no automatic approval without a human checking first.

## 2. Overall Architecture

The user uploads a catalogue through a web application. The backend
API validates and stores the PDF, then starts the **Orchestrator
Agent**, which splits the document into individual pages.

The **Page Processing Agent** handles each page: it uses the Page
Classification Skill to check whether the page is an artwork entry,
gets the page text (from the PDF directly if possible, otherwise via
OCR), extracts the artwork image, and sends the text to the Metadata
Extraction Skill.

The **Validation Agent** checks the extracted fields, calculates a
confidence score, and creates a review task for a human. The reviewer
corrects and approves the record. The **Storage Agent** then saves the
approved record in PostgreSQL.

See [`diagrams/architecture-diagram.md`](../diagrams/architecture-diagram.md)
and [`diagrams/workflow-diagram.md`](../diagrams/workflow-diagram.md)
for the visual versions of this flow.

## 3. Agent Responsibilities

| Agent | Main Responsibility | Skills Used | Output |
|---|---|---|---|
| **Orchestrator Agent** | Controls the full workflow: splits the PDF, tracks each page's status, marks the catalogue complete when all pages are done | PDF Page Splitter | Page-processing tasks |
| **Page Processing Agent** | Processes one page at a time: identifies artwork pages, runs OCR if needed, extracts the image, gets metadata | Page Classification, OCR, Image Extraction, Metadata Extraction | Draft artwork record |
| **Validation Agent** | Checks the extracted data: confidence, missing fields, basic rules; sends it for human review | Confidence Scoring, Validation | Review task |
| **Storage Agent** | Saves the approved record and updates status | Database formatting | PostgreSQL record |

**Why these four and not more or fewer:** each agent maps to one clear
job in the pipeline — split the work, process a page, check the
result, save it. Agents decide *what happens next*; they don't do OCR
or extraction themselves — that's what Skills are for.

## 4. AI Skills

Skills are reusable, focused components — each one does a single task
and returns a result. The agent that calls a skill decides what to do
with that result.

- **PDF Page Splitter** — splits the catalogue into individual pages.
- **Page Classification** — decides if a page is an artwork entry,
  cover, index, ad, or other.
- **OCR** — extracts text from a scanned page image.
- **Artwork Image Extraction** — crops the main artwork image from the page.
- **Metadata Extraction** — turns page text into structured fields
  (artist, title, medium, dimensions, estimate, lot number, year,
  description), for example:
```json
  {
    "artist": "Claude Monet",
    "title": "Water Lilies",
    "medium": "Oil on canvas",
    "dimensions": "100 x 200 cm",
    "estimate_low": 500000,
    "estimate_high": 700000,
    "currency": "USD"
  }
```
- **Confidence Scoring** — scores how confident the system is in each
  field, so low-confidence values can be flagged for the reviewer.
- **Basic Validation** — simple rule checks (e.g. artist name isn't
  empty, estimate low is less than estimate high, lot number isn't a
  duplicate in the same catalogue).

See [`examples/sample-artwork-record.json`](../examples/sample-artwork-record.json)
for a full example of what a completed record looks like.

## 5. Human-in-the-Loop Review

Every draft artwork record is reviewed by a human before it's final —
there's no automatic approval in this version, since that's simpler
and safer than trusting AI output directly.

The review screen shows: the original PDF page, the extracted artwork
image, all extracted fields, the confidence score, and any validation
warnings. The reviewer can correct fields, approve the record, reject
it, or leave it for later.

**Review steps:**
1. The system creates a draft artwork record.
2. The Validation Agent checks it and calculates confidence.
3. The record goes to the review screen.
4. The reviewer compares it against the original page.
5. The reviewer corrects any mistakes.
6. The reviewer approves (or rejects) the record.
7. The Storage Agent saves the approved data.

## 6. Data Flow into PostgreSQL

Nothing gets written as a final record right away. The flow is:

```text
AI extraction → Draft record → Human review → Correction → Approval → Final PostgreSQL record
```

A record's status moves through `Draft → Review → Approved` (or
`Draft → Review → Rejected`).

**Simple table design:**

- **catalogues** — one row per uploaded PDF (filename, file path,
  status, page count, upload time).
- **catalogue_pages** — one row per page (which catalogue, page
  number, page type, extracted text, status).
- **artwork_records** — one row per artwork entry (artist, title,
  medium, dimensions, estimate, currency, lot number, image path,
  confidence score, status).
- **review_tasks** — one row per review (which artwork record,
  reviewer, status, comments, completion time).

PDFs, page images, and artwork images are stored in file/object
storage, not in PostgreSQL — only their file paths are stored in the
database.

## 7. Scalability and Reliability

Pages are processed independently, so a 50-page catalogue becomes 50
separate page tasks, and several can run at the same time. If one page
fails, it doesn't stop the rest of the catalogue.

If OCR or metadata extraction fails, the system retries; if it keeps
failing, the page goes to manual review instead. This is kept simple —
no complex event-streaming or advanced scaling setup is needed for a
50-page catalogue.

## 8. Technology Choices (Summary)

- **Backend:** Python + FastAPI — beginner-friendly, works well with
  document-processing libraries.
- **Frontend:** React (or plain HTML/CSS/JS for something simpler).
- **Database:** PostgreSQL, as required.
- **PDF processing:** PyMuPDF or PDFPlumber.
- **OCR:** Tesseract (free, simple) or a managed OCR service for
  better accuracy.
- **Metadata extraction:** an LLM with structured JSON output.
- **Image storage:** a file folder for a simple version, or S3/Blob
  storage for a cloud version.
- **Deployment:** Docker, one backend service, one database — no
  Kubernetes or microservices needed at this scale.

Full reasoning for each choice is in
[`docs/technology-choices.md`](technology-choices.md).

## 9. Assumptions and Trade-offs

Full list in [`docs/assumptions.md`](assumptions.md). The main
trade-off worth calling out: this design always requires human
approval rather than auto-approving high-confidence records. That's
slower, but much safer for a first version — a wrong artist name or
estimate in an auction record is a real problem, and it's easier to
add auto-approval later once the system has a track record than to
walk it back after a mistake.

Similarly, one modular backend service was chosen over microservices —
easier to build, explain, and maintain for a project this size, even
though microservices would scale better at a much larger volume.

## 10. Future Improvements

- Support catalogues in more than one language.
- Handle multiple artworks on a single page, or one artwork spanning
  several pages.
- Match extracted artist names against an artist reference database.
- Allow automatic approval for very high-confidence records, once the
  system has enough of a track record to trust that threshold.
- Use reviewer corrections to improve the metadata extraction Skill
  over time.
