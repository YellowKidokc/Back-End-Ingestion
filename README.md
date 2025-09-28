Perfect — I’ve got the full ingest/TTS/transcribe picture. Here’s a crisp spec you can hand to Codex that covers RSS (incl. OPML), bulk video/text ingest, scoring, transcription, TTS batching, and Cloudflare-first plumbing. I’m flagging pitfalls where Codex might stumble and giving exact API shapes + queues so implementation is unambiguous.

What you want (in one sentence)

A single “Ingest Hub” that can accept RSS/OPML, pasted feed lists, YouTube/TikTok/video files, and raw text, then transcribe → score (HF) → keep if ≥5 → store (R2 now, Postgres later), with batch TTS for long papers, and front-end pages that show daily cards + “expand to 3/6 months” views.

Cloudflare-first architecture (recommended)

Cloudflare R2
Buckets:

raw-media/ (original uploads, audio/video)

transcripts/ (JSON + VTT/SRT)

tts/ (MP3/OGG outputs)

articles/ (normalized JSON from RSS)

Cloudflare D1 (or SQLite now → later migrate to Postgres via Hyperdrive)

Tables for items, feeds, jobs, stats_rollups

Cloudflare Queues

ingestQueue (new URLs/items)

transcribeQueue (audio/video → text)

scoreQueue (text → HF model)

ttsQueue (text chunks → audio)

Cloudflare Workers + Cron Triggers

Cron: poll RSS, process OPML folders, rotate stats

Cloudflare Workers AI (if available for you)

Speech-to-Text (Whisper) for transcription

Text-to-Speech for TTS (or fallback to an external TTS API you choose)

Cloudflare Pages

Hosts the two front-ends (Main dashboard + Antichrist dashboard)

Fetches data from Worker endpoints

“Expanded page” uses a querystring (?topic=…&days=90) to render 3–6 month history.

If Workers AI isn’t enabled for your account, Codex should add adapters to call Hugging Face Inference, OpenAI Whisper API, or a local Whisper service. Keep the interfaces identical so the switch is config-only.

Backend services (clean contracts)
1) Feeds + OPML

POST /feeds/opml

Body: raw OPML (text/xml)

Action: parse, insert/update feeds into feeds(topic, url, source, active)

Return: { imported: n }

POST /feeds/bulk

Body: { topic: "Middle East", urls: "https://a.com/rss\nhttps://b.com/feed\n..." }

Action: parse lines, validate, insert into feeds

Return: { imported: n, skipped: m }

POST /rss/ingest

Body: [ { "topic":"Antichrist", "url":"...", "source":"...", "minScore":5 } ] (optional; else uses active feeds)

Action: enqueue each feed to ingestQueue

Worker cron (15 min)

Dequeues ingestQueue:

GET feed, dedupe by hash(guid|link), extract title|link|published|summary

Push each item to scoreQueue with a normalized payload (see below)

Queue message (RSS item → score)

{
  "type": "rss_item",
  "topic": "Middle East",
  "title": "IDF, Temple Mount …",
  "link": "https://...",
  "source": "jpost",
  "published_at": "2025-02-01T14:23:00Z",
  "fallback_text": "summary if no fulltext",
  "min_score": 5.0
}


Codex MUST:

Use Readability-like extraction if allowed, else fall back to summary.

Set a short timeout and continue; never block the whole batch.

Deduplicate items in D1 by id = sha256(link|published).

2) Video ingestion (YouTube/TikTok/local file)

POST /videos/submit

Body:

{
  "topic": "Antichrist",
  "inputs": [
    {"url":"https://youtube.com/watch?v=..."},
    {"url":"https://www.tiktok.com/@.../video/..."},
    {"upload_key":"raw-media/uploads/xyz.mp4"}  // if user uploaded to R2 first
  ],
  "transcribe": true,
  "tts": false
}


Action: enqueue each input to transcribeQueue with a media locator

Queue message (media → transcribe)

