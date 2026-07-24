# Data Model — PostgreSQL

The schema separates four concerns: the source document (`catalogues`,
`pages`), the extracted business records (`artworks`), reference/master
data used for entity matching (`artists`), and the audit trail of human
corrections (`review_corrections`). Splitting artworks from artists
(rather than storing the artist name as a free-text field) is what
makes entity matching meaningful — extracted names are matched against
a canonical registry rather than trusted as-is.

```mermaid
erDiagram
    CATALOGUES ||--o{ PAGES : contains
    PAGES ||--o| ARTWORKS : "yields (if entry page)"
    ARTISTS ||--o{ ARTWORKS : "matched to"
    ARTWORKS ||--o{ REVIEW_CORRECTIONS : "corrected via"

    CATALOGUES {
        uuid catalogue_id PK
        string filename
        int total_pages
        string status
        timestamp uploaded_at
    }

    PAGES {
        uuid page_id PK
        uuid catalogue_id FK
        int page_number
        string page_type
        float classification_confidence
        string image_uri
    }

    ARTWORKS {
        uuid artwork_id PK
        uuid page_id FK
        uuid artist_id FK
        string artist_name_raw
        string title
        string medium
        string dimensions
        numeric estimate_low
        numeric estimate_high
        string currency
        string image_uri
        float extraction_confidence
        string status
        string reviewed_by
        timestamp reviewed_at
    }

    ARTISTS {
        uuid artist_id PK
        string canonical_name
        string aliases
        int birth_year
        int death_year
        string nationality
    }

    REVIEW_CORRECTIONS {
        uuid correction_id PK
        uuid artwork_id FK
        string field_name
        string original_value
        string corrected_value
        string corrected_by
        timestamp corrected_at
    }
```

## Notes on the schema

**`artist_name_raw` vs. `artist_id`.** The metadata extraction skill
produces whatever text it read off the page (`artist_name_raw`); the
entity matching skill then attempts to resolve that to a canonical
`artist_id` in the `ARTISTS` reference table. Both are kept — the raw
value for audit/debugging when matching goes wrong, the resolved ID for
any downstream querying or reporting by artist.

**`status` on both `PAGES` and `ARTWORKS`.** Pages have a coarse status
(entry vs. non-entry, or processing-failed); artworks have their own
status tracking where they are in the human review lifecycle
(`pending_review`, `approved`, `rejected`). This lets the system report
progress at both the page level ("48 of 50 pages processed") and the
record level ("12 of 15 extracted artworks approved").

**`REVIEW_CORRECTIONS` is intentionally granular (one row per field
correction, not one row per record).** This makes it possible to later
answer questions like "which extracted field is most often wrong —
dimensions or estimates?" which a single before/after snapshot per
record would not support, and which is valuable for deciding where to
invest in improving the metadata extraction skill.
