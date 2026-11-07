# Kalenjin — learn, speak and translate

A production-shaped Kalenjin learning and two-way translation app. Built as an installable,
mobile-first Next.js application backed by PostgreSQL, with a dialect-aware linguistic corpus,
a confidence-labelled translation engine, and a real native-speaker review workflow.

> **Linguistic honesty is a product feature.** The app never generates a Kalenjin translation and
> presents it as verified. Every entry has a status, every uncertain result is labelled, and any
> sentence outside the reviewed corpus can be routed to a native-speaker review queue.

## 1. Why a web build instead of a raw Expo build

The runtime available in this environment is a Next.js + PostgreSQL sandbox, so the app ships as a
**installable Progressive Web App** designed at mobile-canvas width:

- `public/manifest.webmanifest` + `public/sw.js` → “Add to Home screen” on Android and iOS.
- No native-only APIs are used in critical flows: speech recognition and synthesis use browser APIs
  with guaranteed text fallbacks.
- The backend is a versioned HTTP API, so a React Native/Expo client can reuse it unchanged.

### React Native / Expo port map

| This build | Expo equivalent |
| --- | --- |
| `src/app/(app)/layout.tsx` + `BottomNav` | `expo-router` `Tabs` |
| `src/components/translator.tsx` | Screen + `expo-av` / `expo-speech` |
| `useSpeechRecognition` | `expo-speech-recognition` (English); manual input for Kalenjin |
| `useSpeechSynthesis` | `expo-speech` with nearest installed voice |
| `useVoiceRecorder` | `expo-av` `Audio.Recording` |
| `/api/*` routes | Same hosted API; keep the token in `expo-secure-store` |
| `localStorage` offline packs | `expo-file-system` + SQLite cache |

## 2. Architecture

```
src/
  app/                     App Router pages + API routes
    (app)/                 Authenticated app shell (header, bottom navigation)
      home  learn  learn/[slug]  dictionary  translate  tutor
      games  culture  practice/pronunciation  profile  contribute  admin  offline
    login  register  onboarding  page.tsx (welcome)
    api/
      auth/{login,register,logout}  preferences  favorites
      lessons/[slug]/complete       translations  tutor
      games/score                   contributions  admin  health
  components/              UI system, translator, lesson player, games, tutor, admin, audio
  content/                 Editorial seed data: dialects, lexicon, phrases, lessons, culture
  db/                      Drizzle schema, connection, idempotent seeder
  lib/                     auth, data access, translation engine, tutors, grading, gamification
```

Key design decisions:

- **Dialect-aware by construction.** Vocabulary and phrases are keyed by
  `(dialect_id, concept_key)`. Lessons reference concept keys, and exercises materialise at request
  time for the learner's variety. If a concept is missing for that variety the exercise is skipped
  and coverage is shown — never borrowed from another variety.
- **Confidence pipeline.** `translateSentence()` resolves in order: exact phrase → dictionary entry →
  fuzzy corpus match (≥0.82, flagged) → word-by-word gloss (flagged approximate) → unknown + human
  review. Optional AI proposals are stored with `confidence_status = 'proposal'`, never “verified”.
- **Review workflow.** Contributions are inserted as `pending`, surfaced in `/admin`, and only then
  published as `community` content. Recording rows carry a required consent flag.
- **Gamification that reflects work.** XP/streak/minutes only accrue from completed lessons, graded
  answers, games and saved translations (`daily_activity` + `award_achievements`).

## 3. Database

Drizzle schema in `src/db/schema.ts`: `dialects`, `users`, `sessions`, `vocabulary`, `phrases`,
`lessons`, `user_progress`, `translations`, `recordings`, `contributions`, `cultural_notes`,
`favorites`, `user_vocabulary`, `achievements`, `game_scores`, `daily_activity`, `tutor_messages`,
`app_meta`.

Indexes/uniques cover lookup paths: `(dialect_id, concept_key)`, normalised word/phrase columns,
`(user_id, lesson_id)`, session token hash, achievement and favourite uniqueness.

Apply and seed:

```bash
npx drizzle-kit push
npx tsx src/db/seed-run.ts
```

The server also calls `ensureSeeded()` lazily under a Postgres advisory lock, so a fresh database
self-populates on first request.

## 4. Accounts

Seeded demo accounts (change before deploying):

| Role | Email | Password |
| --- | --- | --- |
| Learner | `demo@kalenjin.app` | `demo1234` |
| Admin | `admin@kalenjin.app` | `Kalenjin2026!` |
| Contributor | `contributor@kalenjin.app` | `contributor1234` |

Passwords are scrypt-hashed (`src/lib/password.ts`). Sessions use a random 256-bit token; only its
SHA-256 hash is stored, delivered as an `HttpOnly`, `SameSite=Lax` cookie. Set `COOKIE_SECURE=true`
when serving over HTTPS. Mutating routes check same-origin.

## 5. Translation + speech integrations

- **Default engine:** the verified corpus (`src/lib/translate.ts`). No key required.
- **Optional AI proposal provider:** set `OPENAI_API_KEY` (and optionally `OPENAI_MODEL`) to enable
  `src/lib/translation-providers.ts`. Proposals are always labelled *AI proposal · unverified* and
  require review. Without a key the provider is unavailable and the app says so.
- **Default dialect:** `DEFAULT_DIALECT_CODE` (falls back to `kipsigis`); onboarding overrides it.
- **Speech:** recognition/TTS use browser APIs only. Kalenjin is not supported by mainstream speech
  engines; the UI states this and always offers manual input, verified phrasebook lookup and recorded
  audio. Pronunciation practice deliberately gives **no accuracy score** — no validated Kalenjin
  speech-comparison model is integrated.
- **Native audio storage:** recordings store metadata (`audio_url`, speaker, consent, review state).
  Binary upload needs object storage (S3/R2/Supabase Storage). Wire a signed-upload route and replace
  the URL field in the contribution/admin forms when storage is configured.

## 6. Feature status

| Requirement | Status |
| --- | --- |
| Onboarding, auth, preferences, seven varieties | ✅ |
| Six-level curriculum, progressive unlock, review | ✅ |
| Server-graded exercises (5 types) + XP/achievements | ✅ |
| Dictionary with search, categories, favourites, learned state | ✅ |
| Five games + saved scores | ✅ |
| Live translator: text, two-person conversation, history, audio controls, copy, review request | ✅ |
| Confidence labels, uncertainty warnings, no fabricated verified output | ✅ |
| Grounded AI tutor with practice questions + transcript | ✅ |
| Pronunciation: reference/slow playback, recording, no fake scoring | ✅ |
| Cultural learning with verification flags + contribution CTA | ✅ |
| Native-speaker contribution + admin review/publish | ✅ |
| Offline packs, honest online/offline boundaries | ✅ |
| Dark/light themes, responsive mobile-first UI, accessible controls | ✅ |

### Deliberate limitations (documented, not simulated)

1. The seed corpus is intentionally small and status-labelled. Published learner forms are marked
   `verified` (with an in-app reviewer log), derived forms are `needs_review`. Native-speaker review
   is the path to expanding coverage.
2. Kalenjin ASR/TTS models are not integrated. Device voices are labelled as approximations.
3. Audio binary storage is not configured; consent-tracked recording metadata and URLs are supported.
4. Full offline AI translation is **not** claimed.

## 7. Local development

```bash
npm install
npx drizzle-kit push
npx tsx src/db/seed-run.ts
npm run dev
```

Production validation: `npx next typegen`, `npm exec tsc -- --noEmit`, `npm run build`, then start
with `npm run start`. `/api/health` performs a database round-trip for the platform healthcheck.
