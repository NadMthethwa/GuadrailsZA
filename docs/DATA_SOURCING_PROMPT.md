# Data-sourcing prompt (for an external research assistant, e.g. Grok)

Copy everything between the lines below into the assistant. Paste its output back to
Claude Code; it will be checked before it goes into `data/labelled/`.

---

I'm building a scam-detection model for South African SMS and WhatsApp messages. I need
**real, publicly documented example messages**, not invented ones. Please search the web
and X/Twitter for the verbatim text of scam and legitimate messages in the categories below.

## Where to look
- Official scam warnings: SARS ("scam alerts" pages), SABRIC, Capitec, FNB, Absa,
  Standard Bank, Nedbank, the South African Post Office, The Courier Guy, RAM, DSV, Aramex, Pargo.
- SA news reporting that quotes the message text (News24, IOL, MyBroadband, TimesLIVE,
  BusinessTech, Daily Maverick, Moneyweb, The Citizen).
- Public posts on X/Twitter, Reddit (r/southafrica) and HelloPeter where people quote
  a scam SMS or WhatsApp they received.

## What I need (aim for these counts)
**Scams, about 15 each:**
1. `sars_refund`: "tax refund approved", "eFiling" links not on sars.gov.za, requests for banking details.
2. `capitec_cashsend`: "I sent you money by mistake, please reverse/send back", CashSend PIN or voucher requests.
3. `fnb_ewallet`: same "sent by mistake" pattern for FNB eWallet, withdrawal PIN requests.
4. `post_office_delivery`: "parcel held", "customs fee", small payment (R15 to R60) plus a link.
   Courier-company impersonation counts here as well.

**Hard negatives, about 30 total.** These are *genuine* messages that look scammy:
- real SARS eFiling reminders and notifications
- real Capitec/FNB payment, CashSend and eWallet notifications
- real courier delivery notices with tracking links
- real OTP messages from SA banks and retailers

## Rules (important)
- **Do not invent or paraphrase messages.** Only include text that appears verbatim in a source you can link.
  If you can't find enough real examples, give fewer. Don't pad.
- **One source URL per message**, and that URL must show the text.
- **Anonymise** before giving me the text: replace personal names with `<NAME>`, phone numbers with `<PHONE>`,
  account/card/reference numbers with `<ACCOUNT>`, and specific personal amounts with `<AMOUNT>`.
  **Keep** brand names, domains, URLs and small fee amounts like "R25.50", because the model needs those.
- Keep spelling mistakes, odd capitalisation and emojis exactly as in the original. Those are signals.
- For hard negatives, only use messages that a source (the brand itself or a credible report)
  says are genuine.
- Mark anything you're unsure about with `"confidence": "low"`.

## Output format
Return **only** JSON Lines (one JSON object per line, no commentary, no markdown fences):

{"id": "grok-0001", "channel": "sms", "sender": "<PHONE>", "text": "...", "label": "scam", "typology": "sars_refund", "source": "https://...", "hard_negative": false, "confidence": "high"}

Field rules:
- `channel`: `sms`, `whatsapp` or `webhook`.
- `sender`: the alpha sender ID or short code if shown (e.g. "SARS", "Capitec", "40404"),
  otherwise `<PHONE>`, or `"unknown"` if the source doesn't show it.
- `label`: `scam` or `legit`.
- `typology`: one of `sars_refund`, `capitec_cashsend`, `fnb_ewallet`, `post_office_delivery`, `unknown`.
  Legit messages use the typology they *resemble* (e.g. a real SARS reminder gets `sars_refund`).
- `hard_negative`: `true` for every legit message in the list above, otherwise `false`.
- `source`: the URL where you found the text.

After the JSON Lines, add one final line starting with `# SUMMARY:` that gives the count per
typology and label, and lists any categories where you couldn't find enough real examples.

---

## Notes for whoever pastes the output back
- Spot-check source URLs before accepting a row. Rows whose source doesn't show the text get
  dropped. They must not be relabelled as real (CLAUDE.md, data honesty).
- `confidence` is an extra field. The loader will ignore or strip it.
- Check the anonymisation by hand before committing. This is POPIA.