{
  "type": "media_transcribe",
  "topic": "Antichrist",
  "media": {
    "kind": "youtube" | "tiktok" | "r2",
    "url": "https://...",
    "r2_key": "raw-media/uploads/xyz.mp4"
  },
  "metadata": {"title":"", "source":"youtube"}
}


Codex MUST:

Use yt-dlp in a batch worker environment OR Cloudflare Stream direct upload:

If you want to stay 100% CF-native: Stream handles upload/transcode; store Stream video ID; pull audio for STT via Stream’s API (or fetch the HLS/audio track).

Send audio chunks to Workers AI (Whisper) or your STT provider.

Persist transcripts to transcripts/{id}.json (+ .vtt) in R2.

Produce a normalized item:

topic/title/source/link/published_at/summary/transcript_text

Push to scoreQueue.

3) Scoring (Hugging Face)

Queue message (text → score)

{
  "type": "score_text",
  "topic": "Antichrist",
  "title": "…",
  "link": "https://...",
  "source": "youtube|rss|manual",
  "published_at": "2025-02-01T14:23:00Z",
  "text": "fulltext or transcript (trim 8k chars)",
  "min_score": 5.0,
  "extras": {"transcript_r2": "transcripts/abc.json"}
}


Scoring rules:

Call Hugging Face Inference API.

Map to 0–10 (as we did earlier); keep if score >= min_score.

Write canonical JSON to articles/{yyyy}/{mm}/{id}.json.

Upsert into items table:

id TEXT PK, topic TEXT, title TEXT, link TEXT, source TEXT,
published_at TEXT, score REAL, kept INTEGER, yyyymm TEXT


Update daily rollups (for stats).

Optional fusion with your Character Analyzer

Add a setting to call your Node microservice for character_final_score

Store both hf_score and character_score, plus a blended_score

Threshold stays on blended_score

4) TTS (batch-friendly)

POST /tts/submit

{
  "voice": "narrator_1",
  "inputs": [
    {"text_key":"articles/2025/02/abc.json"},   // fetch & TTS the normalized text
    {"raw_text":"(…long text…)", "title":"Paper 1"},
    {"r2_text_key":"papers/long/uid.txt"}      // direct text files in R2
  ],
  "split": { "max_chars": 2400, "overlap": 120 },  // chunking params
  "output_format": "mp3"
}


Action:

For each input, fetch text, chunk it (hard requirement for long papers)

Enqueue each chunk to ttsQueue

On completion, manifest:

tts/{job_id}/manifest.json listing ordered parts + combined file if you choose to merge

Return: { job_id, status_url: "/jobs/{job_id}" }

Queue message (text chunk → TTS)

{
  "type": "tts_chunk",
  "job_id": "…",
  "voice": "narrator_1",
  "chunk_index": 0,
  "text": "…",
  "out_key": "tts/{job_id}/part_000.mp3"
}


Codex MUST:

Use Workers AI TTS if enabled; otherwise adapter for your TTS provider.

Do atomic writes to R2 (write temp key, then rename).

Provide download + stream links in the manifest.

5) Front-end data (what the cards call)

GET /stats/summary?topic=Temple%20Mount
→ { count_today, count_7d, count_30d, vs_yesterday, avg_7d, avg_30d }

GET /items?topic=Temple%20Mount&days=1&min_score=5&page=1&size=20
→ recent links for card body (RSS anchors)

GET /items/expand?topic=Temple%20Mount&days=90
→ large payload for the “full page” (split into 2 columns client-side)

GET /jobs/{id}
→ job status for big TTS batches or bulk video submits

Pitfalls Codex must avoid (and how)

Blocking ops

Never do STT/TTS/scoring inline on HTTP routes. Always enqueue → worker.

Short timeouts; network errors shouldn’t break batches.

Item duplication

Dedup on id = sha256(link|published) before scoring/transcribing.

Half-written files

Atomic writes to R2: upload to *.tmp then PUT copy to final key; delete temp.

Large texts / long papers

Always chunk with overlap, rejoin conceptually via manifest.

Let user download merged MP3 after all parts finish (server-side concat optional).

YouTube/TikTok

Prefer Cloudflare Stream (if you can) to offload transcode/hosting.

