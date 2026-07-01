# Provenance Guard — Planning

## Architecture Narrative

A creator submits text (poem, story excerpt, blog post) via `POST /submit`. The request
first hits the **Rate Limiter** (Flask-Limiter); requests over quota are rejected with
429 before any expensive work happens.

Accepted requests enter the **Detection Pipeline**, which runs two independent signals:

1. **LLM Classifier (Groq / llama-3.3-70b-versatile):** Sends the raw text to the model
   with a structured prompt asking it to assess whether the text reads as human-written
   or AI-generated. Returns a probability score 0–1 toward AI (`llm_score`).

2. **Stylometric Heuristics (pure Python):** Computes structural statistics over the
   text — sentence-length variance, type-token ratio (vocabulary diversity), and
   punctuation density. These properties are more uniform in AI text and more variable
   in human text. Outputs a 0–1 score toward AI (`stylo_score`).

The two scores feed the **Confidence Scorer**, which computes a weighted average
(`confidence = 0.6 * llm_score + 0.4 * stylo_score`). The LLM signal is weighted
higher because it captures semantic patterns the heuristics miss.

The confidence value drives the **Label Generator**, which maps scores to one of
three transparency labels (see Label Variants below).

The full decision record — content ID, timestamp, attribution verdict, confidence,
raw signal scores — is written to the **Audit Log** (SQLite). The structured response
is returned to the caller.

If a creator contests the result, `POST /appeal` captures their reasoning, appends
the appeal to the audit log entry, and sets the content's status to `"under_review"`.
No automated re-classification occurs; a human moderator handles review.

`GET /log` exposes recent audit log entries. `GET /status/:id` returns the current
state of a specific submission.

---

## Detection Signals

### Signal 1: LLM Semantic Classifier
- **Measures:** Holistic semantic and stylistic coherence — whether phrasing, structure,
  and voice patterns match AI generation.
- **Why it differs:** AI models produce consistently coherent, well-organized prose with
  characteristic hedging and balance. Human writing has idiosyncrasies, tangents, and
  tonal inconsistency.
- **Output:** Float 0–1 toward AI. Obtained by prompting Groq with the text and asking
  for a JSON response `{ "ai_probability": 0.82, "reasoning": "..." }`. The
  `ai_probability` field is used directly as `llm_score`.
- **Blind spot:** Skilled human writers who write formally (academic, journalistic)
  can score high AI probability. Heavily human-edited AI text may score low.

### Signal 2: Stylometric Heuristics
- **Measures:** Statistical structural properties: sentence-length variance (std dev of
  word counts per sentence), type-token ratio (unique/total words), punctuation density.
- **Why it differs:** AI text is statistically more uniform across all three dimensions.
  Human text shows more variance — shorter and longer sentences mixed, richer or more
  idiosyncratic vocabulary, varied punctuation habits.
- **Output:** Each sub-metric is normalized to 0–1 (high = AI-like), then averaged into
  a single `stylo_score` float. Sentence-length variance: low variance → high score.
  Type-token ratio: high diversity → low score. Punctuation density: near-average density
  → higher score.
- **Blind spot:** Short texts (< ~100 words) lack sufficient data. Genre-uniform human
  writing (legal briefs, technical specs) resembles AI structurally.

---

## Confidence Scoring & Thresholds

```
confidence = 0.6 * llm_score + 0.4 * stylo_score
```

| Confidence range | Attribution   | Label variant        |
|-----------------|---------------|----------------------|
| 0.00 – 0.35     | Human         | High-confidence human |
| 0.36 – 0.74     | Uncertain     | Uncertain            |
| 0.75 – 1.00     | AI-generated  | High-confidence AI   |

The AI threshold is intentionally high (0.75) because a false positive — labeling a
human's work as AI — is worse than a false negative on a creative platform.

---

## Label Variants

**High-confidence human** (confidence ≤ 0.35):
> "This content appears to be human-written. Our system found no strong indicators
> of AI generation."

**Uncertain** (0.35 < confidence < 0.75):
> "Our system is uncertain about the origin of this content. It may be human-written,
> AI-generated, or a mix. If you are the creator and believe this label is wrong,
> you can submit an appeal."

**High-confidence AI** (confidence ≥ 0.75):
> "Our system detected strong indicators that this content was AI-generated.
> If you are the creator and believe this is incorrect, you can submit an appeal."

