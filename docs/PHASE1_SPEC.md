# GuardRail SA — Phase 1 specification

## 1. API

### `POST /analyze`
Request:
```json
{
  "channel": "sms | whatsapp | webhook | ussd",
  "sender": "string — short code, MSISDN, alpha sender ID or WhatsApp number",
  "text": "string — message body",
  "received_at": "ISO-8601, optional",
  "metadata": {}
}
```
- `channel == "ussd"` → **501 Not Implemented** with message "USSD support is Phase 2".
- Empty text → 422.
- Max text length 4,096 chars → 413 above that.

Response:
```json
{
  "verdict": "scam | suspicious | legitimate",
  "risk_score": 0.0,
  "typology": "sars_refund | capitec_cashsend | fnb_ewallet | post_office_delivery | unknown | null",
  "claimed_brand": "capitec | null",
  "sender_matches_brand": true,
  "url_findings": [
    {"url": "...", "domain": "...", "official_for_claimed_brand": false, "lookalike_of": "capitecbank.co.za"}
  ],
  "signals": [
    {"id": "urgency_reversal", "weight": 0.25, "explanation": "Asks you to reverse a payment urgently"}
  ],
  "decided_by": "rules | ml",
  "engine_version": "0.1.0",
  "latency_ms": 0
}
```

### `GET /health`
Returns status and engine version.

## 2. Pipeline
1. **Normalise** — lowercase copy for matching, collapse whitespace, unify common SA
   abbreviations ("R 1,500.00" / "R1500"), extract URLs (including bare domains and
   shorteners), basic homoglyph folding (0→o, 1→l, Cyrillic look-alikes).
2. **Brand-impersonation check** (`engine/brands.py`)
   - Detect the claimed brand from text using aliases in `data/brands.yaml`.
   - `sender_matches_brand`: whether the sender is in that brand's official senders.
     Unknown (no official data) → `null`, not `false`.
   - For each URL: is the domain official for the claimed brand? Is it a lookalike of an
     official domain (brand token inside a non-official domain, edit distance ≤ 2,
     homoglyphs, official name as a subdomain of another domain)?
   - URL shorteners (bit.ly, tinyurl, etc.) in a brand-claiming message → signal.
3. **Rule engine** (`engine/rules.py`)
   - Patterns per typology, each with a weight and explanation. Examples:
     - SARS: refund/"tax return approved" + link or request for banking details; "eFiling"
       outside sars.gov.za.
     - Capitec CashSend / FNB eWallet: "sent you by mistake", "please reverse", "send back",
       withdrawal PIN / voucher requests, pressure timelines.
     - Post Office / courier: "parcel held", "customs fee", small payment (R15–R60) +
       link.
     - Generic: urgency ("within 24 hours", "account will be suspended"), OTP/PIN
       requests, "do not share this with anyone" paired with a request to share.
   - Combine: `rule_score = 1 - Π(1 - weight_i)` over fired signals, plus brand-impersonation
     signals. Highest-scoring typology wins.
4. **Decision**
   - `rule_score ≥ 0.65` → scam (decided_by = rules)
   - `rule_score < 0.35` → legitimate (decided_by = rules)
   - Otherwise consult the ML model; final score = 0.5·rule + 0.5·ml. Verdict thresholds:
     ≥ 0.6 scam, 0.4–0.6 suspicious, < 0.4 legitimate. Thresholds live in config.
5. **ML fallback** (`engine/ml.py`)
   - TF-IDF (word 1–2 grams + char 3–5 grams) + LogisticRegression (class_weight
     balanced). Trained by `scripts/train.py`, saved to `models/` with joblib. If no model
     file exists, the API runs rules-only and says so in `/health`.

## 3. Data
- `data/labelled/messages.jsonl` — one object per line:
  `{"id", "channel", "sender", "text", "label": "scam|legit", "typology", "source", "hard_negative": bool}`
- `data/labelled/synthetic.jsonl` — same schema, reported separately.
- `data/brands.yaml` — official identities (see file).

Hard negatives are legitimate messages that look scammy: real SARS eFiling reminders,
real Capitec/FNB payment notifications, real courier delivery notices, OTP messages.

## 4. Evaluation (`scripts/evaluate.py`)
- Stratified 5-fold CV on real data; separate report on synthetic.
- Report per typology: precision, recall, F1; overall false-positive rate on hard
  negatives; confusion matrix; share of decisions made by rules vs ML.
- Print a clear "small sample" warning with n per class.
- Phase 1 targets (on real data): recall ≥ 0.85 on scams, FPR ≤ 0.10 on hard negatives.

## 5. Non-functional
- p95 latency < 50 ms per message on a laptop, rules + ML.
- No message text in logs (see CLAUDE.md).
- Dockerfile, plus `make test`, `make run`, `make eval` (or equivalent scripts).

## 6. Out of scope (Phase 1)
USSD, business email compromise, multilingual (isiZulu/Afrikaans/Sesotho) patterns,
auth/API keys and rate limiting, a sender-ID registry, persistence/database.
