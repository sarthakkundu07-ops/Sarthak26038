# RetinaScreen: offline-first, explainable DR screening (SIH reference build)

Start with `docs/ARCHITECTURE.md` (diagrams, sync sequence, conflict rules, privacy model, referral table).

## File map

| Path | What it is |
|---|---|
| `docs/ARCHITECTURE.md` | System architecture, offline sync sequence, conflict resolution, security, roadmap |
| `db/schema.sql` | PostgreSQL schema: users, patients (encrypted PII), consent ledger, screenings, referrals, clinical audits, reminders, audit triggers, row-level security |
| `backend/app/quality.py` | OpenCV quality gate: blur, darkness, glare, contrast, field-of-view |
| `backend/app/xai.py` | EfficientNet-B0 classifier, **Grad-CAM on the last conv block**, heatmap blending, region boxes |
| `backend/app/risk.py` | 5-stage to referral-tier mapping with escalators and multilingual disclaimer |
| `backend/app/crypto.py` | AES-256-GCM field encryption and HMAC blind indexes |
| `backend/app/sync.py` | `/api/screenings/sync` batch endpoint and the pure field-level merge function |
| `backend/app/main.py` | `/assess-quality`, `/infer`, image upload, review queue, audit, PDF report |
| `frontend/lib/db.js`, `sync.js` | Dexie schema, transactional outbox, sync engine with backoff |
| `frontend/public/sw.js` | Service worker: app shell, offline navigation, Background Sync |
| `frontend/components/ScreeningDashboard.jsx` | Field-worker result screen |
| `frontend/components/DoctorReview.jsx` | Ophthalmologist audit workspace |
| `frontend/locales/{en,hi,bn}.json` | UI strings (Tamil: copy `en.json`) |

## Run the backend

```bash
cd backend && pip install -r requirements.txt
createdb retina && psql retina -f ../db/schema.sql
export DATABASE_URL=postgresql+asyncpg://user:pass@localhost/retina
export JWT_SECRET=$(openssl rand -hex 32)
export PII_ENC_KEY=$(openssl rand -hex 32)      # 64 hex chars
export PII_INDEX_KEY=$(openssl rand -hex 32)
export MODEL_CHECKPOINT=./weights/effb0_dr.pt    # {"state_dict":..., "temperature": 1.3}
uvicorn app.main:app --reload                    # OpenAPI docs at /docs
```

Without `MODEL_CHECKPOINT` the model has random weights, so outputs are meaningless. That mode exists only so the UI and plumbing can be demoed.

## Frontend wiring (Next.js App Router)

1. Copy `frontend/` into your Next app, run `npm i`, and enable Tailwind.
2. Import `lib/i18n.js` once in the root layout, and call `startSyncTriggers()` in a client effect.
3. Register the service worker: `navigator.serviceWorker.register("/sw.js")`.
4. Self-host Noto Sans subsets (Latin, Devanagari, Bengali, Tamil) in `public/fonts` so scripts render offline.
5. Render `<ScreeningDashboard result={...} />` after `/infer` (or from the cached `db.results` row).

## What was tested here

- `quality.py` on synthetic fundus images: sharp image passes; blurred, dark, side-cropped and empty images are rejected with the right reason codes.
- `risk.py` tier mapping and escalation cases.
- The field-level merge function: client-newer wins, client-older loses (with a `needs_confirm` conflict for safety fields), and non-overlapping fields merge cleanly.
- All Python and JS/JSX files parse without syntax errors.

**Not** run here: PyTorch inference and Grad-CAM (torch was unavailable in the build sandbox), the database, and the browser UI. Expect to fix small integration issues when you first run them, and treat the SQL, sync and UI code as reviewed but not integration-tested.

## Before any real-world use

- Train and externally validate the model on Indian fundus data (APTOS 2019 and EyePACS are common starting points). Report sensitivity and specificity at the referable-DR threshold and quadratic weighted kappa, and fit the temperature on a held-out set.
- Have an ophthalmologist approve the referral thresholds and escalators in `risk.py`.
- Have native medical translators review the Hindi and Bengali strings (they are first drafts).
- Regulatory and privacy review: an AI tool that informs referrals may count as Software as a Medical Device under CDSCO rules, and DPDP Act 2023 obligations (notice, consent, erasure, breach reporting) apply. A KMS-backed key setup replaces the env-var keys used here.
- Known gaps: login and refresh-token endpoints, user and facility admin, the reminder worker that sends SMS and WhatsApp (the `reminders` table and scheduling are in place), Indic fonts in the PDF report, rate limiting, and IndexedDB blob encryption with the worker PIN.