---

## Appeals Workflow

**Who can appeal:** Any creator who submitted content (identified by `creator_id`).

**What they provide:** `content_id` (the UUID from the original submission) + free-text
`reason` explaining why they believe the classification is wrong.

**What the system does on receipt:**
1. Looks up the original audit log entry by `content_id`.
2. Returns 404 if not found, 400 if an appeal already exists for this content.
3. Appends an `appeal` object to the log entry: `{ appeal_id, creator_id, reason, appealed_at }`.
4. Sets the entry's `status` field from `"classified"` to `"under_review"`.
5. Returns `{ appeal_id, status: "under_review", message }` to the caller.

**No automated re-classification.** A human moderator opens `GET /log` (filtered to
`status=under_review`), reads the original signals, confidence, and the creator's
reasoning, and makes a manual determination.

**What the moderator sees:** Full audit entry — timestamp, content (or content hash),
`llm_score`, `stylo_score`, combined confidence, original label, and appeal reason.

---

## Anticipated Edge Cases

**1. Short creative poems (< 80 words)**
Stylometric heuristics need enough sentences to compute meaningful variance. A haiku or
short poem has 1–3 sentences — std dev of sentence length is near zero for both human
and AI, making `stylo_score` unreliable. Mitigation: flag short texts in the response
(`"short_text_warning": true`) and weight the LLM signal more heavily (or fall back to
LLM-only) when word count < 80.

**2. Repetitive-structure human poetry**
A human poet who deliberately uses anaphora (repeating a phrase at the start of each
line, e.g. "I am the…" ×8) produces highly uniform sentence lengths and low vocabulary
diversity — both AI-like stylometric signals. The LLM classifier may also score it high
because the repetition resembles templated output. This is a hard case; the system will
likely land in the Uncertain band, which is the honest answer. The appeal path exists
precisely for this.

**3. Lightly edited AI text**
A creator generates AI text and makes minor edits before submitting. The LLM classifier
may catch it (semantic patterns persist through light edits); stylometric heuristics
depend on whether the edits introduced structural variance. No signal catches this
reliably — acknowledged limitation documented here.

**4. Very long submissions (> 5000 words)**
Groq API calls have latency proportional to input length. Long submissions may hit
timeout limits or increase cost per call. Mitigation: truncate input to first 1500 tokens
for the LLM signal (semantic patterns are detectable in a window); stylometric heuristics
run on the full text cheaply.

---

## False Positive Scenario

A poet who writes in a clean, formal style submits their work.
- LLM score: 0.72 (formal prose reads as AI-like)
- Stylo score: 0.58 (uniform sentence lengths)
- Combined confidence: 0.6×0.72 + 0.4×0.58 = **0.664 → Uncertain**

The system returns the *Uncertain* label, not the AI label. The creator can appeal
with their reasoning. This is the system working correctly: genuine ambiguity produces
a genuinely uncertain label, not a false accusation.

---

## API Surface

| Endpoint          | Method | Description                              |
|-------------------|--------|------------------------------------------|
| `/submit`         | POST   | Submit content for attribution analysis  |
| `/appeal`         | POST   | Contest a classification result          |
| `/log`            | GET    | View recent audit log entries            |
| `/status/:id`     | GET    | Get current status of a submission       |

### POST /submit
**Input:** `{ "content": "...", "creator_id": "..." }`  
**Output:**
```json
{
  "id": "uuid",
  "attribution": "human | uncertain | ai_generated",
  "confidence": 0.42,
  "label_text": "Our system is uncertain...",
  "signals": {
    "llm_score": 0.51,
    "stylo_score": 0.28
  }
}
```

### POST /appeal
**Input:** `{ "content_id": "uuid", "reason": "...", "creator_id": "..." }`  
**Output:** `{ "appeal_id": "uuid", "status": "under_review", "message": "Your appeal has been logged." }`

### GET /log
**Output:** Array of audit log entries (most recent first), each including signals, appeal if any.

### GET /status/:id
**Output:** Full submission record including current status and any appeal.

---

## Rate Limiting

Applied to `POST /submit` only.

- **10 requests per minute per IP** — a legitimate creator submitting work won't hit
  this; it stops burst flooding.
