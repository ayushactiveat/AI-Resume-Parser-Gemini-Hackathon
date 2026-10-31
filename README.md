# AI Resume Parser & Job Matcher

An asynchronous multi-service backend that parses resumes (PDF / DOCX / TXT / image) into structured JSON and scores them against a job description using sentence embeddings plus skill overlap.

Built solo as my submission for the **Gemini Hackathon 2025** (Nov–Dec 2025). It uses open-source Hugging Face models so it runs fully locally.

## What it does

You upload a resume. The API saves it, creates a database record and queues a background job, then returns `202 Accepted` immediately. A Celery worker extracts text (Tesseract OCR for image files), runs a BERT NER model, and stores the structured result in Postgres. You then send a job description to the match endpoint and get back an overall score, a recommendation, and a list of required skills the resume is missing.

Matching is not pure keyword search. 60% of the score comes from sentence-embedding similarity, so related wording can still score well; the other 40% is exact overlap with the job's required skills.

## Architecture

Four services orchestrated by Docker Compose:

- **FastAPI (api)**: validates uploads (10 MB limit), stores the file, creates a record with status `processing`, enqueues a task and returns `202`. It also serves results and runs the matching endpoint.
- **Celery worker (worker)**: picks jobs off the queue and does the heavy work: text extraction, OCR and NER. It uses the same image as the API.
- **PostgreSQL (db)**: stores resume metadata, status and the parsed output in a JSON column, so the output shape can change without a schema migration.
- **Redis (redis)**: Celery broker and result backend.

The API never waits for parsing: an upload returns quickly even when model inference takes several seconds, because inference happens in the worker. The match endpoint is synchronous and runs in the API process.

## Stack

- Python 3.11, FastAPI, Celery, Redis, PostgreSQL, SQLAlchemy
- Hugging Face Transformers: `dslim/bert-base-NER` for entity extraction
- `sentence-transformers`: `all-MiniLM-L6-v2` for embedding-based matching
- PyMuPDF and python-docx for text extraction; Tesseract OCR for images
- Docker Compose for the full stack

## Run it

You need Docker Desktop (or Docker Engine) and Docker Compose.

```bash
chmod +x setup.sh
./setup.sh
```

The first build takes 15–20 minutes because of PyTorch. `setup.sh` runs `docker-compose down -v`, so it **resets the database** each time. It creates the `resumes` table with SQLAlchemy `create_all`.

API docs (Swagger UI): http://localhost:8000/docs

## Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/health` | Liveness check |
| POST | `/resumes/upload` | Upload a PDF, DOCX, TXT, JPG or PNG; returns `202` with a resume ID |
| GET | `/resumes/{id}/status` | `processing`, `completed` or `failed` |
| GET | `/resumes/{id}` | Parsed data once the worker finishes |
| POST | `/resumes/{id}/match` | Score against a job description (`409` if parsing is not complete) |

### 1. Upload a resume

`POST /resumes/upload` accepts a file and returns `202 Accepted` with a resume ID.

<img width="1284" height="945" alt="Upload endpoint" src="https://github.com/user-attachments/assets/269ee6b0-b19a-4c25-978a-d1f901578812" />

### 2. Fetch parsed data

`GET /resumes/{id}` returns the stored record. `parsed_data` holds the extracted name, companies, job titles and skills, plus processing metadata.

<img width="1277" height="853" alt="Get parsed resume" src="https://github.com/user-attachments/assets/0f53454a-95dd-40a0-98a3-13a10ab4a6bf" />

### 3. Match against a job description

`POST /resumes/{id}/match` takes a job description JSON and returns an overall score (0–100), a recommendation, category scores and a gap analysis.

<img width="1274" height="745" alt="Match endpoint request" src="https://github.com/user-attachments/assets/14649054-2d3c-42f6-902d-86a7472e415f" />

<img width="1276" height="895" alt="Match endpoint response" src="https://github.com/user-attachments/assets/88f9afc9-96bd-45ef-85fb-6702892f21b4" />

## ML notes

**Entity extraction.** `dslim/bert-base-NER` is a BERT model fine-tuned on CoNLL-2003. It predicts `PER`, `ORG`, `LOC` and `MISC`. I map `PER` to the candidate name, `ORG` to companies, and use `MISC` as a heuristic for job titles and skills. It does not extract dates, emails, phone numbers or education; those are not populated yet. I did not fine-tune a model for the hackathon scope.

**Matching.** The final score is a weighted blend:

- 40%: share of the job's *required* skills found in the resume (exact, case-insensitive match)
- 60%: cosine similarity between `all-MiniLM-L6-v2` embeddings (384-dim) of the resume text (skills and job titles) and the job description, rescaled to 0–100

Recommendation bands: 80+ Strong Match, 60+ Good Match, otherwise Needs Development. The gap analysis lists required skills that were not found by exact match. Weights and thresholds are hand-picked, not tuned.

**Failure handling.** Model loading and inference are wrapped in try/except. If the worker fails on a resume, it marks that record `failed` with the error message, and the API stays up. If matching fails, the endpoint returns a safe fallback response instead of a 500.

## Known limitations

- Scanned (image-only) PDFs are not supported yet; OCR works for JPG/PNG files.
- Long resumes are truncated by the models' token limits (512 for BERT).
- Skill extraction is heuristic and noisy; a resume-specific model would improve it.
- Matching runs in the API process rather than the worker.
- No authentication, no retries on worker failure, no automated tests.
- Uploaded files are shared between the API and worker through a bind mount, which works on a single host only.

## Roadmap

- Store parsed output as `JSONB`; add file-type validation
- Rasterise scanned PDFs for OCR; chunk long text
- Fine-tuned resume NER (dates, email, phone, education)
- Celery retries, section-level semantic gap analysis, object storage for uploads, pytest suite
