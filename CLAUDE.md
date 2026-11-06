# GuardRail SA — project brief for Claude Code

## What this is
GuardRail SA is an API-first fraud detection microservice for South Africa. It reads the
text of incoming SMS / WhatsApp / webhook messages and flags social-engineering scams in
real time, before a victim acts on them.

It is the **content layer**. It complements, not competes with:
- MTN Chenosis "Scam Signal" (network signals: SIM swap, active-call detection)
- bank in-house models (Absa Nia, Capitec, TymeBank behavioural monitoring)
- transaction monitoring (e.g. Orca Fraud)

Distribution target: Chenosis API marketplace listing, paid pilots with mid-tier banks,
insurers, logistics firms, and SMS aggregators (outbound screening).

## Phase 1 scope (build this, nothing more)
Full spec: `docs/PHASE1_SPEC.md`. Summary:
1. **Rule engine (primary)** — SA-specific regex/keyword patterns per typology.
2. **Brand-impersonation check** — compare the brand a message *claims* to be from with
   the actual sender and the domains of any links, using `data/brands.yaml`.
3. **ML fallback** — TF-IDF + LogisticRegression, consulted only when the rule score is
   in the ambiguous band.
4. **FastAPI `POST /analyze`** returning a verdict, score, typology and human-readable
   signals.
5. **Evaluation script** with per-typology precision/recall and false-positive rate on
   hard negatives.

Typologies: `sars_refund`, `capitec_cashsend`, `fnb_ewallet`, `post_office_delivery`
(+ `bec` reserved, not built in Phase 1).
USSD channel: return **HTTP 501** in Phase 1.

## Stack and conventions
- Python 3.11+, FastAPI, Pydantic v2, scikit-learn, PyYAML, pytest. Use `uv` if available,
  otherwise `pip` + `requirements.txt`.
- Package layout: `guardrail/` (engine + api), `tests/`, `scripts/`, `data/`, `docs/`.
- Type hints everywhere. Small pure functions in the engine; the API layer only wires.
- Every flag must carry an **explanation string** a fraud analyst can read
  (e.g. "Claims to be Capitec but link goes to capitec-reversal.co, not capitecbank.co.za").
- Deterministic: the same input gives the same output. Fixed random seeds.
- Ask before adding any dependency not listed above.

## POPIA / privacy rules (non-negotiable)
- Do **not** persist or log message text by default. Log a SHA-256 hash, length, channel,
  verdict and latency only. A `GUARDRAIL_DEBUG_LOG_TEXT` env flag may enable text logging
  for local development only, defaulting to off.
- No phone numbers in logs; hash them too.
- Training data in `data/labelled/` is anonymised: replace real names, numbers and
  account details with placeholders like `<NAME>`, `<PHONE>`, `<AMOUNT>`.

## Data honesty
- The dataset is small (~31 scams, ~16 hard negatives to start). Report metrics with
  stratified cross-validation and say plainly that they are small-sample.
- Never invent "real" scam examples and label them as real. Synthetic examples go in a
  separate file (`data/labelled/synthetic.jsonl`) and are reported separately.
- Fields in `data/brands.yaml` marked `verified: false` must not be treated as ground
  truth in tests until a human confirms them.

## Working style
- Work milestone by milestone (see `KICKOFF_PROMPT.md`). Stop after each milestone,
  summarise what changed, show test results, and wait for a go-ahead.
- Run `pytest` before declaring a milestone done.