- **50 requests per day per IP** — prevents sustained automated abuse while allowing
  a prolific creator to submit throughout the day.

Read endpoints (`/log`, `/status`) are not rate-limited — they're cheap and needed
for appeals workflows.

---

## Architecture

### Submission Flow

```
Client
  │
  │  POST /submit { content, creator_id }
  ▼
Rate Limiter ──── 429 if over limit
  │
  │  raw text
  ▼
Detection Pipeline
  ├── Signal 1: LLM Classifier (Groq)  ──► llm_score (0–1)
  └── Signal 2: Stylometric Heuristics ──► stylo_score (0–1)
  │
  │  llm_score, stylo_score
  ▼
Confidence Scorer
  │  confidence = 0.6*llm + 0.4*stylo
  ▼
Label Generator
  │  attribution, label_text
  ▼
Audit Log (SQLite)  ◄── writes full decision record
  │
  ▼
Response { id, attribution, confidence, label_text, signals }
```

### Appeal Flow

```
Client
  │
  │  POST /appeal { content_id, reason, creator_id }
  ▼
Appeal Handler
  │  look up original decision
  ▼
Audit Log (SQLite)  ◄── appends appeal, sets status = "under_review"
  │
  ▼
Response { appeal_id, status: "under_review", message }
```

---

## AI Tool Plan

### M3 — Submission Endpoint + Signal 1 (LLM Classifier)

**Spec sections to provide:** Detection Signals (Signal 1 only) + Architecture diagram
(submission flow) + API Surface (POST /submit shape).

**What to ask for:**
- Flask app skeleton with `POST /submit` endpoint
- Rate limiter wired up (Flask-Limiter, 10/min + 50/day on `/submit`)
- `classify_with_llm(text)` function that calls Groq and returns `llm_score` (0–1 float)
- SQLite audit log initialization and a `log_decision()` helper
- `/submit` returns `{ id, attribution, confidence, label_text, signals }` (confidence
  and label can be stubs for now — just the LLM score wired through)

**How to verify:** Call `POST /submit` with 3 test inputs — one clearly AI-generated
paragraph, one personal anecdote, one ambiguous formal essay. Confirm `llm_score` varies
meaningfully across them before wiring into the full endpoint.

---

### M4 — Signal 2 + Confidence Scoring

**Spec sections to provide:** Detection Signals (Signal 2) + Confidence Scoring &
Thresholds + Architecture diagram.

**What to ask for:**
- `compute_stylometric_score(text)` function implementing sentence-length variance,
  type-token ratio, and punctuation density, normalized and averaged into `stylo_score`
- Confidence scorer: `confidence = 0.6 * llm_score + 0.4 * stylo_score`
- Wire both signals into `/submit` so response includes `signals.llm_score`,
  `signals.stylo_score`, and final `confidence`
- Short-text warning: if word count < 80, add `"short_text_warning": true` to response

**How to verify:** Submit the same 3 test inputs from M3. Check that `stylo_score` is
independently meaningful (doesn't just mirror `llm_score`). Confirm combined `confidence`
values spread across the 0–1 range, not clustered at one end.

---

### M5 — Production Layer (Labels + Appeals + Audit Log)

**Spec sections to provide:** Label Variants (exact text) + Appeals Workflow + Confidence
Scoring thresholds + Architecture (appeal flow diagram).

**What to ask for:**
- `generate_label(confidence)` function mapping score ranges to attribution string +
  exact label text
- `POST /appeal` endpoint: validates `content_id` exists, appends appeal to audit log,
  sets status to `"under_review"`
- `GET /log` endpoint returning last N audit entries (default 20), including any appeals
- `GET /status/<id>` endpoint
- Ensure all three label variants are reachable at their thresholds

**How to verify:**
- Submit content that hits each of the three confidence ranges → confirm correct label
  text is returned verbatim
- Submit an appeal for a classified item → confirm status changes to `"under_review"` in
  `GET /status/<id>`
- Check `GET /log` shows at least 3 entries with full fields (signals, confidence, appeal
  if present)

---

## Stretch Features Planned

- [ ] **Ensemble detection** — add a third signal (e.g., perplexity estimate) with
  documented weighting
- [ ] **Analytics dashboard** — simple view of detection patterns and appeal rates
