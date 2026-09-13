# OpportunityHunter · codename **DARKMATTER**

> **GLOBAL OPPORTUNITY INTELLIGENCE** — One agent. Hundreds of sources. Better opportunities.

An AI agent that searches the internet's *hidden opportunity graph* to find jobs, clients and companies **before** they show up on conventional job boards. Like dark matter, most opportunities are invisible if you only look at LinkedIn and Indeed — you detect them through the ecosystems around them.

Built for the **Multi-App AI Agent Hackathon 2026**.

---

## 1. Problem
Job seekers and freelancers search the same few job boards as everyone else. By the time a role is posted there, hundreds of people have applied. Many companies (trading firms at university career fairs, startups posting in founder threads, local businesses that need software) never show up on those boards at all.

## 2. Solution
OpportunityHunter is **not another job scraper**. It's a multi-step agent that:
1. parses what you want (skills, industries, city),
2. plans searches across *ecosystems*,
3. queries several external apps in parallel,
4. deduplicates companies across sources,
5. researches each company's public website for careers pages and hiring systems (ATS),
6. scores every opportunity 0–100 with a written explanation,
7. writes them to **Google Sheets** (with a CSV fallback) and a live dashboard.

## 3. Why conventional job search falls short
| Job boards | OpportunityHunter |
|---|---|
| One source | Many ecosystems, run in parallel |
| Only posted roles | Company leads, founder posts, local client leads |
| Keyword match | Explainable score with a source URL for each claim |
| Everyone sees the same list | Finds companies through university, trading, OSS, maps and community signals |

```text
SOURCE  →  COMPANY  →  PEOPLE  →  HIRING SIGNAL  →  OPPORTUNITY
```

## 4. Architecture
```text
React + TS + Tailwind dashboard  ──SSE──▶  Node API (zero dependencies)
                                            │
USER REQUEST → INTENT PARSER (Claude | rules)
            → SEARCH PLANNER (ecosystem queries)
            → MULTI-SOURCE DISCOVERY (Promise.allSettled + per-source timeout)
            → DEDUPLICATION (normalized company names, merged signals/sources)
            → COMPANY WEBSITE RESEARCH (careers link, ATS, generic public inbox)
            → QUALIFICATION + EXPLAINABLE SCORING
            → GOOGLE SHEETS / CSV
            → DASHBOARD (live trace, table, company intelligence, source graph)
```
| File | Role |
|---|---|
| `server/agent.mjs` | Intent → plan → discover → dedup → research → score |
| `server/sources.mjs` | **Source registry**: each connector is one object `{id, app, weight, configured(), search()}` |
| `server/sheets.mjs` | Google Sheets append (service-account JWT, no SDK) + CSV |
| `server/demo.mjs` | Labelled demo dataset |
| `server/index.mjs` | HTTP + Server-Sent Events agent trace |
| `web/src/App.tsx` | Dashboard |
| `tests/reliability.test.mjs` | Reliability & evaluation suite |

## 5. Agent workflow (multi-step)
The trace is streamed live to the **Agent Console**: intent → search plan (15+ queries) → per-source status cards (running / ✓ count + latency / ✕ failed / ○ not configured) → dedup → website research → ranked results.

## 6. External apps
| App | Used for | Auth | Status without keys |
|---|---|---|---|
| **GitHub REST API** | Organizations located in the city, filtered by language; org website/email | optional `GITHUB_TOKEN` | ✅ live |
| **Hacker News (Algolia API)** | “Who is hiring” founder/engineer posts mentioning the city | none | ✅ live |
| **OpenStreetMap Overpass API** | Map-based company discovery (IT/financial/research offices with websites) | none | ✅ live (sometimes rate-limited, handled) |
| **Arbeitnow API** | EU job-board vacancies | none | ✅ live |
| **Remotive API** | Remote vacancies open to the region | none | ✅ live |
| **Company websites** | Careers page + ATS detection (Greenhouse, Lever, Ashby, Workable, Recruitee…) | none | ✅ live |
| **Google Maps Places API** | Business discovery (“fintech companies Amsterdam”, “restaurants Amsterdam”) | `GOOGLE_MAPS_API_KEY` | ○ needs key |
| **Web Search (Tavily / Serper)** | University career fairs, VC portfolios, hackathon sponsors, trading firms, X posts | `TAVILY_API_KEY` or `SERPER_API_KEY` | ○ needs key |
| **Google Sheets API** | Operational database (21 columns) | service account | ○ needs creds → CSV fallback |
| **Anthropic Claude** | Intent parsing | `ANTHROPIC_API_KEY` | ○ rule-based fallback |

