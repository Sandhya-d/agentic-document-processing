# Agentic AI Document Processing

This project presents a system design for processing scanned artwork catalogues into structured records using an agentic AI workflow.

This is a design project only. No implementation or code is included.

## Project Structure

- `docs/design-document.md` – System design and architecture
- `diagrams/architecture.md` – Architecture diagram
- `diagrams/workflow.md` – Workflow diagram
- `diagrams/data-model.md` – Database design

## Overview

The system uses AI agents to coordinate the workflow and AI skills to process documents. All extracted records are reviewed by a human before being stored in PostgreSQL.

## Main Components

**Agents**
- Coordinator
- Classification
- Extraction
- Validation
- Persistence

**Skills**
- OCR
- Layout Detection
- Image Extraction
- Metadata Extraction
- Entity Matching
- Confidence Scoring

For more details, see `docs/design-document.md`.
