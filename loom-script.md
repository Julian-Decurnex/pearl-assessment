# Loom Script — Pearl Assessment (5-10 min)

---

## [0:00 — 0:30] Intro

> "Hi, I'm [nombre]. In this video I'll walk you through the candidate scoring workflow I built for the Pearl assessment. I'll cover how it works end-to-end, do a live run, and then explain how a non-technical teammate could use and monitor this day-to-day. Let's go."

---

## [0:30 — 2:00] Google Sheet — Candidates tab

> "Everything starts here — a Google Sheet with 10 candidates. Each row has the candidate's name, location, years of experience, a resume summary, a psychometric score, and an English proficiency score. This is the input the workflow reads from."

Scrolleá las filas, señalá las columnas.

> "I made these candidates realistic — you've got strong fits, borderline cases, and clear rejections, so we can see the scoring work across the full range."

---

## [2:00 — 4:00] Workflow en N8N

> "This is the workflow in N8N Cloud. Let me walk through each step."

Clickeá cada nodo mientras explicás:

- **Manual Trigger** → "Kicks off the workflow. In production this could be a webhook or a form submission trigger."
- **Read Candidates Sheet** → "Reads all rows from the Candidates tab via Google Sheets OAuth."
- **Validate Required Fields** → "Filters out any rows missing a name or resume summary — so bad data doesn't reach the LLM."
- **Loop Over Items** → "Processes candidates one at a time. This is intentional — Groq's free tier has a token-per-minute limit, so we batch size 1 and add a wait between calls."
- **Score with Groq LLM** → "This is the core. We call LLaMA 3.3-70B via Groq's free API with a structured prompt that includes the full job description and all candidate fields. The model returns JSON with three dimension scores and a flag."
- **Parse & Validate Score** → "This Code node parses the LLM response, catches any malformed JSON, and — importantly — recomputes the weighted overall score here in code rather than trusting the LLM's arithmetic."
- **Write to Results Sheet** → "Appends one row per candidate to the Results tab."
- **Wait** → "5 second pause before looping to the next candidate — keeps us within rate limits."

---

## [4:00 — 6:00] Live run

> "Let me run it live."

Ejecutá el workflow (o mostrá el historial de ejecuciones si ya corrió).

> "You can see it processing one candidate at a time — each loop iteration hits Groq, parses the response, and writes to the sheet."

Mostrá la Results tab:

> "Each candidate gets an overall score from 0 to 100, broken down across three dimensions: technical skills — which weights Ashby ATS proficiency heavily — experience relevance, and communication indicators. Then a one-sentence explanation and a flag: Qualified, Needs Review, or Unqualified."

Señalá un Qualified y un Unqualified, leé la match_explanation de cada uno.

---

## [6:00 — 8:00] Para un teammate no técnico

> "Now, if I had to hand this off to someone on the team who doesn't write code — here's what they need to know."

> "To run it: open N8N, click Execute. That's it. The workflow reads the Sheet, scores everyone, and writes results automatically."

> "To add new candidates: just add rows to the Candidates tab in the Sheet and run the workflow again. Don't touch anything in N8N."

> "If something breaks: N8N logs every execution here — you can see which candidate failed and what the error was. The most common issue would be a Groq rate limit, which shows up as an error on the Groq node. Fix: just re-run."

> "The Results tab is append-only, so if you run twice you'll get duplicate rows — just delete the old ones before re-running."

---

## [8:00 — 9:00] Cierre — qué mejorarías

> "A few things I'd add with more time: a webhook trigger so it runs automatically when a new candidate is added to the Sheet, retry logic on the Groq node for transient errors, and a Slack notification when the run finishes with a summary of how many Qualified came through."

> "At scale I'd move to a paid LLM tier and a proper queue, but for the volume Pearl operates at today, this runs for under a dollar per thousand candidates."

> "That's it — thanks for watching."

---

## Tips para grabar

- Usá Loom Desktop para compartir pantalla completa
- Tené la Sheet y N8N abiertos en pestañas separadas, alternás entre las dos
- Si el workflow ya corrió, mostrá el historial de ejecuciones en vez de esperar que corra en vivo — es más limpio
- Grabalo en inglés
