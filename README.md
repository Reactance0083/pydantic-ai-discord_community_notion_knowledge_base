# Discord Community → Notion Knowledge Base

Automatically transform Discord channel conversations into structured Notion database entries. Uses pydantic-ai to classify messages as questions, answers, decisions, or noise — then clusters threads and writes clean, deduplicated records to Notion with source links back to Discord.

---

## What It Does

- **Polls Discord channels** via bot token on a configurable interval
- **Classifies every message** using pydantic-ai + OpenAI into one of four types: `question`, `answer`, `decision`, `noise`
- **Clusters threaded conversations** into Q&A pairs and decision records
- **Writes structured entries** to a Notion database with title, summary, message type, participants, source link, and timestamp
- **Deduplicates** using message ID tracking so re-runs never create duplicate Notion pages
- **Weekly digest mode** aggregates the last 7 days of decisions and FAQs into a single Notion summary page

---

## Prerequisites

| Requirement | Version |
|---|---|
| Python | 3.11+ |
| Discord Bot Token | Bot must have `MESSAGE_CONTENT` intent enabled |
| Notion Integration Token | Internal integration with write access to target database |
| OpenAI API Key | `gpt-4o` used by default |

You need three external accounts set up before running anything:

1. **Discord** — Create a bot at https://discord.com/developers/applications, enable `Message Content Intent` under Bot settings, invite it to your server with `Read Message History` and `Read Messages` permissions
2. **Notion** — Create an internal integration at https://www.notion.so/my-integrations, share your target database with the integration
3. **OpenAI** — Grab your API key from https://platform.openai.com/api-keys

---

## Setup

**1. Clone the repo and enter the directory**

```bash
git clone https://github.com/yourname/discord-notion-kb.git
cd discord-notion-kb
```

**2. Create and activate a virtual environment**

```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
```

**3. Install dependencies**

```bash
pip install -r requirements.txt
```

**4. Copy the environment file and fill in your secrets**

```bash
cp .env.example .env
```

