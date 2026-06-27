# Provenance Guard — planning.md

---

Provenance Guard is a backend system that a creative sharing platform could use to classify submitted content, score confidence in those classifications, display a transparency label to users, and manage appeals from creators who believe they've been misclassified.

---

# Architecture Narrative

Text is accepted through a rate-limited `POST /submit` endpoint. The endpoint passes the text through a two-signal detection pipeline, an LLM-based classifier (via Groq) and a set of stylometric heuristics computed locally. The two signal scores are combined into a single confidence value, which drives both the attribution result (`human`, `ai`, or `uncertain`) and the plain-language transparency label returned in the response. Every decision (along with the raw signal values, confidence score, and label) is written to a structured audit log. Creators who dispute a classification can submit a `POST /appeal`; the appeal captures their reasoning, is appended to the audit log entry, and flips the content's status to `under_review`.

---

## Detection Signals
<!-- For each signal, write: what property of the text it measures, why that property differs between human and AI writing, and what it can't capture (every signal has blind spots). If you can't describe the blind spot, you don't understand the signal yet. -->

### Signal 1 — LLM-based classification (Groq)

Asks the model to assess whether text reads as human or AI-generated based on holistic semantic and stylistic coherence. AI-generated text tends toward consistent register, smooth transitions, and topic sentences that closely mirror the prompt — patterns a language model can recognize because it produces them.

**Blind spot:** Easily fooled by prompting a model to "write like a person" or by heavy human editing of AI output. Also unreliable on texts under ~100 words where there is not enough signal for the classifier to form a confident judgment. Performance degrades on niche genres (e.g., experimental poetry) where the model has little training signal.

---

### Signal 2 — Stylometric heuristics

Measures surface statistical features of vocabulary and sentence structure that differ in distribution between human and AI corpora.

| Feature | Human (reference) | AI (reference) | Direction |
|---------|-------------------|----------------|-----------|
| Type-Token Ratio (TTR) | ~0.70 | ~0.60 | lower = more AI |
| Sentence Length CV (SD/mean) | ~0.50 | ~0.25 | lower = more AI |
| Average Word Length (chars) | ~4.5 | ~5.5 | higher = more AI |

- **Type-Token Ratio (TTR):** measures lexical diversity — unique words divided by total words. AI models recycle a narrower set of high-probability tokens, producing lower TTR. **Blind spot:** Short texts have artificially high TTR regardless of authorship; this feature only becomes reliable beyond ~300 words.
- **Sentence Length CV (coefficient of variation = SD / mean):** captures how uniform sentence lengths are. AI output tends toward consistent, moderate sentence lengths; human writing has more variance (short punchy sentences mixed with longer ones). **Blind spot:** Listicles and instructional human writing have deliberately uniform sentence lengths, mimicking low CV.
- **Average Word Length:** formal AI vocabulary (transformative, implications, stakeholders) averages longer than casual human writing. **Blind spot:** Academic human writing and legal text also skew toward long words, causing false positives on formal human content.

---

### False Positive Problems/Anticipated Edge Cases

Non-native English writers are the highest-risk group for false positives. Their prose can share surface features with AI output, as they may have more uniform sentence length, smaller active vocabulary, and heavier reliance on common connective phrases. A system that flags "unusual regularity" will disproportionately misclassify their work as AI-generated.

Formal or heavily edited human writing (academic papers, grant proposals) also clusters near AI distributions on most of these features, since the editing process smooths out the variance that distinguishes casual human writing.

---

## Uncertainty Representation

Both signals return a float in [0.0, 1.0], where 0.0 means human and 1.0 means AI. The combined confidence is a weighted average:

```
confidence = 0.6 × llm_score + 0.4 × heuristic_score
```

The LLM classifier gets the higher weight (0.6) because it captures holistic semantic patterns — two texts can look identical on stylometric features while being clearly distinguishable to a language model. Heuristics are cheap and provide a structural cross-check, but they are noisier, so they receive less weight.

A score of 0.60 means the system leans toward AI-generated but without strong conviction: either both signals point slightly toward AI, or one points strongly while the other is near 0.5. The uncertain zone (0.35–0.65) is intentionally wide — roughly 30 points on each side of 0.5 — because forcing a label in the middle of the distribution is worse than saying "we don't know."

The thresholds are asymmetric by design. Labeling a human's work as AI-generated (false positive) is more harmful to creators than failing to detect AI content (false negative). So the human threshold is placed at 0.35 rather than 0.50 — the system only asserts "human" when the AI probability is clearly low.

