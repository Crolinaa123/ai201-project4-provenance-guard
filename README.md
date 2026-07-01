# ai201-project4-provenance-guard

# Provenance Guard

A backend system for classifying whether submitted creative content (poems, stories, blog posts) was human-written or AI-generated. Built for creative sharing platforms that want to surface transparent attribution labels without policing creativity.

## How to Run

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# Add your Groq API key to .env: GROQ_API_KEY=your_key_here
python3 app.py
```

The server runs on `http://localhost:5000`.

---

## API Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/submit` | POST | Submit text for attribution analysis |
| `/appeal` | POST | Contest a classification |
| `/log` | GET | View recent audit log entries |
| `/status/<id>` | GET | Get status of a specific submission |

### POST /submit
**Input:** `{ "text": "...", "creator_id": "..." }`  
**Output:** `{ "content_id", "attribution", "confidence", "label_text", "signals": { "llm_score", "stylo_score" } }`

### POST /appeal
**Input:** `{ "content_id": "...", "creator_reasoning": "..." }`  
**Output:** `{ "appeal_id", "content_id", "status": "under_review", "message" }`

---

## Detection Pipeline

### Why two signals?

A single signal can only capture one dimension of text. The LLM classifier reads holistically — it catches semantic patterns, tone, and phrasing — but it has no sense of statistical structure. Stylometric heuristics measure structure but are blind to meaning. Together they are genuinely independent, and disagreement between them is informative: it means the content is ambiguous, which should produce an uncertain label, not a forced binary.

### Signal 1: LLM Semantic Classifier (Groq / llama-3.3-70b-versatile)

Sends the text to the Groq API with a structured prompt asking for a JSON response with `ai_probability` (0–1). The model assesses holistic properties: whether the phrasing, structure, and voice match AI generation patterns.

**Why it works:** AI models produce consistently coherent, well-organized prose with characteristic hedging ("It is important to note…") and balanced sentence structure. Human writing has idiosyncrasies, tangents, and tonal inconsistency that the LLM is trained to distinguish.

**Blind spot:** Skilled human writers who write formally (academic, journalistic) can score high AI probability. Heavily human-edited AI text may score low.

### Signal 2: Stylometric Heuristics (pure Python)

Computes three structural statistics and averages them into a single `stylo_score`:

1. **Sentence-length variance** — low variance → AI-like (normalized: `1 - std_dev / 15`)
2. **Type-token ratio** — low vocabulary diversity → AI-like (`1 - unique_words / total_words`)
3. **Punctuation density** — density near the average range (0.15–0.25) → AI-like

**Why it works:** AI text is statistically more uniform — consistent sentence lengths, moderate vocabulary, predictable punctuation. Human text shows more variance across all three dimensions.

**Blind spot:** Texts under ~80 words lack enough sentences for meaningful statistics (returns 0.5 as a neutral fallback). Genre-uniform human writing (legal briefs, technical specs) resembles AI structurally.

### Confidence Scoring

```
confidence = 0.6 × llm_score + 0.4 × stylo_score
```

The LLM signal is weighted higher (60%) because it captures semantic patterns the heuristics miss. The stylometric signal acts as a structural check — it is most useful when the LLM is uncertain.

**Thresholds:**

| Confidence | Attribution | Label variant |
|---|---|---|
| 0.00 – 0.35 | human | High-confidence human |
| 0.36 – 0.74 | uncertain | Uncertain |
| 0.75 – 1.00 | ai_generated | High-confidence AI |

The AI threshold is intentionally high (0.75) because a false positive — labeling a human's work as AI-generated — is worse than a false negative on a creative platform.

### Two Example Submissions

**Clearly AI-generated paragraph:**
> "Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous, it is equally essential to consider the ethical implications."

- `llm_score`: 0.80, `stylo_score`: 0.50, `confidence`: **0.68** → Uncertain

**Clearly human-written (casual voice):**
> "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it..."

- `llm_score`: 0.20, `stylo_score`: 0.56, `confidence`: **0.34** → Human

The scores differ by 0.34 — a meaningful spread, not noise.

---

## Transparency Labels

All three label variants, exactly as returned by the API:

**High-confidence human** (confidence ≤ 0.35):
> "This content appears to be human-written. Our system found no strong indicators of AI generation."

**Uncertain** (0.35 < confidence < 0.75):
> "Our system is uncertain about the origin of this content. It may be human-written, AI-generated, or a mix. If you are the creator and believe this label is wrong, you can submit an appeal."

