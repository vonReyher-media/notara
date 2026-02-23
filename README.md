# 🎵 Notara

**Open-source sight reading trainer for treble and bass clef.**  
Train your note recognition the same way typing trainers build keyboard muscle memory — one note at a time, timed, progressive, and ruthlessly focused on speed.

> Read faster. Think less. Play more.

🌐 [notara.app](https://notara.app) · ⭐ [Star us on GitHub](https://github.com/vonReyher-media/notara) · 🐛 [Report a bug](https://github.com/vonReyher-media/notara/issues) · 💬 [Discussions](https://github.com/vonReyher-media/notara/discussions)

---

## Why Notara?

Most note reading apps bury you in features. Notara does one thing: shows you a note on a staff and times how fast you name it. No subscriptions, no paywalls, no accounts required to train. Works offline. Open source forever.

---

## Features

- **Treble clef + Bass clef training** — natural notes, accidentals, ledger lines
- **Speed-focused** — response time is your score, not just accuracy
- **Progressive difficulty** — notes unlock as your confidence grows
- **Weighted repetition** — weak notes appear more often, automatically
- **Fully offline** — complete PWA, works without a connection
- **No account required** — train immediately, stats saved locally
- **Global leaderboard** — opt-in, account required, tamper-resistant
- **Passkey auth** — username + passkey, zero passwords stored
- **Multilingual** — English and German from day one, more languages via community contributions
- **Open source** — MIT licensed, self-hostable

---

## Anti-Cheat Architecture

Notara's leaderboard is designed to be as tamper-resistant as possible without sacrificing the open-source nature of the project. Here's exactly how it works and what the trade-offs are — we believe in full transparency.

### The honest truth about open-source web app security

> The client can never be fully trusted. Anyone can read the source, modify it, and submit fake data. The goal is to make cheating hard enough that it isn't worth doing.

### What Notara does to prevent leaderboard manipulation

#### 1. Server-issued session tokens

The server generates a signed session token (HMAC-SHA256) when a training session begins. Every score submission must include this token. Tokens expire after 30 minutes and are single-use — submitting the same token twice is rejected.

```
Client                          Server (Cloudflare Worker)
  |                               |
  |-- POST /api/session/start --> |
  |                               | generates signed token
  |                               | stores token hash in D1
  |<-- { token, expiresAt } ----- |
  |                               |
  | [user trains...]              |
  |                               |
  |-- POST /api/score/submit ---> |
  |   { token, sessionReplay }   | verifies token signature
  |                               | marks token as used in D1
  |                               | recomputes score from replay
  |                               | validates timing plausibility
  |<-- { accepted / rejected } -- |
```

#### 2. Session replay, not raw scores

The client **never sends a score number**. It sends the full session replay — every note shown, every answer given, the timestamp of each event. The server independently recomputes the score and ignores any client-claimed result entirely.

```typescript
// What the client sends
interface SessionReplay {
  token: string
  events: Array<{
    noteId: string        // which note was shown
    answerId: string      // what the user answered
    shownAt: number       // unix ms timestamp
    answeredAt: number    // unix ms timestamp
  }>
}

// The server recomputes: score, accuracy, avg response time
// The server never uses any score value sent by the client
```

#### 3. Timing plausibility checks

The server validates that response times are humanly plausible:

- **< 80ms** response → rejected (faster than human reaction time)
- **Perfectly uniform timing** → flagged (e.g. exactly 300ms on every single note — bot pattern)
- **> 120 notes per minute** sustained → flagged
- **Zero incorrect answers on a brand new account's first session** → flagged for review

These thresholds are configurable and publicly documented. This is not security through obscurity — it is a reasonableness check.

#### 4. Statistical anomaly detection

Each user has a rolling performance profile stored in D1. A submission is flagged if:

- It is more than **3 standard deviations** above the user's rolling average
- Accuracy jumps more than **40 percentage points** between consecutive sessions
- A brand new account posts a top-10 score on their first submission

Flagged scores are held in a pending state, excluded from the leaderboard, and automatically cleared after 7 days of consistent matching performance.

#### 5. Rate limiting

Enforced at the Cloudflare Worker level before requests even reach application logic:

- Max **10 session starts** per user per hour
- Max **10 score submissions** per user per hour
- Multiple accounts from the same IP are throttled

#### 6. Logged-out users: local leaderboard only

If you're not logged in, Notara is fully functional — but your scores are saved to `localStorage` only. There is no global leaderboard entry without an authenticated session token.

- Guests get full training functionality and personal bests locally
- The global board is only populated by verified passkey account holders
- No anonymous leaderboard manipulation is possible

#### What this does NOT prevent

We believe in honesty about limitations:

- A determined attacker who reads this documentation could craft a valid-looking replay with plausible timing. This is difficult but not impossible.
- The server cannot distinguish a very fast human from a slow bot if timings look plausible.
- Self-hosted instances have no connection to the official leaderboard.

For a music note reading trainer the leaderboard is for fun and motivation, not prize money. If you find a bypass, please open a GitHub issue — we'd rather know.

---

## Stack

| Layer | Technology | Why |
|---|---|---|
| Framework | SvelteKit | Lightweight, first-class SSR + server functions, excellent PWA support |
| Adapter | Cloudflare (via `@sveltejs/adapter-cloudflare`) | Edge deployment, global CDN, D1 bindings, generous free tier |
| Styling | Tailwind CSS v4 + typography + forms plugins | Zero component library deps, full control, tiny output |
| Notation | Custom SVG | No VexFlow dependency — note rendering is simple enough to own |
| Auth | Passkeys via SimpleWebAuthn | No passwords stored, phishing-resistant, works on all modern devices |
| Database | Cloudflare D1 (SQLite) | Native edge bindings, no extra auth token, same SQLite API as Turso |
| i18n | Paraglide (inlang) | Compile-time translations, fully type-safe, zero runtime overhead |
| PWA | Vite PWA Plugin | Service worker, offline support, install prompt |
| Testing | Playwright | End-to-end game loop and anti-cheat replay validation tests |
| Component dev | Storybook | Build and tune SVG notation components in isolation |
| Linting | ESLint + Prettier | Consistent code style across contributors |
| AI tooling | MCP | Expose Notara tools to AI assistants for adaptive training features |

---

## Getting Started

### Prerequisites

- Node.js 20+
- pnpm — install via `npm install -g pnpm` or [pnpm.io](https://pnpm.io/installation)
- A Cloudflare account (free tier is sufficient)

### Scaffold the project

To recreate this project from scratch with the exact same configuration:

```sh
pnpm dlx sv create --template minimal --types ts \
  --add prettier eslint playwright \
  tailwindcss="plugins:typography,forms" \
  sveltekit-adapter="adapter:cloudflare+cfTarget:workers" \
  devtools-json \
  paraglide="languageTags:en,de+demo:yes" \
  storybook \
  mcp="ide:opencode+setup:remote" \
  --install pnpm notara
```

Or clone the repo directly:

```sh
git clone https://github.com/vonReyher-media/notara
cd notara
pnpm install
```

### Create your D1 database

```sh
pnpm wrangler d1 create notara-db
```

Copy the database ID into your `wrangler.toml`:

```toml
[[d1_databases]]
binding = "DB"
database_name = "notara-db"
database_id = "your-database-id-here"
```

### Environment variables

For local development create a `.dev.vars` file (Cloudflare's equivalent of `.env`):

```env
SESSION_HMAC_SECRET=your_random_secret_min_32_chars
RP_ID=localhost
RP_NAME=Notara
ORIGIN=http://localhost:5173
```

For production set these in the Cloudflare dashboard under Workers & Pages → your project → Settings → Variables.

### Run migrations

```sh
pnpm wrangler d1 execute notara-db --local --file=./migrations/001_init.sql
```

### Development

```sh
pnpm dev

# or open in browser automatically
pnpm dev -- --open
```

### Preview on Cloudflare Workers locally

```sh
pnpm build
pnpm wrangler pages dev .svelte-kit/cloudflare
```

### Deploy

```sh
pnpm build
pnpm wrangler pages deploy .svelte-kit/cloudflare
```

---

## Project Structure

```
src/
├── lib/
│   ├── notation/              # SVG note and staff rendering (pure, no framework deps)
│   │   ├── staff.ts           # Staff lines, clef symbols, key signatures
│   │   ├── notes.ts           # Note positions, accidentals, ledger lines
│   │   └── theory.ts          # Note names, pitch data, difficulty tiers
│   ├── game/                  # Core training engine (framework-agnostic, fully testable)
│   │   ├── session.ts         # Session state, event recording, replay serialization
│   │   ├── scoring.ts         # Score computation — SAME code runs client AND server
│   │   └── progress.ts        # Weighted note selection, confidence model
│   ├── anticheat/             # Shared validation logic
│   │   ├── token.ts           # HMAC session token generation and verification
│   │   ├── replay.ts          # Replay validation, timing plausibility checks
│   │   └── anomaly.ts         # Statistical anomaly detection against rolling averages
│   ├── auth/                  # SimpleWebAuthn passkey helpers
│   ├── db/                    # D1 client bindings, schema, typed queries
│   └── i18n/                  # Paraglide runtime helpers and type exports
├── routes/
│   ├── +page.svelte           # Landing page + guest training (no login required)
│   ├── train/
│   │   └── +page.svelte       # Main trainer view
│   ├── stats/                 # Personal progress and history
│   ├── leaderboard/           # Global leaderboard (read-only for guests)
│   ├── learn/                 # SEO content routes
│   │   ├── how-to-read-sheet-music/
│   │   ├── treble-clef-notes/
│   │   └── bass-clef-notes/
│   └── api/
│       ├── session/start/     # POST — issues signed session token (auth required)
│       └── score/submit/      # POST — validates replay, recomputes score, updates board
├── paraglide/                 # Auto-generated by Paraglide — do not edit manually
│   └── messages/
│       ├── en.js
│       └── de.js
migrations/
│   ├── 001_init.sql           # Users, sessions, scores, leaderboard tables
│   └── 002_anomaly.sql        # Rolling performance profile tables
messages/                      # Paraglide source translation files
│   ├── en.json                # English (default)
│   └── de.json                # German
wrangler.toml                  # Cloudflare Workers + D1 config
```

### Key design principle: shared scoring code

`game/scoring.ts` is intentionally framework-agnostic and runs identically on both client and server. The client uses it to show a live score during training. The server uses the exact same module to recompute the score from the replay and verify it. One source of truth, no drift.

---

## Internationalisation (i18n)

Notara uses **[Paraglide](https://inlang.com/m/gerre34r/library-inlang-paraglideJs)** for compile-time, fully type-safe translations with zero runtime overhead. All strings are inlined at build time per locale — no JSON fetching, no runtime parsing.

### Supported languages

| Code | Language | Status |
|---|---|---|
| `en` | English | ✅ Complete |
| `de` | German | ✅ Complete |

### Adding a new language

1. Create `messages/[locale].json` following the structure of `messages/en.json`
2. Run `pnpm dlx @inlang/paraglide-js compile --project ./project.inlang`
3. Open a PR — all language additions are welcome

### URL structure

Notara serves localised content under language-prefixed routes, which benefits SEO significantly:

```
notara.app/en/practice/treble-clef   → English
notara.app/de/practice/violinschluessel → German (canonical)
notara.app/de/practice/bassschluessel  → German bass clef
```

Paraglide handles routing and `hreflang` meta tags automatically via the SvelteKit integration.

### Translation keys example

```json
// messages/en.json
{
  "hero.title": "Train your note reading",
  "hero.subtitle": "See a note. Name it fast. Build muscle memory.",
  "clef.treble": "Treble clef",
  "clef.bass": "Bass clef",
  "game.correct": "Correct!",
  "game.wrong": "Wrong — it was {note}",
  "leaderboard.title": "Global leaderboard"
}

// messages/de.json
{
  "hero.title": "Notenlesung trainieren",
  "hero.subtitle": "Note sehen. Schnell benennen. Muskelgedächtnis aufbauen.",
  "clef.treble": "Violinschlüssel",
  "clef.bass": "Bassschlüssel",
  "game.correct": "Richtig!",
  "game.wrong": "Falsch — es war {note}",
  "leaderboard.title": "Weltrangliste"
}
```

---

## Game Logic

### Note selection

Notes are selected using a confidence-weighted random system. Each note has a confidence score from 0–1. Notes with low confidence appear more frequently. Confidence rises on correct fast answers and drops on slow or wrong answers.

```
confidence(note) = clamp(correct_streak × speed_factor, 0, 1)
```

New notes unlock when your average confidence across already-unlocked notes exceeds 0.7.

### Difficulty progression

| Stage | Content |
|---|---|
| 1 | Treble clef — middle octave naturals (C4–G5) |
| 2 | Bass clef — middle octave naturals (C2–G3) |
| 3 | Extended range with ledger lines |
| 4 | Sharps and flats |
| 5 | Mixed clef mode |

---

## Auth

Authentication uses the **WebAuthn / Passkey** standard via [SimpleWebAuthn](https://simplewebauthn.dev/). No passwords are stored anywhere. Users register with a username and their device biometrics or a security key handles authentication.

All session validation and HMAC token signing happens inside Cloudflare Workers. The client never sees the secret.

---

## SEO

Notara targets high-value keywords through content and a deliberate URL structure, with Paraglide providing proper `hreflang` alternate tags for all localised routes.

| Route (EN) | Route (DE) | Target keywords |
|---|---|---|
| `/en` | `/de` | sight reading trainer, Noten lesen lernen |
| `/en/practice/treble-clef` | `/de/practice/violinschluessel` | treble clef practice, Violinschlüssel lernen |
| `/en/practice/bass-clef` | `/de/practice/bassschluessel` | bass clef practice, Bassschlüssel lernen |
| `/en/learn/how-to-read-sheet-music` | `/de/learn/noten-lesen-lernen` | how to read sheet music, Noten lesen |
| `/en/learn/treble-clef-notes` | `/de/learn/violinschluessel-noten` | treble clef note names |
| `/en/learn/bass-clef-notes` | `/de/learn/bassschluessel-noten` | bass clef note names |

---

## Component Development with Storybook

The SVG notation components — staff, noteheads, ledger lines, accidentals, clef symbols — are developed and documented in isolation using Storybook.

```bash
pnpm storybook
```

This is especially useful for tuning the visual rendering of notes across different screen sizes and device pixel ratios before integrating into the full trainer view.

---

## Testing

Notara uses **Playwright** for end-to-end tests. Key test scenarios:

- Full training session: note shown → answer given → score computed
- Anti-cheat: submitting a replay with sub-80ms timings is rejected
- Anti-cheat: submitting the same token twice is rejected
- Anti-cheat: raw score in request body is ignored by server
- Auth: passkey registration and login flow
- i18n: correct language served per route prefix

```bash
pnpm test:e2e
```

---

## Self-Hosting

Notara is fully self-hostable on any Node.js server or on your own Cloudflare account. Self-hosted instances are completely independent and do not connect to the official Notara leaderboard.

```bash
# Clone and deploy to your own Cloudflare account
pnpm wrangler d1 create my-notara-db
pnpm build
pnpm wrangler pages deploy .svelte-kit/cloudflare
```

---

## Contributing

All contributions welcome. Good first issues:

- Additional clefs (alto, tenor)
- Interval recognition mode
- Rhythm reading mode
- Chord recognition mode
- German note names (B → H)
- Solfège mode (Do Re Mi)
- New language translation files
- Capacitor native app wrapper (iOS + Android)

Please open an issue before starting large features so we can coordinate.

---

## Roadmap

- [ ] Core trainer — treble + bass clef, natural notes
- [ ] Accidentals and ledger lines
- [ ] Passkey auth + user accounts
- [ ] Session replay + server-side score validation
- [ ] Statistical anomaly detection
- [ ] Global leaderboard
- [ ] Offline PWA support
- [ ] Progress sync across devices
- [ ] PWA install prompt + app manifest
- [ ] Storybook notation component library
- [ ] Playwright anti-cheat test suite
- [ ] Capacitor iOS / Android wrapper
- [ ] Interval recognition mode
- [ ] Chord recognition mode
- [ ] Alto and tenor clef support
- [ ] Teacher / student mode with private leaderboards
- [ ] MCP tool exposure for AI-assisted adaptive training

---

## License

MIT — free to use, modify, and distribute. Self-host it, fork it, build on it.

---

*Built for musicians who want to read faster and think less.*  
*Notara — train your eyes the way you train your fingers.*