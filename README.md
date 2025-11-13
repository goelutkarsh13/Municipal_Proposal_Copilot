# Municipal_Proposal_Copilot
Developed an AI-powered assistant that drafts infrastructure grant proposals using local government data and policy documents. Built with Python (FastAPI), LangChain, FAISS, and sentence-transformers to ensure factual, policy-aligned, and efficient proposal generation.
[README.md](https://github.com/user-attachments/files/23532433/README.md)
# Municipal Proposal Copilot Backend

Municipal Proposal Copilot is a digital assistant designed to help city and county teams tell their story when applying for infrastructure grants. Instead of starting from a blank page, staff can describe their community and funding goals, and the service suggests polished language for each section of a proposal. It keeps track of local priorities, pulls in helpful reference materials, and assembles everything into a coherent draft that can be reviewed and customized by humans.

## What it does

- Guides staff through sharing community background information in plain language.
- Drafts proposal sections such as executive summaries, problem statements, budgets, and more.
- Suggests evidence-based talking points by referencing a curated library of guidance documents.
- Keeps a running record of what has been generated so teams can revisit and revise their drafts together.

The rest of this README is aimed at developers and technical operators who want to run or extend the service.

## Requirements

- Python 3.11+
- FAISS CPU libraries (installed via `faiss-cpu` PyPI package)
- Environment variables:
  - `DEEPSEEK_API_KEY`: API key for DeepSeek's OpenAI-compatible endpoint
  - `DEEPSEEK_API_BASE` (optional): Override base URL (defaults to `https://api.deepseek.com/v1`)
  - `FAISS_INDEX_PATH` (optional): Filesystem path where the index is stored (defaults to `backend/app/faiss_index`)

## Setup

```bash
cd WillYouFundMe/backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Build the Retrieval Index

The API now builds the FAISS retrieval index automatically on startup using the `.txt` files in `backend/corpus/`. If you need to trigger a rebuild manually (for example after updating the corpus) you can either:

- Call the `POST /index/rebuild` endpoint (documented below), or
- Run the helper script:

  ```bash
  python scripts/build_index.py --corpus corpus --index app/faiss_index
  ```

The rebuild process chunks the corpus with LangChain's `RecursiveCharacterTextSplitter`, embeds passages with `sentence-transformers/all-MiniLM-L6-v2`, and writes a FAISS index to `app/faiss_index/`.

## Running the API

```bash
uvicorn app.main:app --reload --port 8000
```

CORS is enabled for `http://localhost:1420` by default.

## Quickstart Workflow

Once the server is running you can walk through the entire proposal drafting flow with a handful of HTTP requests. The snippets below use `curl`, but any REST client will work.

1. **Check the retrieval index (optional):**

   ```bash
   curl http://localhost:8000/index/status
   ```

2. **Upload or update the community profile:**

   ```bash
   curl -X POST http://localhost:8000/intake_profile \
     -H "Content-Type: application/json" \
     -d @intake.json
   ```

   Save the payload shown in the [endpoint reference](#post-intake_profile) as `intake.json` or craft your own.

3. **Draft an individual section:**

   ```bash
   curl -X POST http://localhost:8000/sections/complete \
     -H "Content-Type: application/json" \
     -d @section.json
   ```

4. **Generate a full proposal in one request:**

   ```bash
   curl -X POST http://localhost:8000/proposal/complete \
     -H "Content-Type: application/json" \
     -d @proposal.json
   ```

5. **Inspect the stored session state:**

   ```bash
   curl http://localhost:8000/session/demo-session
   ```

If you add new `.txt` source documents later, call `POST /index/rebuild` to refresh the retrieval index before generating more content.

## Endpoints

### `GET /index/status`
Returns metadata describing the currently configured FAISS index.

**Response**
```json
{
  "index_path": ".../backend/app/faiss_index",
  "corpus_path": ".../backend/corpus",
  "exists": true,
  "documents_indexed": 4,
  "chunks_indexed": 256
}
```

### `POST /index/rebuild`
Rebuilds the FAISS index. Provide optional overrides if your corpus or index are stored in non-default locations.

**Request**
```json
{
  "corpus_path": "corpus",
  "index_path": "app/faiss_index",
  "chunk_size": 600,
  "chunk_overlap": 120
}
```

### `POST /intake_profile`
Stores or replaces the profile for a session and clears previous outputs.

**Request**
```json
{
  "session_id": "demo-session",
  "profile": {
    "name": "Riverbend",
    "population": 120000,
    "region": "Midwest",
    "priorities": ["stormwater resilience", "green jobs"],
    "constraints": ["aging pump stations"],
    "assets": ["university partnership"],
    "baseline_metrics": {"gallons_treated_daily": 3200000},
    "notes": "Focus on flood mitigation"
  }
}
```

### `POST /sections/complete`
Generates a single section with retrieval-augmented context, structured output, and validation results.

**Request**
```json
{
  "session_id": "demo-session",
  "query": "Focus on resilience upgrades and workforce development",
  "section_spec": {
    "id": "exec_summary",
    "title": "Executive Summary",
    "type": "narrative",
    "word_max": 180
  },
  "grant": {
    "title": "Climate Resilience Infrastructure Grant",
    "sponsor": "State Resilience Office",
    "criteria": [
      "flood mitigation",
      "community workforce",
      "data-driven impact"
    ]
  }
}
```

**Response**
```json
{
  "volume": {
    "id": "exec_summary",
    "title": "Executive Summary",
    "type": "narrative",
    "body": "..."
  },
  "citations": [
    {"source": "sample_guidance.txt", "snippet": "..."}
  ],
  "validation": {
    "passed": true,
    "issues": []
  }
}
```

### `POST /proposal/complete`
Generates multiple sections in sequence, saving the result as the session's `last_proposal`.

**Request**
```json
{
  "session_id": "demo-session",
  "query": "Update to emphasize sensor upgrades and measurable KPIs",
  "volume_list": [
    {"id": "exec_summary", "title": "Executive Summary", "type": "narrative", "word_max": 180},
    {"id": "problem", "title": "Problem Statement", "type": "narrative", "word_max": 250},
    {"id": "objectives", "title": "Objectives", "type": "bullets"},
    {"id": "budget", "title": "Budget", "type": "table"}
  ],
  "grant": {
    "title": "Climate Resilience Infrastructure Grant",
    "sponsor": "State Resilience Office",
    "criteria": [
      "flood mitigation",
      "community workforce",
      "data-driven impact"
    ]
  }
}
```

### `GET /session/{id}`
Returns the persisted session state, including the stored profile and any last generated proposal.

## Sample End-to-End Flow

1. POST an intake profile.
2. Build the FAISS index (`python scripts/build_index.py --corpus corpus --index app/faiss_index`).
3. (Optional) Check `/index/status` to confirm that the retrieval index is ready. If you add new corpus documents later, call `/index/rebuild` to refresh it.
4. Generate individual sections with `/sections/complete` or a full proposal via `/proposal/complete`.
5. Use `/session/{id}` to inspect the stored profile and latest proposal payload for debugging.

The service validates narrative lengths, required grant term usage, bullet counts, and budget contingency requirements. Failed validations trigger one automatic revision attempt before returning issues to the client.

