# HL Package-Expiry Reminder Worklist

A small staff web app for working package-expiry (and low-credit) reminders as a
3-stage conveyor: **Precheck → Ready to send → Sent**. Replaces the Excel tracker —
the database surfaces "what's due / what's at my station" live, which Excel couldn't.

## Architecture

```
new Hapana export (inbox/) ─▶ Python (local) ─▶ Supabase Postgres
        push_raw.py  ───upsert (invoice_no,status), dedup in cloud──▶ reminder_raw_transactions  (raw audit trail)
        compute_*    ───clean / map / net refunds / usage+credits──▶ (enriched CSV, local)
        push_to_supabase.py ──upsert by invoice_no───────────────────▶ reminder_packs   (ingest-owned facts)
                                                                       reminder_state    (staff edits — ingest NEVER touches)
                                                                              │
                                                       reminder_worklist (view: live days_left, ALL active packs)
                                                                              │
                                          index.html (this app, supabase-js + password) ─▶ staff
```
Worklist = **all active packs** (precheck vets everything); `days_left` + `reason`
(Expiry / Low-credit / Both / Upcoming) tell staff *when* to act.

- **Supabase project:** `sbh-attendence` (`geheirnfbhqnhrjmrrax`). *(Deliberately NOT the Stripe "Hapana Add on" project — that stays untouched.)*
- **Auth:** Supabase email+password. A single shared `sbhadmin` account (no email infra). Only authenticated users can read/write (RLS). The publishable key in `index.html` is safe to expose. Staff type their initials (top-right field, remembered per device) so `checked_by`/`sent_by` still record who did the work despite the shared login.
- **Hosting:** GitHub Pages (static single file).

## Ingest / refresh (every ~2 weeks)

The pipeline is self-contained under [`pipeline/`](pipeline/). Drop the fresh Hapana
transactions export + check-in export into `pipeline/data/inbox/`, then run **from the
`pipeline/` directory** in order:

```
cd pipeline
python3 push_raw.py           # raw transactions -> cloud (dedup by invoice_no+status), pulls deduped set back
python3 ingest_ledger.py      # check-ins -> local ledger (usage/phone proxy)
python3 compute_expiry.py     # map desc->pack, filter, net refunds, compute expiry, join phone
python3 compute_usage.py      # check-in usage proxy + credits-left + low-credit flag
python3 push_to_supabase.py   # UPSERT reminder_packs
```

**Paths & creds** ([`pipeline/_paths.py`](pipeline/_paths.py)): everything anchors to
the `pipeline/` folder. Data + outputs live in `pipeline/data/` (gitignored — it holds
customer PII). Credentials are found, in order: `pipeline/data/.env` →
`$HL_REMINDER_ENV` → the legacy `attendance-bot/.env`. The service key is **never**
committed. A guardrail (`assert_reminder_project`) refuses to run against any project
other than the reminder DB (`geheirnfbhqnhrjmrrax`), so the Stripe project can't be hit
by accident. Override data/inbox locations with `HL_REMINDER_DATA` / `HL_REMINDER_INBOX`.

Overlapping uploads are safe: raw dedup is enforced in the cloud by the
`(invoice_no, payment_status)` primary key, and `reminder_packs` by `invoice_no`.
Only `reminder_packs` is upserted; staff edits in `reminder_state` are never overwritten.

> **Pipeline data is PII and stays out of git.** `pipeline/data/` and all `*.csv`
> (except the mapping config) are gitignored. This repo is public — code only, no data.

## Deploy (one-time)

1. **Push to GitHub** and enable Pages:
   ```
   gh repo create hl-package-reminders --private --source=. --remote=origin --push
   gh api -X POST repos/:owner/hl-package-reminders/pages -f source.branch=main -f source.path=/
   ```
   Pages URL will be `https://<you>.github.io/hl-package-reminders/`.
   Live URL: **https://testing20251234.github.io/hl-package-reminders/**
   Live URL: **https://testing20251234.github.io/hl-package-reminders/**
2. **Create the shared login** — Supabase dashboard → Authentication → Users → "Add user":
   - Email: `sbhadmin@houselongevity.com`  ·  set a password  ·  tick **Auto Confirm User**.
   - That's the only account; share its password with staff. (Email+password = no email
     infra, no rate limits, no spam — most reliable.)
3. Share the URL with staff → they enter `sbhadmin` email + password → in. Each types their
   initials (top-right) so `checked_by`/`sent_by` records who acted.

## Security notes
- `anon`/unauthenticated users get **nothing** (RLS denies by default). Only `authenticated` staff can read/write — the right boundary for customer PII.
- The `authenticated` policies are `USING(true)` on purpose: any signed-in staff member works the whole shared worklist. `checked_by` / `sent_by` auto-fill from the logged-in identity.
- No service-role key is in this repo or the page; ingest uses the local `.env` only.

## Low-credit caveat
Credit counts are **estimated** from check-in counts (Hapana has no per-pack balance export),
so low-credit messages use soft language ("down to your last session or two") — never a hard number.
