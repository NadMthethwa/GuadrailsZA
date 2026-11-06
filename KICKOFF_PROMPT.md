# Kickoff prompt — paste this as your first message in Claude Code

---

Read `CLAUDE.md` and `docs/PHASE1_SPEC.md` first. We're building GuardRail SA Phase 1.
Work through the milestones below **one at a time**. After each one: run `pytest`,
summarise what you changed and the test results, and stop until I say "go".

Before Milestone 1, tell me in 5 bullets how you understand the system, and list any
questions about the spec. Don't write code yet.

**M1 — Skeleton.** Project layout, `pyproject.toml` (or `requirements.txt`), FastAPI app
with `GET /health` and a stub `POST /analyze` using the Pydantic request/response models
from the spec. USSD returns 501. Privacy-safe logging middleware (hash, length, channel,
verdict, latency — no text). Tests for validation, 501 and the logging rule.

**M2 — Normaliser + brand check.** `engine/normalise.py` (URL extraction incl. bare
domains, homoglyph folding, amount normalisation) and `engine/brands.py` (claimed brand,
sender match, official/lookalike domains, shorteners) driven by `data/brands.yaml`.
Unit tests with clear cases, e.g. "capitec-reversal.co", "sars-gov.za.refund.top",
"fnb.co.za.secure-login.net", "capitecbank.co.za" (official), plus a message mentioning
no brand.

**M3 — Rule engine.** `engine/rules.py` with weighted, explained patterns for the four
typologies + generic urgency/OTP signals; score combination and typology selection as
specced. `/analyze` returns real results using rules only. Tests per typology, including
hard negatives (real-looking SARS reminders, FNB notifications, courier notices, OTPs).

**M4 — ML fallback + training.** `scripts/train.py` and `engine/ml.py` per spec; ambiguous
band wiring; rules-only mode when no model file exists (reported in `/health`).

**M5 — Evaluation + packaging.** `scripts/evaluate.py` (stratified CV, per-typology
metrics, FPR on hard negatives, rules-vs-ML share, small-sample warning; synthetic data
reported separately), latency check, Dockerfile, Makefile, and a README with a curl
example.

Rules: ask before adding dependencies; never write fake "real" scam data — if you need
test fixtures, write them inline in tests or in `data/labelled/synthetic.jsonl`, marked
synthetic.
