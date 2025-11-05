# Career Hub – AI Job & Internship Recommender (CrewAI + HF Spaces + n8n)

This repository documents a production-ready multi-agent pipeline built with CrewAI to source, scrape, and rank student and entry-level opportunities, then deliver them to learners via n8n and email.  
I personally designed and implemented all tools, prompts, and API endpoints, and deployed the full stack using Docker on Hugging Face Spaces.

---

## Highlights

- **Single Owner** – Full system design and implementation (CrewAI agents, tools, prompts, API, Docker, deployment).  
- **Multi-Agent Pipeline** – Query generation → Search → URL filtering → Page scraping → Ranking → Export.  
- **Student-First Focus** – Optimized strictly for internships, junior, trainee, and entry-level roles.  
- **HTTP Ingestion** – Receives student data via POST requests from an n8n workflow.  
- **Handoff to n8n** – Sends ranked results through a webhook for automated email delivery.  

---

## System Overview

Client (Form / n8n) → /webhook (FastAPI on HF Space)  
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

---

## Features

- Precise targeting: combines interests, skills, and preferred locations to find early-career roles only.  
- Strict domain filtering: LinkedIn, Bayt, Akhtaboot, Indeed, Glassdoor.  
- Accurate scraping: extracts exact on-page data using ScrapeGraph.  
- Intelligent ranking: scores jobs by skill match, interest, seniority, location, freshness, and language.  
- Stateless API: POST `/webhook` to trigger a run, GET `/health` for service status.  

---

## API Endpoints

### POST /webhook

Receives a student profile, saves it to `./incoming/`, and schedules the pipeline.

**Headers**  
Content-Type: application/json

**Body Example**
```json
{
  "id": "s-001",
  "name": "Student Name",
  "email": "student@example.com",
  "interests": "data engineering, analytics",
  "job_skills": "python, sql",
  "locations": ["Amman", "Remote"]
}
```

**Response**
```json
{
  "ok": true,
  "message": "Payload received. Pipeline scheduled.",
  "saved": "./incoming/payload_20250101-120000.json",
  "output_dir": "./daily-job-recommendations"
}
```

### POST /run?sync=true|false

Runs the pipeline manually (synchronously or in the background).

### GET /health

Returns readiness information and pipeline import status.

---

## Agents and Tools

All tools were coded manually in Python and integrated into CrewAI agents using prompt-engineered instructions.

### Agent 1 — Job Search Query Generator

Goal: Build job search queries from interests, seniority, and locations.  
Output: Up to 12 diverse queries.  
Purpose: Keep the search focused on student/entry-level opportunities.

### Agent 2 — Tavily Job Search Specialist

Goal: Retrieve real job posting URLs from Tavily.  
Filters: Allowed domains and path heuristics (e.g., /job/, /jobs/view/).  
Output: Deduplicated and scored job URLs.

### Agent 3 — Job Data Scraper

Goal: Extract verified job details (title, company, location, salary, etc.).  
Engine: scrapegraph_py SmartScraper using a structured JSON schema.

---

## Ranking

Jobs are ranked using weighted factors:

- Skill match  
- Interest match  
- Seniority (intern/junior/entry)  
- Location fit (Jordan, GCC, or Remote)  
- Posting freshness  
- Language  
- Domain priority (LinkedIn, Bayt, etc.)  

Results include a `top_jobs` list with reasons and metadata.

---

## Data Model Example
```json
{
  "jobs": [
    {
      "page_url": "...",
      "title": "...",
      "company": "...",
      "location": "...",
      "is_remote": true,
      "seniority": "internship|junior|entry-level",
      "posted_at": "YYYY-MM-DD",
      "salary": "...",
      "description": "...",
      "requirements": ["...", "..."],
      "benefits": ["...", "..."],
      "apply_url": "...",
      "source_domain": "linkedin.com",
      "language": "English|Arabic",
      "scrape_status": "success"
    }
  ]
}
```

---

## Deployment

### Docker

The repository includes a Dockerfile for containerized deployment.  
Expose the FastAPI app and set environment variables via Docker `--env` or `.env`.

### Hugging Face Spaces

Space type: Docker or FastAPI  
Add secrets under **Settings → Repository secrets**  

Output paths:  
- `./incoming/` for received payloads  
- `./daily-job-recommendations/` for pipeline outputs  

---

## n8n Integration

**Incoming:** n8n sends POST requests to `https://<space>/webhook` with student JSON.  
**Outgoing:** The FastAPI app POSTs ranked results to `RESULTS_WEBHOOK_URL`, where n8n emails the student.

**n8n HTTP Request Node (sending to server)**  
- Method: POST  
- URL: `https://<space-domain>/webhook`  
- Headers: `Content-Type: application/json`  
- Body: JSON (student profile)

**n8n Webhook Node (receiving results)**  
- Method: POST  
- Expects ranked JSON to trigger email delivery.

---

## Ownership

**Owner:** Designed and implemented all CrewAI agents, tools, prompts, API server, Docker/HF Spaces deployment, and n8n integration.  
**Teammate:** Handles downstream email delivery via n8n using the webhook payload.