If using yt-dlp, run in a controlled worker/VM (outside Workers runtime), then upload to R2. Respect ToS.

Model output variability

Normalize every model output to 0–10 before thresholding.

Log raw provider responses for audit.

Stats drift

Maintain daily rollups (counts per topic) once per ingest batch.

/stats/summary should be precomputed, not aggregating large tables on the fly.

Minimal DB (D1/SQLite) schema
CREATE TABLE IF NOT EXISTS feeds (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  topic TEXT NOT NULL,
  url TEXT NOT NULL,
  source TEXT,
  active INTEGER DEFAULT 1,
  UNIQUE(url)
);

CREATE TABLE IF NOT EXISTS items (
  id TEXT PRIMARY KEY,
  topic TEXT,
  title TEXT,
  link TEXT,
  source TEXT,
  published_at TEXT,
  score REAL,
  kept INTEGER,
  yyyymm TEXT
);

CREATE TABLE IF NOT EXISTS jobs (
  id TEXT PRIMARY KEY,
  kind TEXT,               -- ingest|transcribe|score|tts
  status TEXT,             -- queued|running|done|error
  created_at TEXT,
  updated_at TEXT,
  meta TEXT                -- JSON
);

CREATE TABLE IF NOT EXISTS stats_daily (
  topic TEXT,
  day TEXT,                -- YYYY-MM-DD
  kept_count INTEGER,
  PRIMARY KEY(topic, day)
);

Exact instructions to Codex (copy/paste)

Stand up the backend as FastAPI (or Node/Express if preferred), exposing:

POST /feeds/opml, POST /feeds/bulk, POST /rss/ingest

POST /videos/submit, POST /tts/submit

GET /items, GET /items/expand, GET /stats/summary, GET /jobs/:id

Storage adapters for Cloudflare R2, and DB adapter for D1/SQLite (env-switchable).

Implement Cloudflare Workers for:

Queues: ingestQueue, transcribeQueue, scoreQueue, ttsQueue

Cron: poll all active feeds every 15 minutes → push items

Workers AI adapters: STT/TTS (with fallbacks)

OPML: parse outline elements, support nested groups; insert into feeds.

Bulk feed paste: split on newline, trim, validate as URL; insert with a topic tag.

Transcription: accept YouTube/TikTok URLs or existing r2_key.

If Stream is enabled: upload → poll until ready → extract audio → send to STT.

Else: use yt-dlp in batch environment, upload to R2, then STT.

Scoring: call Hugging Face Inference; map to 0–10; keep if ≥5; write canonical articles/{yyyy}/{mm}/{id}.json; upsert items; update stats_daily.

TTS batch: chunk long texts; enqueue each chunk; write outputs to tts/{job_id}/part_###.mp3; create manifest.json and (optionally) a merged MP3.

Atomic writes everywhere; retries with backoff on network; never block HTTP routes with long work.

Front-end contract: ensure endpoints return the fields exactly as listed; dates in ISO8601 UTC; pagination stable.

UI notes (so Codex aligns with your pages)

Cards are full-height (no inner scroll); page scrolls vertically.

Each card:

Clickable title → your long-form article page

Two stat lines: vs_yesterday, avg_7d, avg_30d, count_today, count_7d, count_30d

RSS list: 4–7 latest links (or however many available that day)

Small badges: 3m / 6m / Full (linking to items/expand)

Tags beneath stats (topic-specific keywords)

“Expanded view” is a separate page that calls /items/expand?topic=…&days=90/180/365, shows 2 columns of links + richer stats & charts.

Final sanity checks (before you run)

Put HF token + Workers AI credentials in env/secret bindings.

Test with a small OPML and 2–3 videos.

Confirm R2 keys appear under expected prefixes and that items rows populate.

Confirm /stats/summary returns fast (precomputed rollups).

Try a TTS job with a multi-page text (verify chunking & manifest).

If Codex follows this, you’ll have your always-on ingest hub: RSS & video flow in, everything is transcribed, scored, stored, and the UI gets instant stats with expandable history.

ChatGPT can make mistakes. Check important info.