| Confidence range | Result | Rationale |
|---|---|---|
| ≤ 0.35 | human | AI probability clearly low |
| 0.35 – 0.65 | uncertain | Insufficient signal to decide |
| > 0.65 | ai | AI probability clearly high |

---

## Transparency Labels

Three label variants. The confidence percentage is rounded to the nearest whole number.

**High-confidence human** (confidence ≤ 0.35):
> "This content appears to have been written by a person. Our detection system is [X]% confident in this result."

**Uncertain** (0.35 < confidence ≤ 0.65):
> "Our detection system could not confidently determine whether this content was written by a person or generated by AI (confidence: [X]%). If you believe this classification is incorrect, the creator can submit an appeal."

**High-confidence AI** (confidence > 0.65):
> "This content appears to have been generated by an AI. Our detection system is [X]% confident in this result. If this is incorrect, the creator can submit an appeal."

Design rationale: the AI label avoids punitive framing — AI assistance is not prohibited on this platform; the label exists for transparency. The human label omits the appeal link because a human classification is the favorable outcome and rarely contested. The uncertain label always includes the appeal link because uncertainty is the case where a creator is most motivated to challenge the result.

---

## API Surface Sketch

### `POST /submit`
Submit text for attribution analysis.

**Request body**
```json
{
  "content": "string (the text to classify)",
  "creator_id": "string (opaque identifier for the submitting user)"
}
```

**Response**
```json
{
  "content_id": "uuid",
  "result": "human | ai | uncertain",
  "confidence": 0.0,
  "label": "string (plain-language transparency label)",
  "signals": {
    "llm_score": 0.0,
    "llm_reasoning": "string",
    "heuristic_score": 0.0,
    "ttr": 0.0,
    "len_cv": 0.0,
    "avg_word_length": 0.0,
    "ttr_score": 0.0,
    "len_cv_score": 0.0,
    "word_len_score": 0.0
  },
  "status": "classified"
}
```

**Rate limit:** 10 requests / minute per IP (see reasoning in Required Features).

---

### `GET /log`
Return the full audit log as a JSON array, newest first.

**Response**
```json
[
  {
    "id": "uuid",
    "timestamp": "ISO 8601",
    "creator_id": "string",
    "result": "human | ai | uncertain",
    "confidence": 0.0,
    "signals": { ... },
    "label": "string",
    "status": "classified | under_review",
    "appeal": null
  }
]
```

---

### `POST /appeal`
Submit a creator appeal against a classification.

**Request body**
```json
{
  "content_id": "uuid",
  "creator_reasoning": "string"
}
```

**Response**
```json
{
  "appeal_id": "uuid",
  "content_id": "uuid",
  "status": "under_review"
}
```

The appeal is appended to the matching audit log entry and flips `status` to `under_review`.

---

## Appeals Workflow
<!-- Who can submit an appeal? What information do they provide? What does the system do when an appeal is received — what status changes, what gets logged? What would a human reviewer see when they open the appeal queue? -->

**Who can submit:** Any creator whose submission has already been classified (status = `classified`). An appeal against an unknown or already-under-review `submission_id` returns a 404 or 409 respectively.

**What they provide:** The `submission_id` of the decision they are contesting and a `creator_reasoning` string explaining why they believe the classification is wrong. No minimum length is enforced, but empty reasoning is rejected with a 400.

**What the system does when an appeal is received:**
1. Looks up the audit log entry for `submission_id`; returns 404 if not found.
2. Generates a new `appeal_id` (UUID) and records the appeal timestamp.
3. Appends an appeal object to the log entry: `{ "appeal_id", "timestamp", "creator_reasoning" }`.
4. Sets the entry's `status` from `classified` → `under_review`.
5. Returns `{ appeal_id, submission_id, status: "under_review" }` to the caller.

No automated re-classification is triggered. The original `result`, `confidence`, and `signals` values are preserved unchanged.

**What a human reviewer sees:** Calling `GET /log` returns each entry with both the original decision (result, confidence, raw signal values, label) and the appeal object side by side. That is enough context to evaluate the appeal and make a manual decision. The reviewer's resolution step is out of scope for this system — the audit log records the appeal; a human acts on it externally.

---

## Architecture

