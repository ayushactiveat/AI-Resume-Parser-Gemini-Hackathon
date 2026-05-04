# AI Resume Parser & Job Matcher

A multi-service system that parses resumes (PDF / DOCX / image) into structured data and scores them against a job description using sentence embeddings.

Built solo as my submission for the **Gemini Hackathon 2025** (Nov–Dec 2025).

## What it does

You upload a resume → the system extracts text (with OCR fallback for images), runs NER to pull out entities like names, roles, and dates, and stores the structured result in Postgres. You then send a job description and get back a similarity score plus a gap analysis showing what the candidate is missing.

The point isn't keyword matching. It uses sentence embeddings, so "DevOps practices" and "Docker/Kubernetes" map close together even though the words don't overlap.

## Architecture

Four services orchestrated by Docker Compose:

- **FastAPI** — the request-handling layer. Validates uploads, writes a job to the queue, and returns `202 Accepted` immediately. The user never waits for model inference.
- **Celery worker** — picks jobs off the queue and runs the heavy work: text extraction, OCR, NER, embedding generation.
- **PostgreSQL** — stores resume metadata and parsed output. The parsed output goes into a `JSONB` column so the schema can change as the model output evolves without a migration.
- **Redis** — Celery broker and result backend.

This split is the whole point of the design: an upload returns in milliseconds even if model inference takes 10+ seconds, because the API never blocks on the worker.

## Stack

- Python 3, FastAPI, Celery, Redis, PostgreSQL
- Hugging Face Transformers — `dslim/bert-base-NER` for entity extraction
- `sentence-transformers` — `all-MiniLM-L6-v2` for embedding-based matching
- Tesseract OCR for image-based resumes
- Docker Compose for the full stack

## Run it

You need Docker Desktop (or Docker Engine) and Docker Compose.

```bash
chmod +x setup.sh
./setup.sh
```

API docs (Swagger UI): http://localhost:8000/docs

## Endpoints

### 1. Upload a resume

`POST /resumes/upload` — accepts PDF, DOCX, or image. Returns `202 Accepted` with a resume ID.

<img width="1284" height="945" alt="Upload endpoint" src="https://github.com/user-attachments/assets/269ee6b0-b19a-4c25-978a-d1f901578812" />

### 2. Fetch parsed data

`GET /resumes/{id}` — returns the structured profile (skills, experience, contact info) once the worker finishes processing.

<img width="1277" height="853" alt="Get parsed resume" src="https://github.com/user-attachments/assets/0f53454a-95dd-40a0-98a3-13a10ab4a6bf" />

### 3. Match against a job description

`POST /resumes/{id}/match` — send a job description JSON, get back an overall score and a gap analysis.

<img width="1274" height="745" alt="Match endpoint request" src="https://github.com/user-attachments/assets/14649054-2d3c-42f6-902d-86a7472e415f" />

<img width="1276" height="895" alt="Match endpoint response" src="https://github.com/user-attachments/assets/88f9afc9-96bd-45ef-85fb-6702892f21b4" />

## ML notes

**Entity extraction.** `dslim/bert-base-NER` is a pre-trained BERT fine-tuned on CoNLL-2003. It handles `PER`, `ORG`, `LOC`, `MISC` reasonably well out of the box, which covers most of what you actually need from a resume — names, companies, locations, dates. Good enough that I didn't need to fine-tune for the hackathon scope.

**Matching.** `all-MiniLM-L6-v2` produces 384-dim sentence embeddings. The matcher embeds both the resume and the JD, takes cosine similarity over relevant sections, and returns a normalized score plus a list of JD requirements that don't have a strong match in the resume (the "gap analysis").

**Failure handling.** Model calls are wrapped so a single inference failure doesn't take down the API — the worker logs and moves on, the API stays up, the user gets an error on that specific resume rather than a 500 across the system.
