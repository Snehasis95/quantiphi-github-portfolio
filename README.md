# ⟨/⟩ DevFolio — GitHub Developer Portfolio

A professional developer dashboard that turns a GitHub profile into a **personal branding page**. It shows coding activity, repository stats, top languages and streaks, scores how well a developer's stack fits a job description, and includes an AI portfolio assistant that answers recruiter questions.

> Built for the Quantiphi Vibe Coding Round. The backend is **FastAPI + SQLite** and the frontend is **Vanilla JS + Chart.js**. All business logic runs on the server.

---

## ✨ Features

### Core
| Feature | Description |
|---|---|
| **Contribution Graph** | A GitHub-style heatmap of daily activity over the last 365 days. Each day gets an intensity level from 0 to 4, computed on the server from quartiles of the active days. |
| **Coding Streaks** | Current streak and longest streak. The current streak stays alive if you haven't committed yet today. |
| **Repository Cards** | Stars, forks, language tag (in GitHub's color) and description. Sort by stars, recently updated, forks or name, and choose whether to include forks. |
| **Profile Header** | Avatar, bio, follower count, total stars (forks excluded) and a **Top Languages pie chart** weighted by bytes of code. |
| **GitHub API Integration** | Uses **GraphQL** when a token is set: one request returns the profile, repos, languages and contribution calendar. Without a token it falls back to **REST**. |
| **SQLite Caching Engine** | A read-through cache with a TTL that reduces rate-limit issues and speeds up page loads. If GitHub is unavailable, the app serves stale cached data instead of an error. |

### 🎯 The Vibe Check: Tech Stack Compatibility
Paste a job description and the backend extracts the technologies it asks for, compares them with the languages in the user's **cached** repositories, and shows a **Match Percentage badge** at the top of the profile. Green means 70% or more, amber means 40% or more, and red means below 40%.

### 🤖 Extra feature 1: Portfolio Chatbot
A chat widget where recruiters can ask things like *"What are the top projects?"*, *"Which repos use Python?"* or *"How active is this developer?"*
- **Rule-based intent engine** that works offline and needs no API key. It answers from the SQLite cache.
- **Optional LLM mode:** if `ANTHROPIC_API_KEY` is set, answers come from an LLM that is given only the developer's cached facts, which keeps it grounded. If the LLM call fails, it falls back to the rule-based engine.

### 💡 Extra feature 2: AI Career Insights / Skill-Gap Advisor
Built on top of the match result, it gives:
- An overall **verdict** on fit for the role
- **Strengths:** matched skills, each with the share of the user's code written in it
- **Skill gaps**, each with a concrete project idea to close it
- **Repos to pin:** the best repos that show the skills the job needs

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────┐
│  Frontend  (HTML · CSS · JS · Chart.js)       │  presentation + user interaction only
└───────────────────────┬──────────────────────┘
                        │ JSON over /api/*
┌───────────────────────▼──────────────────────┐
│  Routes  (FastAPI)                           │  input validation, HTTP status mapping
├──────────────────────────────────────────────┤
│  Services                                    │  ALL business logic
│   • stats.py      stars, languages, heatmap, streaks, sorting
│   • matcher.py    job description skill extraction + match scoring
│   • assistant.py  chatbot + career insights
├──────────────────────────────────────────────┤
│  cache.py  (SQLite, TTL)                     │──miss/expired──► github_client.py
└──────────────────────────────────────────────┘                  (GraphQL / REST)
```

**Design principle:** the browser never computes anything. Totals, percentages, heatmap levels, streaks, sorting, match scores, chatbot replies and insights all come from the API, ready to render.

### Data flow
1. The user enters a username, and the frontend calls `GET /api/profile/{username}`.
2. `cache.py` checks `users.fetched_at`. If it is within the TTL, data is served from SQLite (a cache hit).
3. Otherwise `github_client.py` fetches from GitHub, normalizes the response, and the cache **upserts** it. Old rows are replaced in one transaction, so deleted repos disappear.
4. The repos, contributions and match endpoints then read from the warm cache.

---

## 🗄️ Caching Engine (SQLite)

| Table | Purpose |
|---|---|
| `users` | Profile data and `fetched_at` timestamp, which drives the TTL |
| `repos` | Repository metadata (stars, forks, language, color, fork flag) |
| `repo_languages` | Bytes of code per language per repo, used for an accurate pie chart |
| `contributions` | Daily contribution counts |
| `settings` | Key/value user settings, such as the last match result so the badge persists across reloads |

- **TTL:** `CACHE_TTL_SECONDS`, default 3600 (1 hour).
- **Forced refresh:** add `?refresh=true`, or use the ↻ Refresh button.
- **Graceful degradation:** if GitHub rate-limits or errors, the app serves stale data with a warning in the UI.
- **Zero-quota features:** match, chat and insights read only from the cache, so they never call GitHub.

---

## 🧮 Match Algorithm

1. **Skill extraction.** A dictionary maps each language to its aliases, for example `C++ ← c++, cpp`, `Go ← golang`, `C# ← csharp, .net`.
   - Word-boundary regex prevents false positives: `go` does not match "good", and `java` does not match "javascript".
   - Ambiguous single-letter names (`C`, `R`, `Go`) are matched **case-sensitively** and only as standalone words.
2. **Framework inference.** Frameworks count as evidence for a language: `React → JavaScript`, `Django → Python`, `Spring → Java`, `Flutter → Dart`.
3. **Equivalents.** Related languages give credit: TypeScript satisfies JavaScript, and Jupyter Notebook satisfies Python.
4. **Scoring.**
   - **Match %** = matched required skills ÷ total required skills × 100
   - **Depth %** = share of the user's code written in the matched languages
5. **Forks are excluded**, so only the developer's own work counts.

---

## 📁 Project Structure

```
github-portfolio/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI app, CORS, router wiring, serves frontend
│   │   ├── config.py            # env config (token, TTL, DB path, LLM key)
│   │   ├── db.py                # SQLite connection + schema
│   │   ├── routes/
│   │   │   ├── deps.py          # username validation, error mapping
│   │   │   ├── profile.py       # GET /api/profile/{username}
│   │   │   ├── repos.py         # GET /api/repos/{username}
│   │   │   ├── contributions.py # GET /api/contributions/{username}
│   │   │   ├── match.py         # POST /api/match, GET /api/match/{username}
│   │   │   └── assistant.py     # POST /api/chat, POST /api/insights
│   │   └── services/
│   │       ├── github_client.py # GraphQL + REST fallback, normalized output
│   │       ├── cache.py         # TTL cache + settings store
│   │       ├── stats.py         # derived metrics
│   │       ├── matcher.py       # compatibility engine
│   │       └── assistant.py     # chatbot + career insights
│   └── requirements.txt
├── frontend/
│   ├── index.html
│   ├── styles.css
│   └── app.js
├── .env.example
├── .gitignore
└── README.md
```

---

## 🚀 Getting Started

### Prerequisites
- Python 3.10 or later
- A GitHub Personal Access Token (classic, `read:user` scope). It is needed for the full contribution heatmap.

### Setup
```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>

cp .env.example .env        # then paste your GITHUB_TOKEN into .env

cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Then open:
- **App:** http://localhost:8000
- **Interactive API docs (Swagger):** http://localhost:8000/docs

You can also open a profile directly with `http://localhost:8000/?user=<username>`.

### Environment variables
| Variable | Required | Default | Description |
|---|---|---|---|
| `GITHUB_TOKEN` | Recommended | — | Enables GraphQL and the full contribution calendar, with higher rate limits |
| `CACHE_TTL_SECONDS` | No | `3600` | How long cached data stays fresh |
| `ANTHROPIC_API_KEY` | No | — | Turns on LLM answers in the chatbot |
| `LLM_MODEL` | No | `claude-haiku-4-5` | LLM model name |
| `DB_PATH` | No | `backend/cache.db` | SQLite file location |

> Without a token, the app still works through the REST API. In that mode the heatmap is approximated from public events (about the last 90 days) and the rate limit is 60 requests per hour.

---

## 📡 API Reference

| Method | Endpoint | Body / Query | Returns |
|---|---|---|---|
| GET | `/api/health` | — | Status and auth mode |
| GET | `/api/profile/{username}` | `refresh=true\|false` | Profile, total stars and forks, top languages, cache metadata |
| GET | `/api/repos/{username}` | `sort=stars\|updated\|forks\|name`, `include_forks`, `limit` | Repo cards |
| GET | `/api/contributions/{username}` | — | Heatmap weeks with `level` 0–4, total, active days, streaks |
| POST | `/api/match` | `{ "username", "job_description" }` | Match %, depth %, badge label and color, matched and missing skills |
| GET | `/api/match/{username}` | — | Last saved match result |
| POST | `/api/chat` | `{ "username", "message" }` | `{ reply, engine: "rules" \| "llm" }` |
| POST | `/api/insights` | `{ "username", "job_description" }` | Match, verdict, strengths, gaps, repos to pin |

### Example
```bash
curl -X POST http://localhost:8000/api/match \
  -H "Content-Type: application/json" \
  -d '{"username":"torvalds","job_description":"Looking for a C developer with Linux kernel and Python experience"}'
```

---

## 🛡️ Validation & Error Handling
- Usernames are validated against GitHub's rules (alphanumeric and hyphens, at most 39 characters). Invalid input returns `400`.
- Job descriptions are limited to 20,000 characters and chat messages to 500, enforced by Pydantic and the services. Invalid input returns `400` or `422`.
- GitHub errors are mapped to clear HTTP codes: `404` for user not found, `429` for rate limit, `401` for a bad token.
- The frontend escapes all user and GitHub content before rendering to prevent XSS.

---

## 🧰 Tech Stack
**Backend:** FastAPI · Pydantic · httpx (async) · SQLite · python-dotenv
**Frontend:** HTML5 · CSS3 (dark GitHub-style theme, responsive) · Vanilla JavaScript · Chart.js
**APIs:** GitHub GraphQL v4 · GitHub REST v3 · Anthropic Messages API (optional)

---

## 🔮 Future Improvements
- Pagination beyond 100 repositories
- Contribution data for multiple years
- Background cache refresh before the TTL expires
- Exporting the portfolio as a PDF or shareable image
- Matching on frameworks and tools from repo topics, not just languages