The **Health** panel shows exactly which apps are available (`X/Y apps available`). A source counts as searched only if its connector actually ran.

## 7. Source ecosystem
```text
Do not only search job boards.
Search the ecosystems that reveal companies:

University career fairs · University employer PDFs · Google Maps · Google Search
Hackathons · Hackathon sponsors · VC portfolios · Accelerators
Conference exhibitors · Conference speakers · Trading ecosystems · Exchange ecosystems
CME ecosystem · DMCC/public business sources · Singapore company sources
GitHub · X · ATS platforms · Company career pages · Remote job boards
Upwork · Remote Rocketship · We Work Remotely · Startup directories
```
All 28 are listed in `SOURCE_REGISTRY`. Each one is marked **live**, **needs key** (sent through web search), **planned** or **human-in-the-loop** (LinkedIn). To add a source, add one object to `LIVE_SOURCES`.

Nothing is hardcoded to Amsterdam: location, country, skills and categories are inputs (the tests use Singapore).

## 8. Opportunity scoring (0–100, explainable)
| Component | Max | Signal |
|---|---|---|
| Technical fit | 25 | Skills matched as whole words in role, description or website |
| Company relevance | 20 | Industry match (Quant ↔ trading/market making, Crypto ↔ blockchain…) |
| Hiring signal | 20 | Careers page, active vacancy, ATS detected |
| International potential | 10 | Remote / visa / relocation / English wording |
| Recent activity | 10 | Post or org update in the last 30 / 120 days |
| Unconventional source | 10 | Source weight (founder threads, maps and OSS rank above job boards) + multi-source bonus |
| Contactability | 5 | Public careers URL, website, generic public inbox |

Each opportunity has a `why[]` list, a full score breakdown shown in Company Intelligence, and source URLs. HIGH ≥ 75, MEDIUM ≥ 55.

## 9. Reliability testing
```bash
npm test                 # 7 tests, includes one real live end-to-end run
OH_OFFLINE=1 npm test    # offline only
```
| Test | Checks |
|---|---|
| Intent parser | Works for any city (Singapore), extracts C++ / AI / Trading / Crypto |
| Planner | Produces university, trading, VC, hackathon and X ecosystem queries |
| Score invariants | 0 ≤ score ≤ 100, explanation not empty, **components sum to total** |
| Precision regression | “AI” no longer matches “em**ai**l” (a real bug the tests caught) |
| CSV contract | All 21 Sheet columns; demo rows are tagged `DEMO DATA` |
| **Chaos: all sources fail** | `OH_FORCE_FAIL=...` → the agent returns an empty result, no crash |
| **Live E2E** | ≥ 3 distinct external apps return data; every opportunity has a source URL |

Latest run: **7/7 pass**. The live test reached Remotive (8), Arbeitnow (1), Hacker News (5) and GitHub (8), giving 16 scored opportunities. OpenStreetMap was rate-limited in that run, and the agent continued without it.

Reliability design: `Promise.allSettled` fan-out, 35 s timeout per source, per-org enrichment failures tolerated, unreachable websites keep the lead, Claude errors fall back to rules, Sheets errors fall back to CSV with the real error shown. **A Sheets success message appears only after a real API write.**

Chaos-test the running app: `OH_FORCE_FAIL=github,openstreetmap npm start`. The console shows ✕ on those sources and “Degraded gracefully — continued with: …”.

