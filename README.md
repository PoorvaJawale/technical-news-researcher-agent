# Tech Intelligence Pipeline

An automated tech news intelligence system built with n8n that collects articles from across the internet, uses AI to analyze them, and delivers daily and weekly reports to Discord, Slack, Gmail, and Telegram — with zero manual intervention.

---

## What It Does

Every few hours, the system collects tech news from 5 sources, runs each article through GPT-4o-mini for analysis, stores everything in PostgreSQL, and automatically generates structured reports delivered to all your channels.

```
News Sources (Guardian, Wired, TechCrunch, MIT Review, GitHub)

                    ↓  every 3 hours
        
  [Workflow 1] Collect & normalize raw articles → PostgreSQL
  
                    ↓
        
  [Workflow 2] AI analyzes each article → scored, categorized, stored
  
                    ↓
        
  [Workflow 3] Daily report → Discord / Slack / Gmail / Telegram (8am daily)
  [Workflow 4] Weekly report → Discord / Slack / Gmail / Telegram (Monday 9am)
```

---

## Workflows

### 1. News Collection Agent
**Schedule:** Every 3 hours

Fetches articles in parallel from 5 sources:
- The Guardian (Technology)
- Wired
- MIT Technology Review
- TechCrunch
- GitHub Trending (new repositories with 10+ stars)

All results are merged, normalized into a consistent schema, and inserted into the `raw_articles` table. Duplicate URLs are skipped automatically. Triggers the AI Processing workflow on completion.

### 2. AI Processing
**Trigger:** Called by News Collection Agent (also runs every 6 hours as a fallback)

Picks up all `PENDING` articles and processes each one through GPT-4o-mini to extract:

| Field | Description |
|---|---|
| `summary` | 2-3 sentence article summary |
| `primary_category` | AI, Startup, Open Source, etc. |
| `relevance_score` | 0–10 relevance rating |
| `sentiment` | positive / negative / neutral |
| `business_impact` | Brief business impact statement |
| `key_companies` | Companies mentioned |
| `key_technologies` | Technologies mentioned |
| `article_type` | news / analysis / tutorial / etc. |

Results are saved to `processed_articles`. Each article is marked `PROCESSED` in `raw_articles` so it is never analyzed twice. On completion, triggers the Daily Report Generator.

### 3. Daily Report Generator
**Schedule:** Every day at 8:00 AM

Pulls the top 20 highest-scored articles from the last 24 hours, sends them to GPT-4o-mini, and generates a structured **Daily Technology Intelligence Report** with:

1. Executive Summary
2. Top Stories
3. AI News
4. Startup News
5. Open Source and Developer Tools
6. Key Takeaways

The report is saved to the `reports` table, chunked into 1900-character pieces for Discord's message limit, and sent in full to Slack, Gmail, and Telegram.

### 4. Weekly Report Generator
**Schedule:** Every Monday at 9:00 AM

Pulls the top 50 highest-scored articles from the past 7 days and generates a **Weekly Technology Trend Report** with:

1. Week in Review
2. Top 3 Emerging Themes
3. AI Developments
4. Startup Activity
5. Open Source Highlights
6. What to Watch Next Week

Delivered to the same channels as the daily report.

---

## Database Schema

```sql
-- Raw articles collected from news sources
raw_articles (
  id              SERIAL PRIMARY KEY,
  source          TEXT,
  source_name     TEXT,
  title           TEXT,
  url             TEXT UNIQUE,
  raw_content     TEXT,
  author          TEXT,
  published_at    TIMESTAMP,
  status          TEXT DEFAULT 'PENDING',  -- PENDING | PROCESSED
  created_at      TIMESTAMP DEFAULT NOW()
)

-- AI-analyzed articles
processed_articles (
  id                SERIAL PRIMARY KEY,
  raw_article_id    INTEGER REFERENCES raw_articles(id),
  summary           TEXT,
  primary_category  TEXT,
  categories        TEXT[],
  relevance_score   INTEGER,
  sentiment         TEXT,
  business_impact   TEXT,
  key_companies     TEXT[],
  key_technologies  TEXT[],
  article_type      TEXT,
  processed_at      TIMESTAMP DEFAULT NOW()
)

-- Generated reports
reports (
  id            SERIAL PRIMARY KEY,
  report_type   TEXT,  -- DAILY | WEEKLY
  content       TEXT,
  generated_at  TIMESTAMP DEFAULT NOW()
)
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow automation |
| GPT-4o-mini | Article analysis and report generation |
| PostgreSQL | Data storage |
| Discord Webhook | Chunked report delivery |
| Slack API | Report delivery |
| Gmail OAuth2 | Email delivery |
| Telegram Bot API | Mobile delivery |

---

## Setup

### Prerequisites
- n8n instance (self-hosted or cloud)
- PostgreSQL database
- OpenAI API key
- Discord webhook URL
- Slack bot token
- Gmail OAuth2 credentials
- Telegram bot token

### 1. Database Setup
Run the schema above to create the three tables in your PostgreSQL database.

### 2. Configure Credentials in n8n
Add the following credentials in your n8n instance:
- **Postgres account** — your PostgreSQL connection
- **OpenAI account** — your OpenAI API key
- **Slack account** — bot token with `chat:write` scope
- **Gmail account** — OAuth2 credentials
- **Telegram account** — bot token from @BotFather

### 3. Import Workflows
Import the four JSON files into n8n in this order:
1. `Daily Report Generator.json`
2. `AI Processing.json`
3. `News Collection Agent (1).json`
4. `Weekly Report Generator.json`

### 4. Update IDs and Targets
After import, update the following:
- Discord webhook URL in HTTP Request nodes
- Telegram `chatId` in Telegram nodes
- Gmail `sendTo` address in Gmail nodes
- Slack `channelId` in Slack nodes
- Workflow IDs in `Execute Workflow` nodes (auto-updated on import if workflows are in the same instance)

### 5. Activate
Activate workflows in this order:
1. Daily Report Generator
2. AI Processing
3. News Collection Agent
4. Weekly Report Generator

---

## Delivery Channels

| Channel | Format | Notes |
|---|---|---|
| Discord | Chunked (1900 chars/message) | Respects Discord's 2000 char limit |
| Slack | Full report | Sent to `#daily-tech-report` channel |
| Gmail | HTML formatted | Sent to configured email address |
| Telegram | Full report | Sent to configured chat ID |

---

## Project Structure

```
├── News Collection Agent (1).json   # Workflow 1 - data collection
├── AI Processing.json               # Workflow 2 - AI analysis
├── Daily Report Generator.json      # Workflow 3 - daily reports
├── Weekly Report Generator.json     # Workflow 4 - weekly reports
└── README.md
```
