# Fast-GPT — Document Ingestion & JD Intelligence Pipeline

A FastAPI + Celery service for ingesting documents and extracting structured intelligence from job descriptions using Google Gemini.

## Features

* **Document ingestion** — upload, chunk, and store documents for downstream search and embedding.
* **JD Intelligence Engine** — extract role, skills, experience, domain, and seniority from job descriptions.
* **Resume Matching** — process resumes and rank candidates using retrieval, reranking, and fit signals.

## Tech Stack

* **API:** FastAPI
* **Async tasks:** Celery + Redis (broker & result backend)
* **Database:** PostgreSQL via SQLAlchemy
* **LLM:** Google Gemini (`gemini-2.5-flash`)
* **Containerization:** Docker Compose

## Project Structure

```text
fastapi/
├── app/
│   ├── api/v1/
│   │   └── ingest.py        # API routes: /process-file, /parse-jd, task status
│   ├── db/
│   │   ├── crud.py          # DB read/write helpers
│   │   └── models.py        # SQLAlchemy models (File, Chunk)
│   ├── celery_worker.py     # Celery app + process_file_task
│   ├── chunker.py           # Gemini-based semantic document chunking
│   ├── jd_parser.py         # Gemini-based JD -> structured JSON
│   ├── schemas.py           # Pydantic models
│   ├── settings.py          # Environment-driven configuration
│   ├── utils.py             # Text helpers
│   └── main.py              # FastAPI app entrypoint
├── data/
│   └── uploads/             # Local file storage (gitignored)
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── .env.example
└── .gitignore
```

## System Flow

```text
┌─────────────────────────────────────────────────────┐
│                     SCREEN 1                        │
│                 Upload Job Description              │
│                                                     │
│ [ Drop JD PDF here or paste text ]                 │
│                                                     │
│                  [ Parse JD ]                       │
│                         ↓                           │
│       Shows extracted skills, role, experience      │
│                                                     │
│          "Is this correct?" → [ Confirm ]           │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│                     SCREEN 2                        │
│                   Upload Resumes                    │
│                                                     │
│ [ Drop multiple resume PDFs here ]                  │
│                                                     │
│ resume1.pdf  ✅ indexed                             │
│ resume2.pdf  ✅ indexed                             │
│ resume3.pdf  ⏳ processing...                       │
│                                                     │
│              [ Find Best Matches ]                  │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│                     SCREEN 3                        │
│                  Ranked Results                     │
│                                                     │
│ #1 John Doe      Final Score: 0.87                  │
│    FAISS: 0.91 | Rerank: 4.2 | Fit: 0.75           │
│                                                     │
│ #2 Jane Smith    Final Score: 0.79                  │
│ #3 Bob Khan      Final Score: 0.71                  │
└─────────────────────────────────────────────────────┘
```

## Getting Started

### Prerequisites

* Docker Desktop installed and running
* A free Gemini API key — [Google AI Studio](https://aistudio.google.com/apikey)

### Setup

#### 1. Clone the repository

```bash
git clone <repo-url>
cd fastapi
```

#### 2. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and add your own `GEMINI_API_KEY`.

**Never commit `.env`.** It is already included in `.gitignore`. Each teammate should use their own API key.

#### 3. Build and run

```bash
docker compose up --build
```

This starts four services:

* FastAPI
* Celery
* Redis
* PostgreSQL

#### 4. Verify the API

The API will be available at:

```text
http://localhost:8000
```

> **Note:** If you change `requirements.txt`, rebuild with `docker compose up --build`. A plain `docker compose up` reuses the existing image and will not install new dependencies.

## API Endpoints

### `POST /api/v1/ingest/process-file`

Uploads a file for chunking and storage. Processing runs asynchronously through Celery.

**Form fields:**

* `file` — required
* `param1` — string
* `param2` — integer
* `src` — string, default `"web"`

**Response:**

```json
{
  "file_id": "...",
  "task_id": "...",
  "status": "submitted"
}
```

### `GET /api/v1/ingest/{task_id}`

Fetch the status/result of a submitted task.

### `DELETE /api/v1/ingest/{task_id}`

Remove a task record.

### `POST /api/v1/ingest/parse-jd`

**Module 1 — JD Intelligence Engine**

Accepts either a PDF file or raw job-description text and returns structured JSON.

The endpoint runs synchronously because it requires a single fast LLM call.

**Form fields:**

Provide one of:

* `file` — a `.pdf` upload
* `text` — raw job-description text

#### Example — Raw Text

```bash
curl -X POST http://localhost:8000/api/v1/ingest/parse-jd \
  -F "text=We are hiring an AI Engineer with 3+ years experience in PyTorch and LLMs. AWS and Docker preferred."
```

#### Example — PDF

```bash
curl -X POST http://localhost:8000/api/v1/ingest/parse-jd \
  -F "file=@/path/to/job_description.pdf"
```

#### Response

```json
{
  "role": "AI Engineer",
  "required_skills": [
    "PyTorch",
    "LLMs"
  ],
  "preferred_skills": [
    "AWS",
    "Docker"
  ],
  "experience_years": 3,
  "domain": "Machine Learning",
  "seniority": "Mid-Senior"
}
```

## Hidden Genius Classifier

The XGBoost model is not committed to the repository because it is a binary file.

Each teammate must generate it once after cloning.

### Option A — Inside Docker

Recommended:

```bash
docker compose exec fastapi python scripts/train_genius_classifier.py
```

### Option B — Locally

```bash
pip install xgboost==2.1.3 joblib==1.4.2
python scripts/train_genius_classifier.py
```

The process takes approximately 3 seconds and creates:

```text
app/models/genius_classifier.pkl
```

FastAPI will return a `503` for `/analysis` endpoints until this model file exists.

## Gemini Model Notes

This project uses:

```text
gemini-2.5-flash
```

If you receive a:

```text
429 RESOURCE_EXHAUSTED
```

error with:

```text
limit: 0
```

check that `GEMINI_MODEL` in `.env` is configured to use a Flash model rather than a Pro model.

## Team Contributors

| Contributor      | Role               |
| ---------------- | ------------------ |
| **Abhay Sharma** | Backend Developer  |
| **Yug Saxena**   | Backend Developer  |
| **Yash Saxena**  | Frontend Developer |
| **Sparsh Gahoi** | Frontend Developer |

## Contributing — Team Workflow

1. Pull the latest `main` before starting work.

```bash
git checkout main
git pull origin main
```

2. Create a feature branch:

```bash
git checkout -b feature/your-feature-name
```

3. Make your changes and commit them with clear messages.

```bash
git add .
git commit -m "Describe your changes"
```

4. Push your feature branch:

```bash
git push origin feature/your-feature-name
```

5. Open a Pull Request into `main` for review before merging.

6. Never commit:

```text
.env
__pycache__/
.idea/
data/uploads/
```

## License

Internal hackathon/team project.

Add an appropriate license if this project is later intended for public distribution.



