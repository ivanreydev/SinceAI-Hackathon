# Hackathon 2026: situation search

**SINCE AI HACKATHON 2026**

**UNZIP THE ZIP FILES AND ADD THE SERVER_and_Client to Hackathon**
**UNZIP THE ZIP FILES AND ADD THE SERVER_and_Client to Hackathon**
**UNZIP THE ZIP FILES AND ADD THE SERVER_and_Client to Hackathon**


**Team name:** A-stars

- Ivar Boström
- Egor But-Gusaim
- Emilia Luoma
- Louise Remmer
- Ivan Reynoso

Search past network incidents by describing a new one. Teams chats and emails are split into
**situations** (one problem and how it was solved). Gemini turns each situation into a vector, and the web app
finds the closest past situations for whatever you type.

## How it fits together

```
data/example_raw/  ──run_pipeline.py──▶  data/text/  ──separation.py──▶  situation_texts/<key>.txt
 (Teams JSON, emails)                    (plain text)    (Gemini)          processing/demo2_situations.db
                                                                                   │
                                                              POST /vectorize      ▼
client (React, :5173) ──/nb/*──▶  notebook API (ingest.ipynb, :8000)  ──▶  situations.usearch + categories.json
                                   /similar, /situations/{key}, /vectorize      (Gemini for each query)
```

| Port | What | Started by |
|------|------|------------|
| 8000 | Notebook API (search) | last cell of `ingest.ipynb` |
| 5173 | Client dev server; forwards `/nb` → 8000, `/api` → 8080 | `npm run dev` in `client/` |
| 8080 | `server/app.py`: serves the built client, forwards `/nb` → 8000 | `uvicorn app:app --port 8080` |

## Folders

| Path | Contents |
|------|----------|
| `client/` | React + Vite app. API calls live in `src/api/` (`notebook.js` talks to the notebook API) |
| `server/` | FastAPI app that serves `client/build` (its `/api/*` routes are still stubs) |
| `processing/` | Cleaning and separation scripts, Gemini wrapper (`gemini.py`), situations DB |
| `prompts/` | Gemini prompts used for separation and vectorization |
| `data/` | Raw example data, cleaned staging JSON, plain text, role/alias dictionaries |
| `situation_texts/` | One text file per situation, named by its DB key |
| `ingest.ipynb` | Vector index, categories, ingest driver and the search API |
| `towers.csv.gz` | Cell tower table used to place situations on a map grid |

## Setup

**Python 3.11+**

```bash
pip install -r server/requirements.txt
pip install usearch numpy requests       # used by the notebook
```

**Node 20+**

```bash
cd client
npm install
```

**Gemini key.** Put the same key in both places:

- `gemkey.txt` (project root): read by the notebook
- `processing/.env` as `GEMINI_API_KEY=...`: read by `processing/gemini.py`

Both files are secrets. Don't commit or share them.

## Run it

1. **Clean the raw data** (only when `data/example_raw/` changes):
   ```bash
   cd processing
   python run_pipeline.py
   ```
2. **Start the search API.** Open `ingest.ipynb` with the project root as the working directory and run all cells.
   The last cells print `API running at http://127.0.0.1:8000`. Interactive docs: http://127.0.0.1:8000/docs
3. **Add situations** (optional, calls Gemini):
   ```bash
   cd processing
   python separation.py
   ```
   It writes new situations to the DB and `situation_texts/`, then calls `POST /vectorize` on the notebook API.
4. **Start the client:**
   ```bash
   cd client
   npm run dev
   ```
   Open http://localhost:5173 and describe a problem.

### Built app (one URL)

```bash
bash client/build.sh                  # builds client/build
cd server
uvicorn app:app --port 8080
```

Open http://127.0.0.1:8080. The notebook API still has to be running on 8000.

### Without a backend

Set `VITE_USE_MOCKS=true` (delete `client/.env.development.local`, which turns mocks off) to use the fake data in
`client/src/api/mocks/`.

## Notebook API

| Method | Path | Returns |
|--------|------|---------|
| POST | `/similar` | `{"text": "...", "k": 10}` → `{"results": [{key, distance, relevance, timestamp, recency, score}]}`, best first |
| GET | `/situations/{key}` | The situation's raw text |
| POST | `/vectorize` | Vectorizes every situation that has no vector yet |

Re-running the API cell restarts the server. To change the port, set `API_PORT` in that cell and the `/nb` target
in `client/vite.config.js`.

## Troubleshooting

| You see | Cause | Fix |
|---------|-------|-----|
| `502 Bad Gateway` with no details | Nothing is listening on 8000 | Run the notebook's cells (step 2) |
| `502 ... 401 UNAUTHENTICATED ... API key not valid` | Gemini rejected the key the notebook loaded | Restart the kernel and run all cells. If it persists, test the key with `cd processing && python gemini.py`, and replace it in both key files if that fails too |
| `Frontend not built` (503) on :8080 | `client/build` is missing | `bash client/build.sh` |
| `Index on disk has dim=...` | `situations.usearch` was built with other settings | Keep `DIM` in the notebook at 128, or delete the index and re-vectorize |
