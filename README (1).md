# VeriCare

**Find help. Find medicine. Trust what’s verified.**

VeriCare puts emergency help and medicine search in one web app. Its main idea: **never claim something is true without proof.** It will not say a medicine is in stock, a hospital can help, or a price is current unless there is real, recent evidence — and it always shows where each fact came from.

- **Frontend:** React + Vite, plain modern CSS, Lucide icons, Leaflet + OpenStreetMap map
- **Backend:** Python + FastAPI, SQLAlchemy + SQLite (ready for PostgreSQL), Pydantic, httpx
- **Live web lookup:** TinyFish web-automation API (backend only)

---

## 1. Quick start

You need **Python 3.11+** and **Node.js 18+**. (Built and tested with Python 3.13 and Node 22.)

### Backend (terminal 1)

```bash
cd vericare/backend
python -m venv .venv
# Windows:      .venv\Scripts\activate
# macOS/Linux:  source .venv/bin/activate
pip install -r requirements.txt
python -m uvicorn main:app --reload --port 8000
```

Check it works: open http://localhost:8000/api/health — you should see `"status": "ok"`.
API docs (auto-generated): http://localhost:8000/docs

Demo data is created automatically every time the backend starts (timestamps are reset so the demo always shows confirmed, expired and unconfirmed examples).

### Frontend (terminal 2)

```bash
cd vericare/frontend
npm install
npm run dev
```

Open **http://localhost:5173**. The frontend calls `/api/...`, and Vite forwards those calls to the backend on port 8000.

---

## 2. Where to put your TinyFish API key

1. Copy `vericare/.env.example` to **`vericare/backend/.env`**.
2. Open `backend/.env` and fill in:
   ```
   TINYFISH_API_KEY=your-key-here
   PHARMACY_SITE_URL=https://the-pharmacy-site-you-are-allowed-to-search.example
   ```
3. Restart the backend. The Find Medicine page will now show a **Run live lookup** button.

Rules that keep the key safe:
- The key is read **only by the backend** (`backend/config.py`) and only sent in the `X-API-Key` header to TinyFish.
- The browser never sees it. `/api/config` and `/api/health` only say “configured / not configured” (there is a test for this).
- Logs never print it.
- `.env` is in `.gitignore`. **Never commit it, never paste it in frontend code.**

**Choosing a pharmacy site:** pick a site whose terms allow automated reading and whose `robots.txt` allows it. VeriCare checks `robots.txt` before every run and refuses if it is disallowed. It never logs in, solves CAPTCHAs or goes past paywalls. You can list several sites, separated by commas — users choose from that list only.

---

## 3. How the trust rules work

| Rule | Where in code |
|---|---|
| Products are “the same” only if **active ingredient + strength + dosage form** all match. Missing info = “needs review”. Similar names never count. | `services/medicine_service.py` → `compare_composition` |
| Price per unit only when both price and pack size are known. | `unit_price` |
| Prices older than **7 days** are flagged “may be outdated” (`PRICE_FRESHNESS_DAYS`). | `price_freshness` |
| “Availability recently confirmed” needs a confirmation from a **trusted source** (pharmacy staff, verified call, pharmacy system), with both timestamps, **less than 2 hours old** (`CONFIRMATION_EXPIRY_MINUTES`). | `services/verification_service.py` |
| A TinyFish page visit or “In stock” text on a website is **never** a confirmation. It is shown as “Website text: …”, status stays *unconfirmed*. | `tinyfish_service.py`, `LiveProductCard` |
| The **Verify stock** button opens the pharmacy’s phone option. It does not change any status. | `Dialogs.jsx` |
| **Urgent Mode**: confirmed listings first, sorted by distance; everything else in a separate “call first” section. Price is ignored, so a cheap unconfirmed listing can never rank as available. | `rank_urgent` |
| Emergency results always say “Availability not verified”. VeriCare never claims beds, ambulances or doctors are available. | `services/emergency_service.py` |
| All seeded records are **fictional** and labelled *Demo* (names end in “(Demo)”, phone numbers are `+91-00000-…`). | `seed.py` |

Stock statuses: **Stock unconfirmed · Availability recently confirmed · Confirmation expired · Unable to verify**.

### Recording a real confirmation (pharmacy staff)

Set `CONFIRMER_TOKEN=some-long-secret` in `backend/.env`, then:

```bash
curl -X POST http://localhost:8000/api/confirmations \
  -H "Content-Type: application/json" -H "X-Confirmer-Token: some-long-secret" \
  -d '{"listing_id": 1, "status": "in_stock", "confirmation_source": "pharmacy_staff", "confirmed_by": "Asha, counter staff"}'
```