Findings from building and testing: the first HN parser counted *candidates* (“Who wants to be hired”, `Location: … Remote: …`) as employers, and substring matching inflated industry matches. Both are fixed and covered by tests.

## 10. Demo (≤ 2 min)
| Time | Action |
|---|---|
| 0:00 | “I don't search for jobs. I search for the ecosystems that contain opportunities.” |
| 0:15 | Agent Console → type *Find Python/C++ opportunities in Amsterdam for someone interested in AI, trading and crypto.* |
| 0:35 | Hunt → the search plan and source cards stream in (LIVE mode shows real latencies) |
| 0:55 | Top opportunities with HIGH/MEDIUM score badges → Opportunities tab |
| 1:20 | Click a company → Company Intelligence: why we found it, hiring signals, score breakdown, source URLs |
| 1:40 | Google Sheets tab → **SYNC TO GOOGLE SHEETS** |
| 1:50 | Source Graph → “It searches the world's ecosystems for opportunities.” |

**LIVE vs DEMO:** the header toggle switches modes. DEMO replays a paced, labelled sample dataset (amber `DEMO DATA` banner; sheet rows say `DEMO DATA — not real research`) so the video works even without network access. Use LIVE to show the real integrations.

## 11. Setup
Requires Node ≥ 18.
```bash
git clone <this repo> && cd opportunityhunter
cp .env.example .env        # optional keys
npm run setup               # installs + builds the dashboard
npm start                   # http://localhost:8787
```
Dev with hot reload: `npm start` in one terminal, `npm run dev:web` in another → http://localhost:5173

**Google Sheets:** create a service account in Google Cloud → enable the Sheets API → download the JSON to `credentials/service-account.json` → share your sheet with the service-account email as Editor → set `GOOGLE_SHEET_ID`.

## 12. Environment variables
See [.env.example](.env.example): `TAVILY_API_KEY` / `SERPER_API_KEY`, `GOOGLE_MAPS_API_KEY`, `GOOGLE_SHEET_ID`, `GOOGLE_SHEET_TAB`, `GOOGLE_SERVICE_ACCOUNT_FILE`, `GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL`, `OH_FORCE_FAIL`, `PORT`.

## 13. Screenshots
Put them in `screenshots/`: `console.png` (live trace), `opportunities.png`, `company.png`, `sources.png`, `sheets.png`.

## 14. Roadmap
- Extract company names from career-fair PDFs (download → text → named-entity extraction)
- Greenhouse/Lever/Ashby job-list APIs for each detected ATS
- Upwork, We Work Remotely and Remote Rocketship connectors
- LLM-written outreach drafts for review (never auto-sent)
- Deduplication against rows already in the Sheet, plus status tracking (Applied / Interview)
- Scheduled hunts with alerts for new HIGH opportunities

## 15. Privacy & responsible use
- Uses only public APIs and public web pages, with one GET per homepage. No login walls, CAPTCHA bypass, anti-bot evasion or hidden-field extraction.
- **No personal phone numbers or private emails.** Only generic role inboxes (jobs@, careers@, info@) shown publicly on company sites are recorded.
- People research (founder/CTO/recruiter) is **human-in-the-loop**: the UI opens a public LinkedIn search, and nothing is scraped from LinkedIn.
- Secrets live in `.env` / `credentials/` (git-ignored). CSV output goes to `data/private/` (git-ignored).

## 16. Limitations
- Without web-search or Places keys, the university, VC, hackathon, trading and X ecosystems are *planned but not run*, and the UI and health panel say so.
- Scoring is heuristic, not learned. Company-name deduplication is string-normalized and can miss aliases.
- OpenStreetMap coverage and the Overpass rate limits vary. Arbeitnow skews toward Germany.
- The demo dataset lists real public employers with **illustrative** signals. Verify before acting on them.
- Built in a ~35-minute sprint.

---
*OpportunityHunter doesn't search one website for jobs. It searches the world's ecosystems for opportunities.*
