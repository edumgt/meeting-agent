# AI Meeting Summarizer

**English** | [한국어](readme-kr.md)

A practical example that **records a meeting in the browser** and, on the server, **saves it as mp3, transcribes it (STT), and generates a meeting report** in one pass. Besides meeting notes, it also includes **brokerage analyst-report extraction and consensus aggregation**, an **AICC call-center QA assistant**, and a **global markets dashboard** (FX rates, world time zones, and major equity indices). The default language and NLP models are Korean.

- **FE (static web UI, served directly by FastAPI)**
  - Browser recording with `MediaRecorder`
  - Audio and PDF upload
  - Transcript, markdown report, and structured JSON review
  - Analyst-report extraction, ticker consensus lookup, and call QA screens
- **BE (FastAPI)**
  - Audio → mp3 conversion (`ffmpeg`)
  - OpenAI speech transcription (`gpt-4o-mini-transcribe` by default)
  - Existing AI pipeline (summary / extraction / report)
  - Analyst reports and call QA history stored in Postgres (shared instance)

---

If `ffmpeg` is missing, the server returns:

```
Error: HTTP 500: {"detail":"ffmpeg is not installed on server."}
```

```
sudo apt update
sudo apt install -y ffmpeg
```

## 1. Architecture

1) Record in the FE (webm) or upload an audio file  
2) Send it with `POST /transcribe-and-report`  
3) The BE converts it to mp3 and stores it in a server directory  
4) Transcribe the audio with OpenAI STT  
5) Run the existing `/report` pipeline to summarize and extract decisions and action items  
6) Show the result in the FE, including an mp3 download link

---

## 2. Requirements

- Python 3.10+
- `ffmpeg` (server-side audio conversion)
- `OPENAI_API_KEY` environment variable
- A reachable Postgres instance when using analyst or call QA features — meeting-note core features work without it

---

## 3. Quick start

### 3.1 Run the server (venv)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r server/requirements.txt

export OPENAI_API_KEY=""
export TRANSCRIBE_MODEL="gpt-4o-mini-transcribe"
export TRANSCRIBE_LANGUAGE="ko"
export MP3_OUTPUT_DIR="/tmp/ai_meeting_audio"
# Set Postgres connection settings to enable analyst / call QA (those features stay disabled otherwise)
export POSTGRES_HOST="localhost"
export POSTGRES_PASSWORD="password"

uvicorn app:app --host 0.0.0.0 --port 8000 --app-dir server
```

> `analyst.py`, `callcenter.py`, `pipeline.py`, `db.py`, and `tickers.py` import one another as flat modules from inside `server/` (`import pipeline`, `import db`, and so on). Run with `--app-dir server` so that `server/` is the import root, and start `app:app` (not dotted module notation). The Docker image does the same: it copies the contents of `server/` into `WORKDIR /app` and starts `app:app`.

FastAPI serves the web UI directly at `http://localhost:8000` — there is no separate frontend server (`client/public` is copied and mounted at `/static`).

- Web: http://localhost:8000
- API docs: http://localhost:8000/docs

### 3.2 Run with docker-compose

```bash
docker compose up --build
```

`docker-compose.yml` attaches to an external Docker network named `shared-net` and reuses a `postgres` container already running on that network. If the network does not exist, create it with `docker network create shared-net` and attach the Postgres container. If you only need meeting notes, you can remove the `shared-net` settings from `docker-compose.yml`.

---

## 4. Core APIs

### 4.1 `POST /transcribe-and-report`

Upload audio as `multipart/form-data` and get a transcript and report in one request.

Fields:

- `audio` (required): uploaded file
- `meeting_title` (optional)
- `meeting_date_hint` (optional, `YYYY-MM-DD`)
- `include_summary` (optional, `true`/`false`)
- `language` (optional, default `ko`)

Response:

- `transcript`: transcribed text
- `markdown`: formatted meeting notes
- `extracted`: structured result (JSON)
- `mp3_download_url`, `mp3_file_name`: access info for the mp3 stored on the server

Example:

```bash
curl -X POST "http://localhost:8000/transcribe-and-report" \
  -F "audio=@./sample.webm" \
  -F "meeting_title=Weekly engineering sync" \
  -F "meeting_date_hint=2026-02-08" \
  -F "include_summary=true"
```

### 4.2 `GET /audio/{file_name}`

Download an mp3 stored on the server.

### 4.3 Analyst reports

