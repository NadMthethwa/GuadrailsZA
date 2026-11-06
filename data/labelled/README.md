# Labelled data

| File | What it is | Used for training/eval? |
|---|---|---|
| `messages.jsonl` | Real messages, verbatim, source URL checked by hand | Yes |
| `needs_review.jsonl` | Collected but not yet verified (see `review_note`) | **No**, not until a human moves it into `messages.jsonl` |
| `synthetic.jsonl` | Synthetic examples (not created yet) | Reported separately |

Extra fields beyond the spec schema: `confidence` (from the collector), `source_checked`
(date the source URL was confirmed to show the text) and `review_note`. The loader should
ignore unknown fields.

## Placeholders
`[tracking code]`, `(4 digit number)` → `<ACCOUNT>`; `[Personal details]` → `<NAME>`;
`(phone number)` → `<PHONE>`. Typos and casing are kept verbatim (e.g. "gas arrived",
"cutt.Oly", which the SAPO release says is "not a real link").

## Batch log
- 2026-11-06: 8 rows from a Grok search. 5 verified against their sources, 3 held in
  `needs_review.jsonl`.