Open `.env` and add your tokens (see [Configuration](#configuration) below).

**5. Create the Notion database**

Your Notion database must have these properties (exact names, exact types):

| Property Name | Type |
|---|---|
| Title | Title |
| Type | Select |
| Summary | Text |
| Participants | Multi-select |
| Source URL | URL |
| Channel | Text |
| Discord Message ID | Text |
| Created At | Date |

**6. Start the server**

```bash
uvicorn app.main:app --reload --port 8000
```

The API is now running at `http://localhost:8000`. Visit `http://localhost:8000/docs` for the interactive Swagger UI.

**7. Trigger your first sync**

```bash
curl -X POST http://localhost:8000/sync \
  -H "Content-Type: application/json" \
  -d '{"channel_ids": ["YOUR_CHANNEL_ID"], "lookback_hours": 24}'
```

---

## Usage

### One-off sync

Poll specific channels and classify everything from the last N hours:

```bash
curl -X POST http://localhost:8000/sync \
  -H "Content-Type: application/json" \
  -d '{
    "channel_ids": ["1234567890", "9876543210"],
    "lookback_hours": 48,
    "dry_run": false
  }'
```

### Dry run (classify without writing to Notion)

```bash
curl -X POST http://localhost:8000/sync \
  -H "Content-Type: application/json" \
  -d '{
    "channel_ids": ["1234567890"],
    "lookback_hours": 12,
    "dry_run": true
  }'
```

### Weekly digest

```bash
curl -X POST http://localhost:8000/digest \
  -H "Content-Type: application/json" \
  -d '{"channel_ids": ["1234567890", "9876543210"]}'
```

### Start the background poller

```bash
curl -X POST http://localhost:8000/poller/start
```

### Stop the background poller

```bash
curl -X POST http://localhost:8000/poller/stop
```

---

## API Endpoints

### `POST /sync`

Polls Discord channels and writes classified messages to Notion.

**Request body:**

```json
{
  "channel_ids": ["string"],
  "lookback_hours": 24,
  "dry_run": false
}
```

**Response:**

```json
{
  "status": "completed",
  "messages_fetched": 142,
  "messages_classified": 89,
  "noise_filtered": 53,
  "notion_pages_created": 34,
  "notion_pages_skipped_duplicate": 12,
  "elapsed_seconds": 8.4
}
```

**Example:**

```bash
curl -X POST http://localhost:8000/sync \
  -H "Content-Type: application/json" \
  -d '{"channel_ids": ["1234567890"], "lookback_hours": 24, "dry_run": false}'
```

---

### `POST /digest`

Generates a weekly digest page in Notion summarizing decisions and top FAQs from the last 7 days.

**Request body:**

```json
{
  "channel_ids": ["string"],
  "lookback_days": 7
}
```

**Response:**

```json
{
  "status": "completed",
  "notion_page_url": "https://www.notion.so/Weekly-Digest-abc123",
  "decisions_included": 8,
  "faqs_included": 21
}
```

**Example:**

```bash
curl -X POST http://localhost:8000/digest \
  -H "Content-Type: application/json" \
  -d '{"channel_ids": ["1234567890"], "lookback_days": 7}'
```

---

### `GET /status`

Returns current sync status, poller state, and message type breakdown.

**Response:**

```json
{
  "poller_running": false,
  "last_sync_at": "2024-01-15T14:32:00Z",
  "total_pages_created": 247,
  "type_breakdown": {
    "question": 89,
    "answer": 91,
    "decision": 34,
    "noise": 421
  }
}
```

**Example:**

```bash
curl http://localhost:8000/status
```

---

### `POST /poller/start`

Starts the background poller. Polls all channels in `DISCORD_CHANNEL_IDS` every `POLL_INTERVAL_MINUTES` minutes.

**Response:**

```json
{
  "status": "started",
  "interval_minutes": 30,
  "watching_channels": ["1234567890", "9876543210"]
}
```

**Example:**

```bash
curl -X POST http://localhost:8000/poller/start
```

---

### `POST /poller/stop`

Stops the background poller.

**Response:**

```json
{
  "status": "stopped"
}
```

**Example:**

```bash
curl -X POST http://localhost:8000/poller/stop
```

---

### `GET /classifications`

Returns recent message classifications stored in the local cache.

**Query parameters:**

| Param | Type | Default | Description |
|---|---|---|---|
| `limit` | int | 50 | Number of records to return |
| `type` | string | all | Filter by `question`, `answer`, `decision`, `noise` |
| `channel_id` | string | all | Filter by channel |

**Example:**

```bash
curl "http://localhost:8000/classifications?limit=20&type=decision"
```

```bash
# Python / httpx
import httpx
r = httpx.get("http://localhost:8000/classifications", params={"limit": 20, "type": "decision"})
print(r.json())
```

---

### `DELETE /cache`

Clears the local deduplication cache. Next sync will re-evaluate all messages (Notion dedup still prevents duplicate pages).

**Example:**

```bash
curl -X DELETE http://localhost:8000/cache
```

---

## Configuration

Copy `.env.example` to `.env` and set these values:

```env
# ── Discord ──────────────────────────────────────────────────
DISCORD_BOT_TOKEN=your_discord_bot_token_here

# Comma-separated list of channel IDs to watch by default
DISCORD_CHANNEL_IDS=1234567890,9876543210

# How many messages to fetch per channel per API call (max 100)
DISCORD_FETCH_LIMIT=100

# ── Notion ───────────────────────────────────────────────────
NOTION_TOKEN=secret_your_notion_integration_token

# The ID from your Notion database URL
# URL format: https://www.notion.so/{workspace}/{DATABASE_ID}?v=...
NOTION_DATABASE_ID=your_notion_database_id_here

# ── OpenAI ───────────────────────────────────────────────────
OPENAI_API_KEY=sk-your_openai_api_key_here

# Model used for message classification
OPENAI_MODEL=gpt-4o

# ── Poller ───────────────────────────────────────────────────
# How often the background poller runs (minutes)
POLL_INTERVAL_MINUTES=30

# How far back to look on each poll cycle (hours)
POLL_LOOKBACK_HOURS=1

# ── Classification ───────────────────────────────────────────
# Minimum message length to classify (shorter = noise)
MIN_MESSAGE_LENGTH=20

# Confidence threshold below which a message is marked noise (0.0–1.0)
CLASSIFICATION_CONFIDENCE_THRESHOLD=0.7

# ── Cache ────────────────────────────────────────────────────
# SQLite path for dedup tracking
CACHE_DB_PATH=./data/cache.db
```

---

## Customization

**Change the classification prompt**

Edit `app/agent.py`. The pydantic-ai agent uses a structured output model (`MessageClassification`) — you can add new types like `announcement` or `bug_report` by extending the `MessageType` enum and updating the system prompt.

**Add new Notion properties**

1. Add the property to your Notion database
2. Extend the `NotionPage` model in `app/notion_client.py`
3. Map the new field in `build_page_properties()` in the same file

**Change clustering logic**

Thread clustering lives in `app/clustering.py`. By default it groups messages by Discord thread ID, then falls back to a 5-minute time window for channels without threads. Adjust `CLUSTER_TIME_WINDOW_SECONDS` in `.env` or rewrite `cluster_messages()` for custom logic.

**Run on a schedule without the built-in poller**

The built-in poller is an asyncio background task. If you prefer external scheduling (cron, GitHub Actions, Railway cron):

```bash
# Disable built-in poller by not calling /poller/start
# Then hit /sync from your external scheduler:
curl -X POST https://your-deployed-app.com/sync \
  -H "Content-Type: application/json" \
  -d '{"channel_ids": ["1234567890"], "lookback_hours": 1}'
```

**Digest via cron (every Monday 9am)**

```cron
0 9 * * 1 curl -s -X POST https://your-deployed-app.com/digest \
  -H "Content-Type: application/json" \
  -d '{"channel_ids": ["1234567890"]}' >> /var/log/digest.log 2>&1
```

---

## License

MIT