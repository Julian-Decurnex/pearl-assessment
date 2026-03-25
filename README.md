# Pearl Talent — Candidate Scoring Workflow (A320 Recruitment Coordinator)

N8N workflow that reads candidates from Google Sheets, scores them against the Recruitment Coordinator (A320) JD using Groq's free LLM API, and writes structured results to a Results tab.

---

## What I Built and How It Works

**Stack:** N8N Cloud · Google Sheets · Groq (LLaMA 3.3-70B, free tier)

The workflow runs end-to-end in N8N Cloud:

1. **Read** — Google Sheets node reads the `Candidates` tab (10 rows: name, resume summary, years of experience, psychometric score, English proficiency score)
2. **Validate** — Filter node drops rows missing name or resume_summary, routing them to an error path
3. **Loop** — Loop Over Items (batch size 1) processes candidates sequentially with a 5-second Wait between calls to stay within Groq's 12,000 TPM free-tier rate limit
4. **Score** — HTTP Request node calls Groq (LLaMA 3.3-70B) with a structured prompt that includes the full JD and all candidate fields. The model is instructed to return only valid JSON with: `overall_score` (1-100), three dimension scores (`technical_skills`, `experience_relevance`, `communication_indicators`), a one-sentence `match_explanation`, and a `flag` (Qualified / Needs Review / Unqualified)
5. **Parse & validate** — Code node (JS) parses the LLM JSON, catches malformed responses, recomputes the weighted overall score server-side to prevent LLM arithmetic drift, and assigns the flag deterministically
6. **Write** — Google Sheets Append node writes one row per candidate to the `Results` tab

```
Manual Trigger → Read Candidates → Validate → Loop Over Items
                                                    │ (loop)
                                                    ▼
                                          Score with Groq LLM
                                                    ▼
                                          Parse & Validate Score
                                                    ▼
                                          Write to Results Sheet
                                                    ▼
                                                 Wait 5s
                                                    │
                                                    └──▶ Loop Over Items (input)
```

**Scoring model:**

| Dimension | Weight | What it measures |
|---|---|---|
| `technical_skills` | 35% | ATS proficiency (Ashby critical), tools, automation capability |
| `experience_relevance` | 40% | Years in recruiting coordination, startup exposure, role history fit |
| `communication_indicators` | 25% | English proficiency score + resume communication quality |

| Flag | Score |
|---|---|
| Qualified | ≥ 70 |
| Needs Review | 45–69 |
| Unqualified | < 45 |

---

## What I'd Improve With More Time

- **Trigger on form submit / sheet row added** instead of manual — fully automated intake
- **Retry logic** for transient Groq errors (currently a failed call stops the loop item)
- **Prompt versioning** — store the JD and scoring rubric in a separate sheet cell so it's editable without touching the workflow
- **Confidence score** — ask the LLM to self-report certainty; flag low-confidence scores for human review regardless of tier
- **Deduplication** — check if candidate email already exists in Results before scoring again

---

## What Broke or Surprised Me

- **LLM decommission mid-build:** `llama-3.1-70b-versatile` was deprecated while building. Swapped to `llama-3.3-70b-versatile` — no prompt changes needed.
- **Rate limiting:** Free Groq tier caps at 12,000 TPM. All 10 candidates hitting the API simultaneously blew through the limit in one shot. Fix: Loop Over Items (batch=1) + 5s Wait between calls.
- **N8N item pairing:** In "Run Once for Each Item" mode, referencing upstream nodes requires careful attention to which node name you use — a wrong reference silently returns the wrong item's data.
- **Google Sheets Append mode:** Imported workflow JSON doesn't preserve the write operation type — had to manually set "Append" and "Map Automatically" after import.

---

## How I'd Make This Production-Ready

**Reliability**
- Move to a paid LLM tier (Groq Dev or OpenAI) to remove rate limit constraints and add SLA guarantees
- Add N8N error workflow: any failed execution triggers a Slack alert with the candidate name and error message
- Store raw LLM responses alongside parsed results for audit trail

**Cost**
- Current free-tier cost: $0. At scale, LLaMA 3.3-70B on Groq Dev is ~$0.59/1M tokens — at ~750 tokens/candidate, scoring 1,000 candidates/month costs under $1
- If volume grows: batch candidates in groups of 5 per LLM call to reduce API overhead

**Scale**
- Replace manual trigger with a webhook or Google Sheets `onChange` trigger for real-time processing
- For 1,000+ candidates/day: move to N8N self-hosted on a small VM + a queue (Redis or SQS) to decouple ingestion from scoring

**Monitoring**
- N8N Cloud execution logs cover basic observability
- Add a summary row at the bottom of Results with: run timestamp, total processed, error count, average overall score
- Weekly digest email (N8N + Gmail node) to recruiting team with tier distribution

---

## Files

| File | Description |
|---|---|
| `workflow.json` | N8N workflow export — import directly into N8N Cloud |
| `candidates_sample.csv` | 10 realistic test candidates covering all flag tiers |
| `README.md` | This document |

---

## Setup

### 1. Google Sheets

Create a spreadsheet with two tabs:

**Tab: `Candidates`** — headers in row 1:
```
name | email | location | years_experience | resume_summary | psychometric_score | english_proficiency_score
```

**Tab: `Results`** — headers in row 1:
```
name | email | location | years_experience | overall_score | technical_skills | experience_relevance | communication_indicators | match_explanation | flag | scored_at
```

### 2. Groq API key (free)
1. Sign up at **console.groq.com**
2. API Keys → Create API Key

### 3. N8N Cloud credentials

**Groq:** Credentials → New → HTTP Header Auth
- Name: `Groq API`
- Header Name: `Authorization`
- Value: `Bearer YOUR_KEY`

**Google Sheets:** Credentials → New → Google Sheets OAuth2 API
- Name: `Google Sheets account`
- Sign in with Google

### 4. N8N variable
Variables → Add: `SPREADSHEET_ID` = the ID from your Sheet URL

### 5. Import and run
Import `workflow.json` → set credentials on Google Sheets and HTTP Request nodes → Execute Workflow
