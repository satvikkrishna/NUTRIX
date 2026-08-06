# NUTRIX

A nutrition intelligence app — scan or search any packaged food for an instant health score and diet/allergen check, snap a photo of a home-cooked or restaurant dish (specialized for Indian cuisine) to identify it and estimate its nutrition, log meals and track macros over time, and get AI-generated nutrition advice based on your actual eating pattern.

Built solo. Full-stack: Next.js frontend, Express API, two LLM providers with an automatic fallback between them.

---

## What it actually does

### Packaged food lookup
Search by name (typo-tolerant — "chiken nuggets" and "snikers" both resolve) or scan a barcode with the camera. Pulls the full product record live from [OpenFoodFacts](https://world.openfoodfacts.org/) — no local food database for this path, no API key needed.

- **Health Score (0–100)** — not just a pass-through of Nutri-Score. Combines four independent penalties: Nutri-Score grade, per-nutrient levels (fat/saturated-fat/sugar/salt), NOVA processing group (penalizes ultra-processed foods), and additive count. Plain almonds score ≈89; a cookie-style snack scores ≈31 — an earlier Nutri-Score-only version clustered almost everything around 90 and didn't discriminate.
- **Diet & allergen flags** — list ingredients to avoid (`peanuts,milk,gluten`); matches handle singular/plural (`peanut` also catches `peanuts`) and fall back to checking the product name/brand when ingredient text is missing.
- **"Better Choice" alternative** — when a product is flagged or scores below 60, searches the same OpenFoodFacts category (most-specific category tried first) for a genuinely higher-scoring, non-flagged alternative — not just a random same-category product.

### AI food image recognition
Photograph a home-cooked meal or restaurant dish (not a packaged product with a barcode) and NUTRIX identifies it and estimates full nutrition. The vision prompt is specifically tuned for Indian cuisine — thalis, curries, rice dishes, snacks — and asks the model to name the *specific* dish (e.g. "Dal Chawal," not "rice with sauce").

**Two-provider fallback, tried in order:**
1. **Gemini** (free tier, 1500 req/day) — primary.
2. **Ollama + llava:7b**, running locally — automatic fallback if Gemini is unavailable or fails.

Whatever the model identifies is then cross-referenced against a **curated local database of ~40+ Indian dishes** (`server/utils/indianFoodDB.js`) with hand-verified nutrition per serving. If the dish matches, NUTRIX overrides the LLM's estimated nutrition with the curated figures and marks confidence "high" — the same grounding instinct behind Meridian's citation system and Anchor's schema constraints, applied here to stop an LLM from freely estimating (and likely getting wrong) the calorie count of a plate of biryani.

### Meal tracking & analytics
- **Meal Logger** — log what you ate by meal time (Breakfast/Lunch/Dinner/Snack), pulled in directly from a search result or an image-recognition result, or entered manually.
- **Daily Progress** — ring-chart view of calories/protein/carbs/fat against your daily targets.
- **Analytics** — weekly/monthly trend charts (via Recharts), average calories/protein, meal-logging streak.
- **Profile & Goals** — set age/weight/height/activity level, or apply a preset (General Health / Weight Loss / Muscle Gain) that auto-computes calorie and macro targets.
- **AI Suggestions** — sends your last 7 days of logged meals plus your goals and profile to Gemini/Ollama and gets back 4–5 specific, actionable nutrition suggestions (uses Indian food examples).

All of this — meals, goals, profile, biomarkers — persists in **browser localStorage**, not the database. MongoDB is scaffolded (`server/models/Food.js`, `server/config/db.js`) but intentionally not connected; every food lookup hits OpenFoodFacts live and nothing server-side is saved. This is a real architectural choice for a single-user local tool, not an oversight — see Limitations below for what it costs.

### Biomarker tracking
Log blood sugar, total cholesterol, HDL, LDL, triglycerides, heart rate, and blood pressure over time, each checked against standard reference ranges (e.g. blood sugar 70–100 mg/dL normal, ≥126 flagged) with a trend indicator.

### Homemade vs. store-bought comparison
Given a packaged product, generates a homemade recipe alternative and compares cost, nutrition, and difficulty side by side — powered by the same Gemini/Ollama pipeline.

### Safety warnings
Severity-tiered (high/medium/low) warnings surfaced alongside image-recognition results — e.g. hygiene category flags for raw produce — plus a transparency score.

---

## Architecture

```
┌────────────┐   GET /api/v1/food/search?query=&type=&forbidden=   ┌──────────────┐
│  Next.js    │ ─────────────────────────────────────────────────▶ │  Express API  │
│  client     │                                                     │  (port 5055)  │
│ (index.js)  │ ◀───────────────────────────────────────────────── │ foodController│
└────────────┘                                                      └──────┬───────┘
                                                                             │ axios + custom UA
                                                                             ▼
                                                                 ┌────────────────────────┐
                                                                 │     OpenFoodFacts       │
                                                                 │  search-a-licious +     │
                                                                 │  /api/v0 + /api/v2      │
                                                                 └────────────────────────┘

┌────────────┐  POST /api/v1/food/analyze-image {image, mediaType} ┌────────────────┐
│  Next.js    │ ────────────────────────────────────────────────▶  │ imageController │
│  client     │                                                     └───┬─────────┬──┘
└────────────┘                                            Gemini first  │         │ Ollama fallback
                                                                          ▼         ▼
                                                                    Gemini API   llava:7b (local)
                                                                          │         │
                                                                          └────┬────┘
                                                                               ▼
                                                                  cross-checked against
                                                                  indianFoodDB.js (~40 dishes)
```

**Name search** goes through a two-step lookup: `search-a-licious` (typo-tolerant, Elasticsearch-backed) returns 24 candidates, then `pickBestMatch()` re-ranks them by token overlap with the query rather than trusting the search engine's raw relevance score directly — OpenFoodFacts' BM25 ranking scores across ingredient text too, so it can surface an unrelated product whose *description* happens to mention your search term. Falls back to the top relevance result if there's zero token overlap anywhere, preserving typo tolerance.

Full technical write-up of the scoring formula, matching logic, and alternative-finder: [IMPLEMENTATION.md](IMPLEMENTATION.md).

---

## API reference

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/food/search` | GET | Name/barcode lookup, health score, diet check, alternative |
| `/api/v1/food/homemade` | GET | Homemade recipe alternative for a given product |
| `/api/v1/food/analyze-image` | POST | AI food image recognition (Gemini → Ollama fallback) |
| `/api/v1/food/suggestions` | POST | AI-generated nutrition advice from a week of logged meals |

`GET /api/v1/food/search` params: `query` (name or barcode), `type` (`name`/`barcode`), `forbidden` (comma-separated ingredients to flag).

---

## Setup

**Prerequisites:** Node.js 18+. Optional: a free [Gemini API key](https://ai.google.dev/) for image recognition/suggestions (works without one via the Ollama fallback, if Ollama + `llava:7b` are running locally).

```bash
# Backend
cd server
npm install
cp .env.example .env    # optionally add GEMINI_API_KEY
node server.js          # http://localhost:5055

# Frontend (separate terminal)
cd client
npm install
npm run dev              # http://localhost:3000
```

No database or API keys are strictly required — packaged-food search works immediately against OpenFoodFacts. Image recognition and AI suggestions need either a Gemini key or a local Ollama instance running `llava:7b` (vision) and `qwen2.5:7b` (text).

---

## Tech stack

| Layer | Stack |
|---|---|
| Frontend | Next.js 16 (Pages Router), React 19, Tailwind CSS 3, Recharts |
| Backend | Express 5, Axios |
| Food data | [OpenFoodFacts](https://world.openfoodfacts.org/) REST + search-a-licious APIs |
| AI vision/text | Gemini (primary) → Ollama `llava:7b` / `qwen2.5:7b` (local fallback) |
| Scanner | `react-zxing` (`@zxing/library`) |
| Persistence | Browser localStorage (meals, goals, profile, biomarkers); MongoDB scaffolded, not active |

---

## Limitations

- **No server-side persistence or accounts.** Meals, goals, and biomarkers live in browser localStorage — clearing browser data loses everything, and nothing syncs across devices. MongoDB is scaffolded but deliberately not wired up (see `FUTURE.md`).
- **Frontend API URL is hardcoded** to `http://localhost:5055`, not yet environment-configurable — fine for local use, blocks a real deployment.
- **The Indian Food DB only covers ~40 dishes.** Anything outside that list still gets an LLM-only nutrition estimate with no grounding backstop.
- **No automated tests** — the health-score formula and diet-matching logic are pure functions and would be straightforward to test, just not done yet.
- **Some region-specific packaged products simply aren't in OpenFoodFacts** — no fallback (e.g. OCR on the nutrition label) for products the database doesn't have.

---

## License

MIT
