# Assumptions, Limitations, and Open Questions

## Assumptions

- The catalogue contains approximately 50 pages.
- The PDF is mainly in English.
- Most artwork entries fit on one page (one page = one main artwork).
- Human review is required before any record is treated as final.
- The system processes one catalogue at a time in this first version.
- The uploaded PDF is not password-protected.
- PostgreSQL is available and set up.
- Artwork images are stored separately from the database (file/object
  storage), with only the file path saved in PostgreSQL.

## Limitations (What This First Version Does Not Do)

- No Kubernetes or multiple microservices — one backend service is enough.
- No automatic model training or fine-tuning.
- No artist-matching against an external artist database/knowledge graph.
- No multilingual support.
- No fully automatic approval — every record needs a human to sign off.
- No real-time processing guarantees — background processing is enough.
- No complex event-streaming systems.
- No advanced analytics dashboards.

These are reasonable to skip for a first version aimed at one catalogue
at a time; they're listed here so it's clear they were intentionally
left out, not overlooked.

## Open Questions

A few things worth checking with whoever owns this system before
building it for real:

- Is 50 pages the typical size, or could catalogues sometimes be much
  larger?
- How many human reviewers will actually be available, and how quickly
  do they need to turn reviews around?
- Does an artist reference list already exist somewhere, or would one
  need to be built from scratch for future artist-matching?
- Are catalogues always scanned/image-based, or sometimes digital PDFs
  with clean text already in them (which would mean OCR isn't always
  needed)?
- Is there a data retention requirement for the original PDFs and
  extracted images (e.g. keep for N years)?