Without the token the endpoint returns 401 (or 403 if no token is set on the server). The expiry time is always set by the server.

---

## 4. API

All responses use one shape: `{"ok": true, "data": ..., "meta": {...}}` or `{"ok": false, "error": {"code", "message"}}`.

| Method | Path | What it does |
|---|---|---|
| GET | `/api/health` | Server, database and TinyFish status |
| GET | `/api/config` | Public settings (freshness, expiry, permitted sites) — no secrets |
| GET | `/api/geocode?q=` | Area name → coordinates (built-in area list; Nominatim if `ENABLE_OSM_LIVE=true`) |
| GET | `/api/medicines/search?q=&lat=&lng=&urgent=` | Search local database; compare mode or Urgent Mode |
| GET | `/api/medicines/{id}/alternatives` | Same-composition products + “needs review” ones |
| POST | `/api/tinyfish/medicine-search` | `{"medicine_name": "...", "site_url": "optional, must be permitted"}` → live, validated listings. 503 if not configured, 504 on timeout, 502 on TinyFish errors |
| GET | `/api/pharmacies/nearby?lat=&lng=&radius_km=` | Pharmacies by distance |
| GET | `/api/emergency/nearby?area=` or `?lat=&lng=` | Nearby hospitals/clinics + emergency number |
| GET | `/api/confirmations/{id}` | One confirmation with its evaluated status |
| POST | `/api/confirmations` | Record stock evidence (needs `X-Confirmer-Token`) |

TinyFish call used (checked against docs.tinyfish.ai and the official `tinyfish` Python SDK v0.9.0): `POST https://agent.tinyfish.ai/v1/automation/run`, header `X-API-Key`, body `{url, goal, agent_config, output_schema}`; reply `{status, run_id, result, error, num_of_steps}`.

---

## 5. Tests

```bash
cd backend && python -m pytest        # 51 tests
cd frontend && npm test               # 11 tests
```

All TinyFish calls in tests are mocked (`httpx.MockTransport`), so no credits are used.

Covered: medicine search, price per unit, wrong/incomplete matches, missing confirmation, expired confirmation, untrusted/future confirmations, Urgent Mode ranking, missing TinyFish fields, TinyFish timeouts, HTTP errors (401/402/429/500), failed runs, bad JSON, invalid/off-site source URLs, site allow-list, confirmation access control, emergency results, frontend loading/error/empty states, live-lookup failure with labelled demo fallback.

---

## 6. Moving to PostgreSQL later

Set `DATABASE_URL=postgresql+psycopg://user:pass@host/vericare`, `pip install psycopg[binary]`, and add Alembic for migrations. Models use only portable types.

---

## 7. Demo script for judges (about 3 minutes)

1. **Problem (20 s).** “In an emergency people drive to a pharmacy that’s out of stock, or trust a price that’s weeks old. Most apps *guess*. VeriCare refuses to guess.”
2. **Home (20 s).** Point to the two cards, the “live availability is not guaranteed” notice, and the four verification rules. Type *Connaught Place* and press **Set**.
3. **Find Medicine — compare (40 s).** Search **paracetamol**. Show that 500 mg tablets, 650 mg tablets and syrup are in **separate groups** — “same name, not the same medicine.” Point at **₹/unit**, the **Price may be outdated** tag (12-day-old price), and the source + time on every card. Click **Same-composition options** on Paracip-Demo 500: only FeverEase-Demo 500 matches.
4. **Urgent Mode (40 s).** Flip the toggle. “Two pharmacies have a confirmation from the last 2 hours — nearest first. The cheapest listing, GreenLeaf at ₹1.60/unit, is *not* at the top, because nobody has confirmed it.” Show the expired and unable-to-verify badges.
5. **Verify stock (15 s).** Click it: “This opens the call option. Clicking it does **not** mark anything verified — only the pharmacy can.”
6. **Live lookup with TinyFish (40 s).** Click **Run live lookup**. While it runs: “TinyFish’s AI browser opens a permitted pharmacy site, searches, and reads only what’s visible. We validate every field with Pydantic, drop off-site links, and even if the site says ‘In stock’ we show it as *website text*, not a confirmation.” Show the result cards with source link and retrieved time.
7. **Emergency (25 s).** Open Emergency Help: big **Call 112** first, then nearby hospitals with distance, directions and “Availability not verified — call ahead.”
8. **Close (10 s).** “Every fact has a source, a time, and an expiry. That’s VeriCare.”

Tip: run one live lookup before you present, in case the site is slow. If live lookup fails, the app shows the reason and offers clearly labelled demo data.

---

VeriCare is a prototype. It is not a medical service, gives no dosage advice, and does not replace emergency services. In an emergency, call **112**.
