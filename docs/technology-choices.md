# Technology Choices

## Frontend

**Chosen: React**
Easy to build upload and review screens with; common technology with
good community support.

*Simpler alternative:* plain HTML, CSS, and JavaScript, if React feels
like too much for a first version.

## Backend

**Chosen: Python + FastAPI**
Easy to understand, works well with AI and document-processing
libraries, supports REST APIs, and automatically generates API
documentation.

## Database

**Chosen: PostgreSQL**
Required by the exercise; well suited to structured artwork records
and the relationships between catalogues, pages, and artwork entries.

## PDF Processing

**Chosen: PyMuPDF or PDFPlumber**
Both can split PDFs into pages, extract embedded text, and render
pages as images. Either is fine for a first version.

## OCR

**Chosen (simple option): Tesseract OCR**
Free and easy to run locally — good enough for a beginner-level
system.

*Alternative:* a managed OCR service (AWS Textract, Azure Document
Intelligence, Google Document AI) for better accuracy on more complex
layouts, at some usage cost.

## Metadata Extraction

**Chosen: an LLM with structured JSON output**
Catalogue layouts vary a lot between publishers, so a flexible
LLM-based approach handles that better than fixed extraction rules —
as long as a human still checks the result (see the design document's
human-in-the-loop section).

## Image Storage

**Chosen: simple file storage folder** for a local/beginner version.
*Alternative:* Amazon S3, Azure Blob Storage, or Google Cloud Storage
for a cloud-hosted version.

## Deployment

**Chosen: Docker, one backend service, one PostgreSQL database.**
No Kubernetes or microservices — not needed at this scale, and would
add complexity without a clear benefit for a first version processing
one catalogue at a time.

## Trade-offs Considered

**Tesseract vs. managed OCR** — Tesseract is free and simple but may
be less accurate; managed OCR handles complex layouts better but costs
money to run. Either is a reasonable starting choice.

**LLM extraction vs. fixed rules** — an LLM adapts to varying catalogue
formats more easily, but its output isn't always correct, which is
exactly why every record goes through human review rather than being
trusted directly.

**One backend service vs. microservices** — one service is easier to
build, explain, and maintain for a project this size. Microservices
would scale better at much higher volume, but that's not a problem
this first version needs to solve.
