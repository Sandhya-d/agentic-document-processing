# End-to-End Workflow Diagram

```mermaid
flowchart TD
    A[User uploads PDF] --> B{Valid PDF?}

    B -- No --> C[Show upload error]
    B -- Yes --> D[Store PDF]

    D --> E[Create processing job]
    E --> F[Split PDF into pages]

    F --> G[Process each page]
    G --> H{Artwork page?}

    H -- No --> I[Mark page as non-artwork]
    H -- Yes --> J[Extract page text]

    J --> K{Text available?}
    K -- No --> L[Run OCR]
    K -- Yes --> M[Use extracted text]

    L --> N[Extract artwork image]
    M --> N

    N --> O[Extract artwork metadata]
    O --> P[Calculate confidence]
    P --> Q[Validate fields]

    Q --> R[Create human review task]
    R --> S[Reviewer checks data]

    S --> T{Approved?}
    T -- No --> U[Correct or reject record]
    U --> S

    T -- Yes --> V[Save approved record in PostgreSQL]
    V --> W[Mark page complete]

    I --> X{All pages processed?}
    W --> X

    X -- No --> G
    X -- Yes --> Y[Mark catalogue complete]
```

## What this shows

Every page goes through the same path: check if it's an artwork page,
get the text (from the PDF directly, or OCR if needed), extract the
image, build the metadata, score confidence, validate, and send to a
human reviewer. Nothing is saved to PostgreSQL until a reviewer approves
it. The system keeps going through pages one at a time until the whole
catalogue is done.
