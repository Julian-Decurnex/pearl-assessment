# Pearl Talent — Candidate Scoring Workflow (A320)

N8N workflow that automatically scores candidates from Google Sheets against the **Recruitment Coordinator (A320)** job description using Groq's free LLM API, then writes structured results to a second sheet tab.

---

## Architecture

```
Manual Trigger
    └─▶ Read "Candidates" sheet
            └─▶ Validate required fields
                    ├─▶ [valid]  Score with Groq LLaMA 3.1 70B
                    │               └─▶ Parse & validate JSON response
                    │                       └─▶ Write to "Results" sheet
                    └─▶ [invalid] Flag skipped row
                                    └─▶ Write SKIPPED to "Results" sheet
```

**Key design decisions:**
- Every row lands in Results — either scored, or flagged as SKIPPED/ERROR — nothing silently disappears
- Weighted score is **recomputed server-side** in the Code node, preventing LLM arithmetic errors from polluting results
- The Filter node's two outputs map to two separate paths, so incomplete rows are always recorded

---

## Scoring Model

| Dimension | Weight | What it measures |
|---|---|---|
| `ashby_score` | 30% | Ashby ATS proficiency (critical — day-one requirement) |
| `coordination_score` | 25% | Recruiting coordination depth and volume |
| `english_score` | 20% | Written/verbal English fluency |
| `startup_score` | 15% | Fast-paced, high-growth environment experience |
| `automation_score` | 10% | Workflow automation exposure |

**Tier thresholds** (applied to weighted `overall_score`):

| Tier | Score |
|---|---|
| Strong Yes | ≥ 7.5 |
| Yes | ≥ 6.0 |
| Maybe | ≥ 4.5 |
| No | < 4.5 |

---

## Setup

### 1. Google Sheets

Create a spreadsheet with two tabs:

**Tab: `Candidates`** — paste the sample data from `candidates_sample.csv`, or use your own with these columns:

```
name | email | location | years_experience | current_role | ats_experience | english_level | startup_experience | automation_experience | remote_experience | summary
```

**Tab: `Results`** — create with these headers in row 1:

```
name | email | location | years_experience | current_role | ats_experience | ashby_score | coordination_score | english_score | startup_score | automation_score | overall_score | tier | top_strengths | main_gap | recommendation | scored_at
```

### 2. N8N Cloud credentials

**Google Sheets OAuth2:**
1. In N8N: Credentials → New → Google Sheets OAuth2 API
2. Follow the OAuth flow to connect your Google account
3. Name it exactly `Google Sheets account`

**Groq API (free tier):**
1. Get a free API key at console.groq.com
2. In N8N: Credentials → New → HTTP Header Auth
3. Name: `Groq API`, Header Name: `Authorization`, Value: `Bearer YOUR_GROQ_API_KEY`

### 3. N8N variable

In N8N Cloud: **Variables** → add:

| Name | Value |
|---|---|
| `SPREADSHEET_ID` | The ID from your Google Sheet URL (`/d/SPREADSHEET_ID/edit`) |

### 4. Import workflow

1. In N8N: **Import from file** → select `workflow.json`
2. Update credentials on the Google Sheets nodes and HTTP Request node if the names differ from above
3. Click **Execute Workflow** to run

---

## Error handling

| Scenario | Behavior |
|---|---|
| Row missing `name` or `summary` | Filtered out → written to Results with `tier: SKIPPED` |
| LLM returns non-JSON or malformed JSON | Code node catches parse error → `tier: ERROR` + error message in `recommendation` |
| LLM returns JSON with missing required fields | Validation in Code node → `tier: ERROR` + lists missing fields |
| Weighted score arithmetic | Recomputed in Code node — LLM value is ignored |
| Groq API timeout | N8N marks execution as failed; row is not written (retryable) |

---

## Sample output

| name | overall_score | tier | top_strengths | main_gap | recommendation |
|---|---|---|---|---|---|
| Camila Ortega | 9.2 | Strong Yes | Ashby expert, startup ops from scratch, automation builder | None | Immediate advance to screening — built the exact system this role requires. |
| Sofia Reyes | 8.1 | Strong Yes | Daily Ashby user, built Zapier automations, remote-native | None | Strong hire — 3 years of direct experience with demonstrated automation impact. |
| Raj Bautista | 7.0 | Yes | Ashby proficiency, remote-first experience, solid ops | Did not build automations independently | Good fit — would need light onboarding on automation ownership. |
| James Tan | 2.8 | No | Entry-level motivation | No Ashby, no startup, limited English, no coordination depth | Does not meet minimum requirements for this role at this time. |

---

## Files

| File | Description |
|---|---|
| `workflow.json` | N8N workflow — import directly |
| `candidates_sample.csv` | 10 realistic test candidates (mix of tiers) |
| `README.md` | This document |

---

## Tech stack

- **N8N Cloud** — workflow orchestration
- **Google Sheets API** — via N8N's native OAuth2 node
- **Groq API** (free tier) — LLaMA 3.1 70B Versatile
- No database, no extra infrastructure — runs entirely in N8N Cloud
