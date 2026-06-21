# TrustAstrology AI

An AI-driven Vedic astrology chat app for baby-name guidance. Given a child's
birth details, it casts a real D1 (Lagna) birth chart, locates the Moon's
Nakshatra and Pada, and reads out the traditional naming syllable through a
conversation with an AI acting as an expert Vedic astrologer.

**Live deployment:** https://trustastrology-moon-git-main-archimanglas-projects.vercel.app/
**Repo:** https://github.com/archimangla/trustastrology-moon

---

## Architecture

The backend is split into two layers: a deterministic chart/lookup step,
and an LLM reasoning step.

```
Birth details
     │
     ▼
/api/chart        →  calls the astro-engine GraphQL API (server-side only,
                      access key never reaches the browser)
     │
     ▼
chart JSON (planets[], ascendant)
     │
     ▼
/api/astrologer    →  STEP 1 (deterministic): find Moon in chart.planets,
                      normalize its nakshatra name, look up the exact
                      naming syllable for that nakshatra + pada in a fixed
                      108-entry reference table. No model involved here.

                      STEP 2 (LLM reasoning): hand those grounded facts to
                      Llama 3.3 70B (via Groq) with a system prompt that
                      forbids inventing chart facts it wasn't given, and
                      asks it to explain the reading and suggest names in
                      plain language.
     │
     ▼
Reply shown in the chat thread
```

The model never determines nakshatra, pada, or syllable, only the
explanation and name suggestions. If the Moon or its nakshatra is missing
or unrecognized, `/api/astrologer` returns an error instead of letting the
model guess (the brief's "without fabricating missing data" requirement).

The Nakshatra→Pada→syllable table and the house-numbering formula were
checked against the two reference videos linked in the assignment (the
worked Sagittarius-ascendant example, and the "Ashwini pada 1 -> chu,
pada 2 -> che" naming rule). See `data/nakshatra-table.js` and
`data/rashi-table.js` for source notes.

---

## Stack

No frontend framework: plain HTML/CSS/JS, deployed as static files.
Backend is two Vercel Serverless Functions (Node, no dependencies, Node
20's built-in `fetch` is enough). Everything lives in one Vercel project.

```
trustastrology-ai/
├── index.html          # birth-detail form + chart/chat layout
├── style.css            # all styling
├── app.js                # form handling, chart SVG rendering, chat logic
├── api/
│   ├── chart.js          # POST /api/chart      → proxies the GraphQL API
│   └── astrologer.js     # POST /api/astrologer  → grounded LLM reasoning (Groq)
├── data/
│   ├── nakshatra-table.js  # 27×4 naming-syllable reference table
│   └── rashi-table.js      # 12 zodiac signs, house-number math
├── package.json
├── .env
└── .gitignore
```

---

## API reference

### `POST /api/chart`

Request body:

```json
{
  "name": "Anaya",
  "gender": "Female",
  "year": 2025, "month": 11, "day": 25,
  "hour": 1, "minute": 55, "second": 0,
  "place": "New Delhi",
  "latitude": 28.6139, "longitude": 77.2090, "timezone": 5.5,
  "chart": "D1"
}
```

`month` accepts either a number (1-12) or a month name. The function
converts it to the name string the GraphQL API expects. Response:
`{ "chart": { name, symbol, planets: [...], ascendant: {...} } }`.

### `POST /api/astrologer`

Request body:

```json
{
  "chart": { /* the object returned by /api/chart */ },
  "messages": [{ "role": "user", "content": "..." }],
  "gender": "Female"
}
```

The full conversation is sent every turn; the function is stateless, no
birth data is stored server-side. Response:
`{ "reply": "...", "facts": { moonNakshatra, moonPada, namingSyllable, ... } }`.

---

## Design notes

Pastel pink palette, no dark theme, no neon. `Fraunces` is used for headings
and the astrologer's chat replies, `Work Sans` for interface copy, and
`IBM Plex Mono` for numeric/data fields (coordinates, timestamps, house
numbers).

The chart is a hand-built SVG of the North Indian (diamond-style) D1 layout
(`renderChart()` in `app.js`), not a chat-bubble UI. House positions are
fixed per the assignment's reference video; the sign number in each house
is computed from the Ascendant; the Moon's house is highlighted. The chat
thread is laid out as a vertical timeline rather than rounded bubbles:
astrologer replies in serif, user messages in mono.

## Known limitations / things to double-check on first real run

- The exact field spellings the live astro-engine API returns for
  `nakshatra` (e.g. "Mrigashira" vs "Mrigashirsha") and `rashi`/`sign`
  names are assumed from the assignment's sample payload and the reference
  videos, since the API's domain wasn't reachable from the environment
  this was built in to test live. Both lookup tables
  (`data/nakshatra-table.js`, `data/rashi-table.js`) include common
  spelling aliases and a fuzzy fallback match, but **the first time you
  run a real chart, check your browser's network tab / Vercel function
  logs for the raw `nakshatra` and `rashi` strings** and add an alias if a
  lookup ever returns `null`.
- This is a naming-guidance tool grounded in the Swar Siddhanta tradition,
  not a substitute for a full consultation. That's stated in the footer
  and in the AI's system prompt (it won't answer questions about doshas,
  career, marriage timing, etc. since the chart data doesn't cover them).
- No birth data is persisted to a database; everything stays in the
  browser tab's memory for the session.