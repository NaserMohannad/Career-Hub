Career Hub – AI Job & Internship Recommender (CrewAI + HF Spaces + n8n)

This repository documents a production-ready multi-agent pipeline I built with CrewAI to source, scrape, and rank student and entry-level opportunities, then deliver them to learners via n8n and email. I designed and coded all tools, prompts, and server endpoints, and deployed the stack with a Dockerfile on Hugging Face Spaces.

Highlights

Single owner: I built the agent system end-to-end (CrewAI agents, tools, prompts, API, deployment).

Multi-agent pipeline: query generation → search → URL filtering → page scraping → ranking → export.

Student-first: prompts and filters are engineered to focus strictly on internships, junior, trainee, and entry-level roles.

HTTP ingestion: the server accepts student profiles from a teammate (n8n) via POST JSON.

Handoff to n8n: ranked results are posted to an n8n webhook for downstream email delivery.

System Overview
Client (Form / n8n)  →  /webhook (FastAPI on HF Space)
                             │
                             ├─ saves payload → ./incoming/payload_*.json
                             └─ schedules run_pipeline()
                                     │
                                     ├─ Agent 1: Generate Queries (CrewAI + LLM)
                                     ├─ Agent 2: Search (Tavily API)
                                     ├─ Agent 3: Scrape (ScrapeGraph)
                                     └─ Rank: filter + score early-career fit
                                     │
                                writes artifacts → ./daily-job-recommendations
                                     │
                                     └─ POST → RESULTS_WEBHOOK_URL (n8n) for emailing

Features

Precise targeting: interests × locations (+ optional skills) → early-career seniorities only.

Strict domain filtering: LinkedIn, Bayt, Akhtaboot, Indeed, Glassdoor.

On-page extraction: no hallucinations; scraper returns exact fields only.

Ranking: blends skills match, interest match, seniority, location, freshness, and language.

Stateless API: POST /webhook to enqueue a run; GET /health for status.

API
POST /webhook

Accepts a student profile, saves it to ./incoming/, and schedules the pipeline.

Headers

Content-Type: application/json


Body (example)

{
  "id": "s-001",
  "name": "Student Name",
  "email": "student@example.com",
  "interests": "data engineering, analytics",
  "job_skills": "python, sql",
  "locations": ["Amman", "Remote"]
}


Response

{
  "ok": true,
  "message": "Payload received. Pipeline scheduled.",
  "saved": "./incoming/payload_20250101-120000.json",
  "output_dir": "./daily-job-recommendations"
}

POST /run?sync=true|false

Trigger the pipeline manually (blocking or in background).

GET /health

Basic readiness info + any pipeline import errors.

Agents & Tools

I authored all tools in Python and wired them to CrewAI agents with strict, prompt-engineered instructions.

Agent 1 — Job Search Query Generator

Goal: Build queries from interests × seniority × locations (+ light skills).

Output: up to 12 unique queries.

Why: keeps the search space aligned to student/entry-level roles.

Agent 2 — Tavily Job Search Specialist

Goal: Use Tavily to fetch real job posting URLs.

Filters: allowed domains + path heuristics (/job/, /jobs/view/, etc.).

Output: deduplicated, scored URLs.

Agent 3 — Job Data Scraper

Goal: Extract on-page details only (title, company, location, description, requirements, salary, benefits, posted date, language, seniority, apply URL).

Engine: scrapegraph_py SmartScraper with a strict JSON schema prompt.

Ranking

Weights: skills match, interest match, early-career seniority, location fit (GCC/Jordan or Remote), freshness, language, small source boost.

Produces a top_jobs list with reasons and metadata.

Data Model (selected)
{
  "jobs": [
    {
      "page_url": "...",
      "title": "...",
      "company": "...",
      "location": "...",
      "is_remote": true,
      "seniority": "internship|junior|entry-level|...",
      "posted_at": "YYYY-MM-DD",
      "job_type": "Full-time|Internship|...",
      "salary": "...",
      "description": "...",
      "requirements": ["...", "..."],
      "benefits": ["...", "..."],
      "apply_url": "...",
      "source_domain": "linkedin.com",
      "language": "English|Arabic",
      "scrape_status": "success|failed|error"
    }
  ]
}

Environment Variables

Set these in HF Spaces (Repository/Space secrets) or locally:

OPENROUTER_API_KEY – OpenRouter API key

OPENROUTER_BASE_URL=https://openrouter.ai/api/v1

TAVILY_API_KEY – Tavily search key

SCRAPEGRAPH_API_KEY (or scrap_key) – ScrapeGraph key

AGENTOPS_API_KEY (optional) – AgentOps session logging

RESULTS_WEBHOOK_URL – n8n endpoint to receive ranked results

Tip: Some LLM stacks also check OPENAI_API_KEY/OPENAI_API_BASE. You can mirror:

OPENAI_API_KEY=$OPENROUTER_API_KEY
OPENAI_API_BASE=https://openrouter.ai/api/v1

Deployment
Docker

The project ships with a Dockerfile to build a reproducible image.

Expose the FastAPI app (app.py) and mount or bake your environment variables.

Hugging Face Spaces

Space type: Docker or FastAPI.

Add secrets in Settings → Repository secrets.

The app persists artifacts under:

./incoming/ for raw payloads

./daily-job-recommendations/ for outputs

n8n Integration

Teammate sends student profiles to POST https://<space>/webhook with JSON body.

After the run finishes, the server POSTs the ranked results to RESULTS_WEBHOOK_URL (n8n), which handles email delivery to the student.

n8n HTTP Request node (sending to our server)

Method: POST

URL: https://<space-domain>/webhook

Send: JSON body (see example above)

Content-Type: application/json

n8n Webhook (receiving from our server)

Method: POST

Expects the ranked JSON; proceed with templated email.

Operations & Monitoring

Logs: Hugging Face Space logs show /webhook requests, payload saves, and pipeline progress.

Error transparency: endpoints return structured JSON with validation errors.

AgentOps (optional): session traces for agent decisions, tools, and timings.

Security Notes

Validate Content-Type: application/json and reject malformed bodies.

Keep secrets in Space/Repo secrets, not in the repo.

Limit allowed outbound domains in the scraper to reduce risk.

Consider an HMAC or token header if you need to restrict /webhook.

Local Testing
curl -i -X POST https://<your-space>.hf.space/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "id":"test-01",
    "name":"Test Student",
    "email":"student@example.com",
    "interests":"data engineering, analytics",
    "job_skills":"python, sql",
    "locations":["Amman","Remote"]
  }'

Why this approach

Deterministic curation: tight prompts and filters prevent noise.

Composable: agents are decoupled via CrewAI tasks and well-scoped tools.

Portable: Docker + FastAPI + Spaces make it easy to deploy and integrate.

Roadmap

Per-student dashboards and saved searches.

Application tracker with reminders.

Multi-language scoring and email localization.

Additional sources with structured APIs.

Ownership

Owner: I designed and implemented the CrewAI agents, tools, prompts, API server, Docker/HF Spaces deployment, and the n8n integration contract.

Teammate: Handles final email delivery via n8n using the webhook payload I send.