```
Client
  │
  ▼
POST /submit
  │
  ├─ Rate Limiter (flask-limiter, 10 req/min per IP)
  │
  ▼
Detection Pipeline
  ├─ Signal 1: LLM Classifier (Groq API)
  │     → llm_score  [0.0 = human, 1.0 = AI]
  │
  └─ Signal 2: Stylometric Heuristics (local)
        → heuristic_score  [0.0 = human, 1.0 = AI]
            computed from TTR, mean sentence length, "of" frequency

  │
  ▼
Confidence Aggregator
  weighted average: 0.6 × llm_score + 0.4 × heuristic_score
  → confidence [0.0 – 1.0]

  │
  ▼
Label Generator
  confidence ≤ 0.35  -> result = "human",     high-confidence human label
  confidence > 0.65  -> result = "ai",         high-confidence AI label
  otherwise          -> result = "uncertain",  uncertain label

  │                             │
  ▼                             ▼
Response to Client         Audit Log entry written (in-memory dict / JSON file)

Appeals
  POST /appeal → appends appeal to audit log entry, sets status = "under_review"
  GET  /log    → returns full audit log
```

---

## AI Tool Plan

### M3 — Submission endpoint + first signal (LLM classifier)

**Sections to provide:** Detection Signals (LLM section only) + API Surface Sketch (`POST /submit`) + Architecture diagram.

**What to ask for:** Flask app skeleton with a single `POST /submit` endpoint, plus a `classify_with_llm(text)` function that calls the Groq API and returns a score between 0 and 1. The prompt to Groq should ask for a structured JSON response containing `{"score": float, "reasoning": string}`.

**How to verify:** Send 3 test inputs manually (clearly human text, clearly AI text, ambiguous text) and check that (1) the endpoint returns HTTP 200 with the required fields, (2) the LLM score is directionally correct for the obvious cases, and (3) the audit log entry is written.

---

### M4 — Second signal + confidence scoring

**Sections to provide:** Detection Signals (Stylometric Heuristics section) + Confidence Aggregator block from the Architecture diagram.

**What to ask for:** A `compute_heuristic_score(text)` function that calculates TTR, mean sentence length, and "of" frequency, normalizes each against the human/AI reference distributions in the table, and returns a combined heuristic score. Then ask for the confidence aggregator that merges `llm_score` and `heuristic_score` using the 0.6 / 0.4 weights and maps the result to `result` + `label`.

**What to check:** Run the same 3 test inputs from M3. Confirm that (1) scores vary meaningfully between the obvious human and obvious AI samples, (2) the uncertain sample lands in the 0.35–0.65 band, and (3) swapping signal weights visibly changes the confidence value.

---

### M5 — Production layer (labels, appeals, rate limiting)

**Sections to provide:** Label variants description + Appeals Workflow + `POST /appeal` API sketch + Architecture diagram.

**What to ask for:** Label generation logic that produces the three exact label strings for each variant (high-confidence AI, high-confidence human, uncertain), the `POST /appeal` endpoint, and flask-limiter configuration for `POST /submit`.

**How to verify:** (1) Trigger each of the three label variants and confirm the exact text matches the spec. (2) Submit an appeal for an existing submission and confirm the audit log entry shows `status: under_review` and the appeal text. (3) Hit `POST /submit` 11 times in a minute and confirm the 11th returns HTTP 429.

---

## Required Features

**Content Submission Endpoint:** Build an API endpoint that accepts a piece of text-based content (a poem, a short story excerpt, a blog post) for attribution analysis. The endpoint must return a structured response including the attribution result, confidence score, and the transparency label text that would be shown to the user.

**Multi-Signal Detection Pipeline:** Your detection pipeline must use at least 2 distinct signals to classify content. Single-signal detection is not acceptable. Your planning.md and README must explain what each signal captures and why you chose them.

**Confidence Scoring with Uncertainty:** Your system must return a confidence score, not just a binary label. The score should reflect genuine uncertainty — a 0.51 confidence should produce a meaningfully different transparency label than a 0.95. Your README must explain how you approached this and how you tested whether your scores are meaningful.

**Transparency Label:** Design and implement the label that would be displayed to a reader on the platform. It must communicate the attribution result in plain language and make the confidence level meaningful to a non-technical reader. Include a typed description of all three label variants (high-confidence AI, high-confidence human, uncertain) in your README — write out the exact text each one displays. You're welcome to include a screenshot or mockup as well, but the written description is what's required.

**Appeals Workflow:** Implement a mechanism for creators to contest a classification. At minimum, an appeal must: capture the creator's reasoning, log the appeal alongside the original decision, and update the content's status to "under review." Automated re-classification is not required.

**Rate Limiting:** Implement rate limiting on your submission endpoint. Your README must document the limits you chose and your reasoning for those specific values.

**Audit Log:** Every attribution decision — including confidence score, signals used, and any appeals — must be captured in a structured audit log. Document the log in your README (or via the GET /log output) with at least 3 entries visible.
