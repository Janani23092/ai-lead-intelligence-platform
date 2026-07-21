# AI Sales & Lead Intelligence Platform

An end-to-end prototype that automates the first, most time-consuming step of B2B sales:
finding, understanding, and reaching out to the right prospects.

Given a company website URL, the platform:
1. **Scrapes** the site (robots.txt-compliant) for basic business signals.
2. **Summarizes** the business using an LLM (LLaMA 3.3 via Groq API).
3. **Scores** the lead against an optional Ideal Customer Profile (ICP).
4. **Drafts** a personalized outreach email referencing the lead's specific context.
5. Surfaces everything in a **Streamlit dashboard** for a sales rep to review, filter, and act on.

Built as a portfolio project targeting SME-focused sales tooling, in the spirit of platforms like Pepagora.

## Architecture

```
┌─────────────────┐      HTTP       ┌──────────────────┐      ┌─────────────┐
│ Streamlit        │ ─────────────▶ │ FastAPI backend   │ ───▶ │ PostgreSQL /│
│ Dashboard         │ ◀───────────── │ (leads, emails)   │      │ SQLite       │
└─────────────────┘                 └──────────────────┘      └─────────────┘
                                              │
                                              ▼
                                     ┌──────────────────┐
                                     │ Scraper           │
                                     │ (BeautifulSoup,   │
                                     │  robots.txt check)│
                                     └──────────────────┘
                                              │
                                              ▼
                                     ┌──────────────────┐
                                     │ Groq LLM service   │
                                     │ (LLaMA 3.3 70B /   │
                                     │  3.1 8B for cost)  │
                                     └──────────────────┘
```

## Tech Stack

- **Backend:** Python, FastAPI, SQLAlchemy
- **Scraping:** BeautifulSoup (Selenium hook included for JS-heavy sites)
- **LLM:** Groq API (LLaMA 3.3 70B for email drafting, LLaMA 3.1 8B for cheaper summarization/scoring)
- **Database:** PostgreSQL (via Docker Compose) or SQLite (zero-config local dev)
- **Dashboard:** Streamlit
- **Containerization:** Docker & Docker Compose
- **Rate limiting:** slowapi

## Cost-Conscious Design Choices

- Summarization + fit-scoring is a **single combined LLM call** (not two), using the cheaper
  8B model — this is the high-volume step (one call per scraped lead).
- Email drafting uses the larger 70B model **only when a rep explicitly requests it** for a
  specific lead, not automatically for every scrape.
- Scraped page text is truncated to ~4000 characters before being sent to the LLM, capping token usage.
- `/leads/scrape` is rate-limited (10/minute) to prevent runaway costs from accidental loops.

## Ethics & Compliance

- The scraper checks `robots.txt` before fetching any page and refuses disallowed paths.
- It identifies itself with a descriptive `User-Agent` string.
- It only scrapes publicly accessible pages — no login walls, no LinkedIn scraping via
  unofficial APIs (LinkedIn's ToS prohibits this; a production version would use
  LinkedIn's official Partner APIs instead).
- A polite delay is added between requests to avoid overloading target servers.

## Project Structure

```
lead-intel/
├── app/
│   ├── main.py                # FastAPI app entrypoint
│   ├── database.py            # DB engine/session config (Postgres or SQLite)
│   ├── schemas.py             # Pydantic request/response models
│   ├── models/
│   │   └── lead.py            # SQLAlchemy ORM models (Lead, OutreachEmail)
│   ├── routers/
│   │   ├── leads.py           # /leads endpoints
│   │   └── emails.py          # /emails endpoints
│   ├── scrapers/
│   │   └── website_scraper.py # robots.txt-aware scraper
│   └── services/
│       └── llm_service.py     # Groq API calls: summarize, score, draft email
├── dashboard/
│   └── app.py                 # Streamlit UI
├── data/                      # SQLite DB lives here in local dev
├── requirements.txt
├── Dockerfile                 # API container
├── Dockerfile.dashboard       # Dashboard container
├── docker-compose.yml         # API + Dashboard + Postgres
└── .env.example
```

## Running Locally (fastest path — SQLite, no Docker)

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
# edit .env and set GROQ_API_KEY (free key at https://console.groq.com)

# Terminal 1 — start the API
uvicorn app.main:app --reload

# Terminal 2 — start the dashboard
streamlit run dashboard/app.py
```

Then open:
- Dashboard: http://localhost:8501
- API docs (Swagger): http://localhost:8000/docs

## Running with Docker Compose (Postgres included)

```bash
cp .env.example .env
# edit .env and set GROQ_API_KEY

docker compose up --build
```

- API: http://localhost:8000/docs
- Dashboard: http://localhost:8501
- Postgres: localhost:5432 (user: leaduser / pass: leadpass / db: leads)

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| POST | `/leads/scrape` | Scrape a URL, summarize + score it, save as a lead |
| GET | `/leads` | List leads (filter by `min_score`, `status`) |
| GET | `/leads/{id}` | Get a single lead |
| PATCH | `/leads/{id}/status` | Update lead status (new/reviewed/contacted/rejected) |
| DELETE | `/leads/{id}` | Delete a lead |
| POST | `/emails/generate` | Generate a personalized outreach email for a lead |
| GET | `/emails/lead/{id}` | List previously generated emails for a lead |

## Example: Scrape a lead via curl

```bash
curl -X POST http://localhost:8000/leads/scrape \
  -H "Content-Type: application/json" \
  -d '{
        "url": "https://example-small-business.com",
        "icp_description": "Small manufacturing businesses in India with 10-200 employees"
      }'
```

## Possible Next Steps

- Swap the ORM's `create_all` for Alembic migrations.
- Add authentication (OAuth2/JWT) for multi-user dashboard access.
- Add a Kafka/RabbitMQ queue so scraping + LLM calls run async instead of blocking the request.
- Add Selenium-backed scraping for JS-rendered sites (hook already stubbed in `scrapers/`).
- Add caching (Redis) for repeated lookups of the same domain.

## Disclaimer

This is a portfolio/demo project. Before using it against real third-party sites in production,
review the target sites' Terms of Service — some explicitly prohibit scraping even where
robots.txt allows it.