- `POST /analyst-report`: JSON `{ text, analyst_firm, report_date_hint }` → extract rating, target price, and rationale
- `POST /analyst-report/pdf`: same extraction from an uploaded PDF
- `GET /analyst-consensus/{ticker}`: ticker consensus from accumulated reports (opinion distribution, target-price average / min / max)

```bash
curl -X POST "http://localhost:8000/analyst-report" \
  -H "Content-Type: application/json" \
  -d '{"text": "Maintain Buy on Samsung Electronics; raise target price to KRW 90,000. Investment point: memory cycle improvement", "analyst_firm": "Korea Securities"}'

curl "http://localhost:8000/analyst-consensus/005930"
```

### 4.4 Call QA / AICC

- `POST /call-summary`: upload a call recording (multipart) → transcribe → inquiry type, order request, compliance check, and risk grade
- `GET /call-summary/{call_id}`: fetch one stored call QA report

Inbound handling (IVR), CTI, real-time streaming, and automatic agent assignment are not included (post-call upload only). Compliance checks are keyword-matching heuristics for reference and do not satisfy legal requirements by themselves.

```bash
curl -X POST "http://localhost:8000/call-summary" \
  -F "audio=@./call.webm" \
  -F "agent_name=Hong Gildong" \
  -F "ai_provider=openai"
```

### 4.5 Global markets

- `GET /market-overview` (`?refresh=true` bypasses the cache and refetches immediately): major FX rates quoted in KRW, plus major equity indices (8, including KOSPI and KOSDAQ)
- FX rates come from [open.er-api.com](https://www.exchangerate-api.com/docs/free) (official free API). Indices come from Yahoo Finance's unofficial chart endpoint. No API key is required.
- The server caches results for 5 minutes. A failed lookup for one currency or index is reported only on that item's `error` field and does not affect the rest of the response.
- World time zones are computed in the browser with `Intl.DateTimeFormat` and stay accurate regardless of external API status.
- **Caution**: the index source (Yahoo's unofficial endpoint) is an undocumented API. Its response format may change, or access may be blocked, without notice. Use it for reference only — not for trading or investment decisions.

```bash
curl "http://localhost:8000/market-overview"
curl "http://localhost:8000/market-overview?refresh=true"
```

---

## 5. Environment variables

### Existing NLP

- `SUM_MODEL_ID` (default `gogamza/kobart-summarization`)
- `CLS_MODEL_ID` (default `Seonghaa/korean-emotion-classifier-roberta`)
- `EMB_MODEL_ID` (default `upskyy/bge-m3-korean`)
- `CHUNK_MAX_TOKENS`, `REDUCE_MAX_TOKENS`

### Speech pipeline

- `OPENAI_API_KEY` (required)
- `TRANSCRIBE_MODEL` (default `gpt-4o-mini-transcribe`)
- `TRANSCRIBE_LANGUAGE` (default `ko`)
- `MP3_OUTPUT_DIR` (default `/tmp/ai_meeting_audio`)

### Analyst report / call QA storage (Postgres)

- `POSTGRES_HOST` (default `postgres`) — with docker-compose, the shared Postgres container name on `shared-net`
- `POSTGRES_PORT` (default `5432`)
- `POSTGRES_USER` (default `postgres`)
- `POSTGRES_PASSWORD` (default `password`; override via `.env`)
- `POSTGRES_MAINTENANCE_DB` (default `postgres`) — used on first startup to check whether the dedicated database should be created
- `POSTGRES_DB` (default `meeting_agent`) — database for this app, so tables stay separate when several lab projects share one Postgres instance
- If Postgres is unreachable, core meeting-note features (summary, classification, embeddings) still work. Only `/analyst-report*` and `/call-summary*` are affected.

---

## 6. Operations tips

- Mount the mp3 store (`MP3_OUTPUT_DIR`) as a persistent volume.
- Add a retention job (for example, delete files after 7 days).
- Restrict CORS allowed origins to production domains.
- To control STT cost, cap file length and add request authentication (JWT or API key).
- The compliance keyword lists in `server/callcenter.py` (`BANNED_PHRASES` / `REQUIRED_DISCLOSURES`) are reference seeds. Have a compliance team review them before production use.
- Ticker mapping (`server/data/tickers_kr.json`) is a static seed. For a real service, replace it with an official master such as KRX.

---

## 7. Legacy endpoints

The existing text endpoints are still available.

- `POST /summarize`
- `POST /extract`
- `POST /report`
- `POST /classify`
- `POST /embed`