**High-confidence AI** (confidence ≥ 0.75):
> "Our system detected strong indicators that this content was AI-generated. If you are the creator and believe this is incorrect, you can submit an appeal."

---

## Appeals Workflow

Any creator can contest a classification by calling `POST /appeal` with the `content_id` and their `creator_reasoning`. The system:

1. Looks up the original audit log entry
2. Rejects duplicate appeals (one per submission)
3. Appends `appeal_id`, `appeal_reason`, and `appealed_at` to the log entry
4. Updates `status` from `"classified"` to `"under_review"`
5. Returns confirmation to the caller

No automated re-classification occurs. A human moderator reviews appeals via `GET /log`.

---

## Rate Limiting

Applied to `POST /submit` only (the expensive endpoint):

- **10 requests per minute per IP**
- **50 requests per day per IP**

**Reasoning:** A legitimate creator submitting their own work would rarely hit 10 submissions in a minute. The per-minute limit stops burst flooding (automated scripts). The per-day limit (50) allows a prolific creator to submit throughout the day while preventing sustained automated abuse. Read endpoints (`/log`, `/status`) are not rate-limited — they are cheap and needed for appeals workflows.

**Rate limit behavior:** Requests over the limit return HTTP 429.

To verify rate limiting is working:
```bash
for i in $(seq 1 12); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/submit \
    -H "Content-Type: application/json" \
    -d '{"text": "rate limit test", "creator_id": "ratelimit-test"}'
done
```
First 10 return `200`; requests 11 and 12 return `429`.

---

## Audit Log

Every attribution decision is written to a SQLite database (`provenance.db`). Each entry includes:

```json
{
  "content_id": "f1b86e04-8386-4437-a9ea-1a576d83cc86",
  "creator_id": "test-user-1",
  "timestamp": "2026-07-01T03:44:44.576169+00:00",
  "attribution": "human",
  "confidence": 0.32,
  "llm_score": 0.2,
  "stylo_score": 0.5,
  "label_text": "This content appears to be human-written. Our system found no strong indicators of AI generation.",
  "status": "under_review",
  "appeal_id": "840114d2-31a6-4d22-977c-9fc5ecb4aac0",
  "appeal_reason": "I wrote this myself from personal experience.",
  "appealed_at": "2026-07-01T03:45:21.389049+00:00"
}
```

View recent entries: `GET /log` (returns last 20 by default, configurable with `?limit=N`).

---

## Known Limitations

**Short creative poems** are the hardest case. A haiku or short poem has 1–3 sentences, so the stylometric signal falls back to a neutral 0.5. The LLM classifier has to carry the whole decision, and short formal poetry often reads as AI-like to language models. The system correctly flags these with `short_text_warning: true`, but the confidence score is less reliable. A better approach would be a separate short-text classifier trained on poetry specifically.

**Repetitive-structure human poetry** (anaphora, refrain-based forms) produces artificially uniform sentence lengths and low vocabulary diversity — both AI-like stylometric signals. The LLM may also score it high. These submissions will likely land in the Uncertain band, which is the honest answer. The appeal path exists precisely for this case.

---

## Spec Reflection

**Where the spec helped:** The planning.md requirement to write out the false positive scenario before writing any code forced a concrete design decision — the AI threshold should be 0.75, not 0.5. If I had started coding without thinking through that scenario, I might have defaulted to a binary 0.5 split and produced a system that falsely accused human writers routinely.

**Where implementation diverged:** The spec called for the stylometric signal to use punctuation density as a strong AI indicator. In practice, punctuation density alone produced scores that clustered near 0.5 for most inputs — it wasn't discriminating enough on its own. The final implementation combines it with sentence-length variance and type-token ratio as an average, which produced more meaningful spread across test inputs.

---

## AI Usage

**Instance 1 — Detection pipeline architecture:** I provided my `planning.md` detection signals section and architecture diagram to Claude and asked it to generate the Flask app skeleton with `POST /submit`, the LLM classifier function, and the SQLite audit log. The generated code had the correct structure but used `storage_uri` without `"memory://"`, causing a Flask-Limiter warning on startup. I corrected this before running.

**Instance 2 — Stylometric signal function:** I asked Claude to implement `compute_stylometric_score()` based on my spec's description of the three sub-metrics (sentence-length variance, type-token ratio, punctuation density). The generated function normalized each metric to 0–1 and averaged them, which matched my spec. I tested it independently on 4 inputs before wiring it into the endpoint to verify the scores were meaningful and independent from the LLM scores.
