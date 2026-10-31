# Architecture Overview: AI Resume Parser

The system is four services orchestrated by Docker Compose. A responsive API gateway is separated from a background worker that does the heavy parsing and OCR.

## Services

| Service | Technology | Role | Scales by |
| --- | --- | --- | --- |
| api | FastAPI + Uvicorn | Validates uploads, stores the file, creates the DB record, enqueues a Celery task, serves status and results, and runs the match endpoint | More API replicas |
| worker | Celery + PyTorch + Transformers | Text extraction (PyMuPDF, python-docx), OCR (Tesseract), NER inference, and saving results | More worker replicas |
| redis | Redis 7 | Celery broker and result backend | Managed Redis |
| db | PostgreSQL 14 | Stores resume metadata, status and parsed output in a JSON column | Managed Postgres |

The API and worker are built from the same image (`docker/Dockerfile.api`) and differ only in their start command. They share uploaded files through the `.:/app` bind mount.

## Request flow

**Upload:**

1. `POST /resumes/upload` sends a file to the API.
2. The API enforces a 10 MB limit, saves the file as `uploads/<uuid><ext>`, and inserts a `resumes` row with status `processing`.
3. The API calls `process_resume.delay(...)`, which places a message on Redis, and returns `202 Accepted` with the ID.
4. The worker extracts text, runs BERT NER, saves the structured JSON, sets status to `completed` (or `failed` with the error), and deletes the uploaded file.
5. The client polls `GET /resumes/{id}/status`, then fetches `GET /resumes/{id}`.

**Match:** `POST /resumes/{id}/match` loads the record, returns `409` if parsing is not complete, then computes the score synchronously in the API process.

## AI strategy

- **Extraction (NER):** `dslim/bert-base-NER` predicts person, organisation, location and misc entities. Post-processing maps these to a name, companies, job titles and skills. Dates, contact details and education are not extracted yet.
- **Matching:** a hybrid score. 40% is exact overlap with the job's required skills; 60% is cosine similarity between `all-MiniLM-L6-v2` embeddings of the resume text and the job description, rescaled to 0–100. The gap analysis lists required skills not found in the resume.

## Design decisions

- **Async via a queue:** inference can take several seconds, so uploads return `202` and the worker processes in the background. This keeps the API responsive and lets workers scale separately.
- **JSON column:** parsed output can change shape without schema migrations.
- **Graceful failure:** model loading is guarded, task failures are recorded on the record, and matching returns a fallback response rather than a 500.

## Known limitations and next steps

- Matching runs in the API; moving it to the worker or caching the model would isolate load.
- Move uploads to object storage so API and worker can run on separate hosts.
- Add Celery retries and late acknowledgement so a worker crash does not leave a job stuck in `processing`.
- Use `JSONB`, add authentication, file-type validation, and tests.
- Support scanned PDFs and chunk long documents.
