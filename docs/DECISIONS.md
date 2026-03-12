# Soniox Web — Decisions

---

## Session: 2026-03-10 — Initial Design & Technology Choices

**Context:** A friend needs to transcribe real-time calls with people speaking
different languages (she speaks Chinese). We researched API providers, self-hosted
options, and hosting tiers, then designed a web app around the Soniox API as the
best cost-to-feature ratio. This session covers all foundational decisions from
provider selection through project structure and data model.

**GVP source:** Inferred inline

### Inferred Goals/Values/Principles (refine later)

- **G1: Cheap multilingual transcription** — Provide real-time transcription and
  translation at the lowest possible cost for regular use (~20 hrs/month)
- **G2: Minimal friction for end user** — The friend should be able to use the
  app with minimal setup, ideally no API key management
- **G3: Future Android app** — The web app should be convertible to a native
  Android app for distribution to friends (with potential ad revenue)
- **V1: Modularity** — UI components and features should be easy to rearrange
  and reuse as the design evolves through real-world use
- **V2: Simplicity** — Prefer the simplest solution that works; avoid
  over-engineering
- **P1: One codebase, multiple targets** — Web and Android from a single
  codebase rather than maintaining separate projects
- **P2: Generic repo, custom deployments** — The repo itself stays
  provider/server-agnostic; specific deployments are configured via forks or
  CI pipelines
- **V3: No data loss** — Switching modes or configurations should never strand
  user data; provide export/import paths

---

### D1: Use Soniox API as the transcription/translation provider

> Chose Soniox over other API providers and self-hosted options for real-time
> multilingual transcription and translation.

- **Chosen:** Soniox API (~$0.12/hr real-time)
- **Rationale:** user said, "i think the soniox API + custom webpage is probably the best bet"
- **Maps to:** G1, V2
- **Tags:** provider, cost

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Soniox Pro app ($20/mo) | discussed | Ready-made app, no coding needed | ~10x more expensive at 20 hrs/month (~$20 vs ~$2.40) |
| Soniox free tier (10 credits/week) | discussed | Free but limited | Not enough for regular use |
| Mistral Voxtral ($0.006/min) | discussed | Cheapest per-minute API | Missing diarization/translation bundling |
| OpenAI Whisper API ($0.006/min) | discussed | Wide language support | No built-in diarization or translation |
| Deepgram Nova-3 ($0.0077/min) | discussed | Fast, good diarization | Chinese support unclear for Nova-3 |
| Self-hosted faster-whisper | discussed | Full control, no per-minute cost | Dev/infra overhead not worth it at this volume |
| Gladia (~$0.01/min) | discussed | Good features, 100+ languages | More expensive, diarization/translation are add-ons |
| Speechmatics | discussed | Strong multilingual + translation | Enterprise-priced, contact sales |

---

### D2: Mobile-first web app (also works on desktop)

> The app targets mobile as the primary form factor since the user will likely
> have it open during phone calls.

- **Chosen:** Mobile-first responsive web app
- **Rationale:** user said, "mobile first, also works on desktop"
- **Maps to:** G2
- **Tags:** platform, UX

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Desktop browser app | discussed | Laptop during calls | user chose mobile-first |
| Mobile-only native app | claude considered | Skip web entirely | Limits accessibility; web-first is more flexible |
| Call recording processor (batch only) | discussed | Upload files after the fact | user said, "realtime transcription with diarisation if supported by the API" |

---

### D3: Feature set — real-time + file upload + translation + diarization

> The app includes real-time mic transcription, file upload for pre-recorded
> media, speaker diarization, and configurable translation.

- **Chosen:** All four features included from the start
- **Rationale:** user said, "would be nice to include options in the app for diarization, translation, real-time conversational stuff between multiple speakers, and processing pre-recorded audio/video files"
- **Maps to:** G1, G2
- **Tags:** features, scope

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Real-time only, add file upload later | claude considered | Smaller initial scope | User explicitly wanted both |
| Transcription only, no translation | claude considered | Simpler | Translation is core to the use case |

---

### D4: Flexible language picker (user-selectable source/target)

> Rather than hardcoding Chinese-English, the app lets users pick any
> source/target language from Soniox's supported list.

- **Chosen:** Configurable language dropdowns
- **Rationale:** Claude recommended option 3 (flexible) since "Soniox supports 60+ languages and it's barely more work to make it configurable", user said, "3 :)"
- **Maps to:** G2, V2
- **Tags:** i18n, UX

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Chinese → English only | discussed | Simplest possible | Too limiting |
| Bidirectional Chinese ↔ English | discussed | Covers main use case | Barely any more work to make it fully flexible |

---

### D5: "View transcript in any language" with translation caching

> Saved transcripts can be viewed in the original language(s) or translated to
> any language on demand. Translations are cached so switching is instant.

- **Chosen:** On-demand translation with per-segment caching in IndexedDB
- **Rationale:** user said, "i also like how their mobile app interface lets you save the transcript afterwards and show (1) the original in whatever language each speaker used or (2) some language you select. so auto-translation built-in. obvs we'd want to cache afterwards so that we aren't retranslating lol"
- **Maps to:** G1
- **Tags:** translation, caching, cost

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Pre-translate to all languages on save | claude considered | Instant switching from the start | Wasteful; user may never view most languages |
| No caching, re-translate each time | claude considered | Simpler storage | user said, "we'd want to cache afterwards so that we aren't retranslating" |

---

### D6: Dual-mode app — Local (standalone) or Server (full backend)

> The app operates in two modes: Local mode (everything client-side, BYO API key,
> IndexedDB storage) or Server mode (full backend with auth, server-side storage,
> managed API key). User selects via a radio in Settings.

- **Chosen:** User-selectable Local/Server mode with a storage abstraction layer
  that both modes implement
- **Rationale:** user said, "i do have a server i can throw this on lol, and
  that makes storing the API key easier. i'm torn; i love the idea of letting my
  friend be fully self-supported and making it easier." Later refined: user said,
  "what if, instead of a configuration option for API calls, we just have a
  generic 'Backend' option? if left blank -- you get the full single-user
  localStorage experience. if you select a backend, it unlocks user
  accounts/login, server-side API calls, backend storage, etc... so our server
  component becomes a full backend, and the frontend has wiring that allows it to
  flip between local/server-provided option."
- **Maps to:** G2, P2
- **Tags:** architecture, deployment, storage

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Static-only (always user-managed key) | discussed | Simplest architecture | Less convenient for the friend |
| Backend-only (always requires server) | discussed | Simpler code path | Loses free GitHub Pages hosting option |
| Auto-detect via `/api/config` endpoint | discussed (initial proposal) | Automatic mode switching | user preferred explicit user-selectable radio over magic detection |

---

### D7: Fork-specific deployment pipeline for server config

> The main repo stays generic. A fork's CI pipeline configures it to point at
> a specific backend server.

- **Chosen:** Fork-level CI config, not in the main repo
- **Rationale:** user said, "we can optionally set her up with a little GitHub deployment pipeline that automatically configures it to point at my server by default. that way the project itself doesn't natively point at my server, but her instance can."
- **Maps to:** P2
- **Tags:** deployment, CI

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Hardcode server URL in repo | claude considered | Simpler | Couples the repo to a specific server |
| Runtime config file | claude considered | Editable without CI | More manual setup for end user |

---

### D8: Expo (React Native) from the start

> Use Expo to build for web and Android from one codebase, rather than starting
> with plain React and converting later.

- **Chosen:** Expo with Expo Router
- **Rationale:** user said, "we might convert this to an android app... at some point, so i'm leaning towards React Native." Claude recommended Expo specifically since "the Android conversion is zero work later" and "Expo has basically won this space."
- **Maps to:** G3, P1
- **Tags:** framework, platform

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| React (Vite) now, convert later | discussed | Lighter web app today | Requires rewriting UI components for React Native later |
| React Native Web (no Expo) | discussed | Middle ground | "more config headaches for little benefit" since Expo covers this |

---

### D9: Shared component library outside feature modules

> UI components live in a top-level `components/` directory, not inside feature
> modules, so they can be reused across features or moved between screens freely.

- **Chosen:** `components/` at project root, separate from `features/`
- **Rationale:** user said, "can we store the components outside of each module, that way we can easily (re-)use them in different modules or swap where they live on a whim?"
- **Maps to:** V1
- **Tags:** architecture, modularity

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Components co-located in feature modules | discussed (initial proposal) | Closer to where they're used | user explicitly wanted them separate for reuse and flexibility |
| Both shared + feature-local components | claude considered | Hybrid approach | Adds ambiguity about where to put things |

---

### D10: Thin page shells with hooks-based feature logic

> Pages in `app/` are thin — they compose shared components and wire up feature
> hooks. Business logic lives in `features/` as hooks and stores.

- **Chosen:** `app/` pages as thin shells, `features/` for logic, `components/` for UI
- **Rationale:** user said, "i have no idea how we'll feel about the UI in practice, and it is highly possible we will move things around, so a very modular design is ideal"
- **Maps to:** V1
- **Tags:** architecture, modularity

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Fat page components with inline logic | claude considered | Fewer files, simpler at first | Harder to rearrange; conflicts with modularity value |
| State management library (Redux, Zustand) | claude considered | More structured state | Not discussed; hooks + stores is simpler for this scope |

---

### D11: IndexedDB for transcript and translation cache storage

> Transcripts and translation caches are persisted to IndexedDB for larger
> storage capacity than localStorage.

- **Chosen:** IndexedDB via a `storage/db.ts` abstraction
- **Rationale:** user said, "IndexedDB is the goldilocks between localStorage size limits and sqlite overkill."
- **Maps to:** V2
- **Tags:** storage, persistence

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| localStorage | claude considered | Simpler API | Size limits (~5-10MB) may be too small for many transcripts |
| SQLite (via WASM) | claude considered | More powerful queries | Heavier dependency, likely overkill |
| Server-side storage | claude considered | Sync across devices | Requires backend; conflicts with static-site mode |

---

### D12: Settings persisted via localStorage with config.json defaults

> The mode selection (Local/Server) and backend URL are stored in localStorage.
> Default values come from a statically served config.json that CI/CD can update.

- **Chosen:** localStorage for settings persistence, config.json for defaults
- **Rationale:** user said, "settings page, persisted via localStorage. default
  value is configurable via a statically served config.json or similar that we
  can update through CI/CD."
- **Maps to:** P2, G2
- **Tags:** settings, deployment, CI

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Build-time env var for defaults | discussed | Baked in at deploy time | Less flexible; user can't change without rebuild |
| Server-side settings only | claude considered | Simpler for server mode | Doesn't work in local mode |

---

### D13: Login screen with "use locally" escape hatch

> When a backend is configured as default, users see a login/signup screen with
> an option to skip and use the app locally without an account.

- **Chosen:** Login screen with "...or use locally without an account" link
- **Rationale:** Claude proposed a pre-login settings menu as an alternative.
  user said a settings menu before login "raises the question 'where are these
  settings even stored?' and feels like a liminal space" — Claude agreed and
  recommended the login screen approach instead.
- **Maps to:** G2
- **Tags:** UX, auth

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Minimal pre-login settings menu | discussed | Settings available before auth | Feels like a "liminal space"; unclear where settings are stored |
| Force login, no local option | claude considered | Simpler auth flow | Conflicts with standalone/GitHub Pages mode |

---

### D14: Export/import in v1 with sync-on-switch prompt

> Export/import of transcripts (as JSON) is a v1 feature. When switching from
> local to server mode, the app prompts to import local transcripts. A persistent
> "Sync local data to server" option exists in Settings > Data.

- **Chosen:** v1 export/import + sync prompt on mode switch + persistent sync
  option in settings
- **Rationale:** user said, "if we *don't* allow export/import, that might feel
  like data loss." and "i might consider including export/import in v1? and if
  someone switches to backend mode, present an option (not a requirement) to
  import their locally stored data either at transition or at any other point (a
  persistent 'Sync local data to server' option that grays out once they do."
- **Maps to:** G2, V3
- **Tags:** data portability, UX, storage

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Defer export/import to v2 | discussed | Smaller v1 scope | user said it "might feel like data loss" without it |
| Keep local data accessible in server mode | discussed | Display message "switch back to local to access" | user said "that feels odd" |
| Auto-migrate on switch (required) | claude considered | No data left behind | user wanted it as an option, not a requirement |

---

### Decisions Requiring Rationale

> None — all decisions have documented rationale.

---
---

## Session: 2026-03-12 — Technical Research & Translation Architecture

**Context:** After completing the initial design, we ran two review agents
against the DECISIONS.md: one with full context (consistency review) and one
cold-read (no context). Both identified gaps that required technical research
rather than discussion — specifically around Soniox API capabilities, Android
audio constraints, Expo workflow requirements, and cross-platform storage. This
session captures decisions informed by that research.

**GVP source:** Inferred inline (same as Session 1)

---

### D15: Pluggable text translation service layer

> Soniox has no text-translation endpoint. Post-hoc translation of saved
> transcripts requires a separate translation layer with a plugin architecture.

- **Chosen:** A `TranslationService` interface that plugins conform to, with
  auto-discovery, per-service configuration (key/value pairs with types and
  optional defaults), and per-service settings persistence. Initial test
  services: Pig Latin and 1337 Speak.
- **Rationale:** Soniox API research confirmed there is no text-translation
  endpoint — translation is only available during audio processing. User said,
  "let's add a text translation layer for translating transcriptions with a text
  translate interface that various plugins for different services could conform
  to." User said services should "define key/value pairs it requires (e.g.: api
  keys, credentials, etc...) in local mode." User said, "selecting a service
  would cause inputs to magically appear for that service's defined inputs.
  configured values get saved -- locally or server-side -- such that, if a user
  switches between translation services, their settings for each service gets
  'remembered'." For initial testing, user said, "i would suggest we add two
  silly services to start with just for testing: pig latin and 1337 speak. the
  1337 speak one in particular should allow configuring custom replacement
  patterns and letters to *not* replace."
- **Maps to:** V1, G1
- **Tags:** translation, architecture, plugins

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Re-process audio through Soniox async API with translation | discussed | Keep everything in Soniox | Expensive (re-processes full audio), slow, wasteful |
| Only support languages translated during live session | discussed | No extra service needed | Severely limits the "view in any language" feature |
| Hardcode a single translation service (e.g., Google Translate) | claude considered | Simpler | Conflicts with V1 (modularity) and P2 (generic repo) |

---

### D16: Expo prebuild (CNG) workflow, not pure managed

> Real-time audio streaming to a WebSocket requires native modules not available
> in Expo's pure managed workflow.

- **Chosen:** Expo prebuild/CNG workflow with `@siteed/expo-audio-studio` for
  real-time PCM audio chunk streaming
- **Rationale:** Research confirmed that Expo's official `expo-audio` and
  `expo-av` only support record-to-file, not real-time chunk streaming. The
  `@siteed/expo-audio-studio` package provides `onAudioStream` callbacks with
  raw PCM data but requires `npx expo prebuild`. Modern Expo prebuild is not the
  old painful "eject" — it still uses Expo tooling and config plugins.
- **Maps to:** G3, P1
- **Tags:** framework, audio, native

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Pure managed workflow with expo-audio | discussed | Simpler setup | No real-time chunk streaming API; record-to-file only |
| MediaRecorder API (web only) | discussed (initial design) | Works on web without native modules | Not available on native Android; doesn't solve cross-platform |
| Web Audio API / AudioWorklet | claude considered | Lower-level web audio control | Web-only, same cross-platform problem |

---

### D17: Custom storage abstraction (IndexedDB on web, expo-sqlite on native)

> IndexedDB is a browser API not available on native Android. Rather than bet on
> expo-sqlite's alpha web support, use a custom abstraction layer.

- **Chosen:** Storage interface with two implementations: IndexedDB for web,
  expo-sqlite for native Android
- **Rationale:** Research confirmed IndexedDB is unavailable on native Android.
  expo-sqlite has web support but it is in alpha, requires WASM config and
  specific HTTP headers (COEP/COOP). User said, "while it's still in alpha, i'd
  go for the custom abstraction."
- **Maps to:** G3, V2
- **Tags:** storage, cross-platform

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| expo-sqlite everywhere (web + native) | discussed | Single library | Web support is alpha; requires WASM and COEP/COOP headers |
| AsyncStorage | discussed | Simple key-value | 6MB size limit on Android; not suitable for transcript storage |
| RxDB | discussed | Reactive, offline-first, sync support | Heavy dependency, likely overkill |
| WatermelonDB | discussed | React Native optimized | No web support |

---

### D18: Audio capture via device microphone (speakerphone model)

> Android restricts direct call audio capture to system/pre-installed apps.
> The app captures audio from the device microphone only.

- **Chosen:** Device microphone capture; user uses speakerphone during calls
- **Rationale:** Research confirmed that Android's audio input sharing policy
  gives phone calls top priority for the microphone — third-party apps receive
  silence. Direct call audio capture requires system-level permissions
  (`CAPTURE_AUDIO_OUTPUT`) unavailable to third-party apps. Google killed the
  accessibility service loophole in May 2022. The AudioPlaybackCapture API
  (Android 10+) also does not capture call audio. The practical approach is
  speakerphone + device mic, which captures both sides of the conversation.
- **Maps to:** G3
- **Tags:** audio, android, constraints

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Direct call audio capture | discussed | Capture phone audio stream directly | Requires system permissions unavailable to third-party apps |
| AudioPlaybackCapture API | discussed | Android 10+ system audio capture | Does not capture call audio (excluded usage type) |
| Accessibility service workaround | discussed | Was used by call recording apps | Google killed this in May 2022; apps removed from Play Store |

---

### D19: Soniox real-time diarization (with known accuracy tradeoff)

> Soniox supports real-time diarization via WebSocket, but with lower accuracy
> than async processing due to latency constraints.

- **Chosen:** Use real-time diarization for live sessions, with the understanding
  that accuracy is lower than async. Async API provides better diarization for
  file uploads.
- **Rationale:** Research confirmed Soniox real-time WebSocket API supports
  diarization (up to 15 speakers) via `enable_speaker_diarization: true`. The
  docs note "real-time speaker diarization is more challenging due to
  low-latency constraints" with possible temporary speaker switches that
  stabilize as more context arrives. The async API provides "significantly higher
  diarization accuracy because the model has access to the full audio context."
- **Maps to:** G1
- **Tags:** diarization, soniox, accuracy

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Skip real-time diarization, post-process via async | claude considered | Better accuracy | Defeats the purpose of real-time transcription |
| Client-side diarization (pyannote via WASM) | claude considered | Independent of Soniox | Heavy, complex, likely too slow for real-time in browser |

---
---

## Session: 2026-03-12 — Implementation Decisions

**Context:** With the architecture and technical constraints established, we
needed to make concrete implementation decisions before starting to code:
backend stack, auth, state management, styling, testing strategy, data sync
semantics, export formats, and distribution plans.

**GVP source:** Inferred inline (same as Session 1)

### New Inferred Values/Principles

- **P3: Tests match requirements, not code** — tests validate behavior against
  specs, not implementation details. No green checkmarks for the sake of green
  checkmarks.
- **P4: Bugs compound** — do not greenlight failing states. If tests fail and
  we can't figure it out, we stop. We do not move on.
- **V4: Extensibility** — export formats, translation providers, and similar
  features should use pluggable/registry patterns so new options can be added
  without modifying existing code.

---

### D20: Node.js + TypeScript backend with ORM supporting SQLite and Postgres

> The backend uses the same language as the frontend (TypeScript) with an ORM
> that abstracts the database.

- **Chosen:** Node.js + TypeScript backend, ORM that supports both SQLite and
  Postgres
- **Rationale:** user said, "if we're doing react, i'd go ahead and keep it
  node + typescript. for database, i always love a good ORM so that it doesn't
  matter :p my only requisite is it must support both sqlite and postgres."
- **Maps to:** V2, P1
- **Tags:** backend, database, stack

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Python/FastAPI | claude considered | Popular for APIs | Different language from frontend; user preferred keeping it TypeScript |
| Raw SQL / no ORM | claude considered | More control | user explicitly wanted an ORM for database portability |

---

### D21: Email/password auth with future OAuth path

> Server mode uses basic email/password authentication, designed so OAuth can
> be added later without breaking changes.

- **Chosen:** Email/password/salt authentication, architected for future OAuth
- **Rationale:** user said, "i'd go with basic email/password/salt for now,
  designing around a future OAuth addition."
- **Maps to:** G2, V2
- **Tags:** auth, server mode

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| OAuth from the start | claude considered | More secure, social login | Over-engineering for current scope |
| Magic links | claude considered | No password management | Not discussed |
| Firebase/Supabase Auth | claude considered | Managed auth service | Adds external dependency; conflicts with self-hosted flexibility |

---

### D22: Zustand for state management

> Feature state is managed with Zustand stores — one store per feature module.

- **Chosen:** Zustand — one store per feature in `features/*/store.ts`
- **Rationale:** user chose Zustand after comparing it with Jotai. Zustand's
  "one store per feature" model maps directly to the `features/*/store.ts`
  structure. user said, "i think zustand sounds good for our purposes."
- **Maps to:** V1, V2
- **Tags:** state management, frontend

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Jotai | discussed | Atomic state model, less boilerplate per atom | Better for lots of independent atoms; our state is more "store-shaped" |
| React Context + useReducer | discussed | Built-in, no dependencies | Can cause unnecessary re-renders |
| Plain module-level variables | discussed | Simplest | Not reactive; components don't auto-update |
| Redux | discussed (D10) | Most structured | Overkill for this scope |

---

### D23: Plain CSS with variables and relative sizing

> Styling uses plain CSS organized into semantic files with CSS variables for
> theming and relative units (rem/em/%) instead of px.

- **Chosen:** Plain CSS files (common, theme, helpers, page-specific as needed),
  CSS variables, relative sizes
- **Rationale:** user said, "i kinda prefer just CSS organized into files as
  relevant (common, theme, helpers, page-specific, whatever makes sense for the
  layout) with gratuitous variables for easy updates. oh, and i tend to favor
  relative sizes vs px."
- **Maps to:** V2
- **Tags:** styling, frontend

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| NativeWind/Tailwind | claude considered | Utility-first, popular in RN | user preferred plain CSS |
| styled-components | claude considered | CSS-in-JS, scoped styles | user preferred plain CSS |
| React Native StyleSheet | claude considered | RN standard | Less flexible than CSS for web-first |

---

### D24: Testing strategy — unit tests, integration tests with real API shapes

> Comprehensive unit tests for everything, integration tests using mock Soniox
> API results based on real documentation or actual API responses.

- **Chosen:** Unit tests for all features, integration tests with mocked Soniox
  API based on real docs/responses. Tests match requirements, not code. No
  moving on with failing tests.
- **Rationale:** user said, "unit tests for everything, at least a handful of
  integration tests that comprehensively test things with mock soniox api results
  (should be based on docs or ideally actual soniox api results -- NOT EVER just
  whatever we imagine the results probably look like). and i shouldn't have to
  say this, but tests should match requirements, not code. no green checkmarks
  for the sake of green checkmarks. also no moving on until all tests pass. if
  that means we have to stop at some point because we are getting failing tests
  but can't figure it out, then we stop. we do not greenlight failing states.
  bugs compound."
- **Maps to:** P3, P4
- **Tags:** testing, quality

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Tests with imagined API responses | discussed | Faster to write | user explicitly forbade this: "NOT EVER just whatever we imagine the results probably look like" |
| Defer testing | claude considered | Ship faster | Contradicts P4 (bugs compound) |

---

### D25: Transcript sync semantics with shared IDs

> Transcripts share IDs across local and server storage. Sync follows clear
> source-of-truth rules depending on mode and direction.

- **Chosen:** Shared transcript IDs. Local-to-server sync overwrites server.
  Server-mode updates propagate to local cache. Server-to-local sync on mode
  switch (optional, overwrites local). Designed for future detailed conflict
  resolution.
- **Rationale:** user said, "i would assume they should share identical
  transcript IDs. let's make sure that's true -- a requirement. when in local
  mode, obvs the local is the SOT. when switching from local to server, the
  local should simply overwrite whatever is on the server. when in server mode
  and we update a transcript/translation, i would first make that update on the
  server-side, then check 'do we have any identical transcript IDs locally?' and
  if yes, then copy the server version as the SOT to the locally cached version.
  when switching from server to local mode, i would ask the user if they want to
  sync all of their server data. this would simply overwrite any local copies
  with server-side stuff. we should design around the idea that we might, in the
  future, want to do more detailed conflict checking (i.e.: if we have
  conflicting translations)."
- **Maps to:** V3, G2
- **Tags:** sync, storage, data model

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Separate IDs per storage backend | claude considered | Simpler, no conflicts possible | user explicitly required shared IDs |
| Full bidirectional sync (CRDT/merge) | claude considered | Most robust | Over-engineering for now; user said "design around the idea that we might, in the future, want to do more detailed conflict checking" |
| No local cache in server mode | claude considered | Simpler | Loses offline viewing capability |

---

### D26: Per-user data with shareable transcript links

> In server mode, each user's data is isolated. Transcripts can be shared via
> links.

- **Chosen:** Per-user isolation with share link capability
- **Rationale:** user said, "per-user, but the ability to create share links
  would be cool!"
- **Maps to:** G2
- **Tags:** multi-user, sharing, server mode

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Fully shared/collaborative workspace | claude considered | Team use case | Not the primary use case; adds complexity |
| Per-user with no sharing | claude considered | Simpler | user wanted share links |

---

### D27: Extensible export formatters (markdown, text, SRT, VTT, JSON)

> Export uses a pluggable formatter registry. Import only accepts JSON.

- **Chosen:** Export dropdown with markdown, text, SRT, VTT, and JSON options
  via extensible formatter pattern. Import restricted to JSON data export format.
- **Rationale:** user said, "i would have the import only accept the JSON data
  export, but for the export i'd allow a dropdown -- markdown, text, srt, vtt,
  or JSON. i'd prefer an extensible formatter setup for this."
- **Maps to:** V4, G2
- **Tags:** export, import, data portability

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| JSON only for both import and export | claude considered | Simplest | Limits usefulness for non-technical users |
| Accept multiple import formats | claude considered | More flexible | user explicitly restricted import to JSON |

---

### D28: Android distribution — sideloading now, Play Store later

> Initial Android distribution via APK sideloading. Play Store submission
> planned for later.

- **Chosen:** Sideloading for initial distribution, Play Store eventually
- **Rationale:** user said, "play store eventually, side-loading for now."
- **Maps to:** G3
- **Tags:** android, distribution

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Play Store from the start | claude considered | Wider reach, auto-updates | More overhead (signing, review, policies) for initial release |
| Web-only, skip native entirely | claude considered | Simpler | Conflicts with G3 (future Android app) |

---

### D29: No backwards compatibility concerns for v1

> v1 is the first version. No backwards compatibility guarantees or migration
> paths from pre-v1 are needed.

- **Chosen:** No backwards compatibility constraints
- **Rationale:** user said, "idgaf about backwards compatibility."
- **Maps to:** V2
- **Tags:** versioning, scope

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Stable data format from day one | claude considered | Easier upgrades later | Over-engineering; user explicitly deprioritized this |

---

## Session: 2026-03-12 — Phase 1 Plan Review Fixes

**Context:** Three review agents and a jest-native research agent found issues
in the Phase 1 implementation plan. These decisions resolve the findings.

### D30: speakerId is string, not number

> Soniox WebSocket API returns speaker IDs as strings (e.g., `"1"`, `"2"`).
> The original design doc listed `speakerId: number`. Updated to `string`
> throughout types to match the wire format.

- **Chosen:** `speakerId: string` in Token and Segment types
- **Maps to:** G1 (correct data handling)
- **Rationale:** Match the actual API response. Avoids unnecessary type
  coercion at the boundary.

### D31: Cross-platform styling via TypeScript theme object

> Plain CSS (D23) is incompatible with React Native — no CSS variables, no
> `:root`, no standard selectors on native. For web-only development D23
> still applies, but the foundation layer uses a TypeScript theme object
> (`styles/theme.ts`) that works with `StyleSheet.create()` on native and
> inline styles on web.

- **Chosen:** TypeScript theme constants (`colors`, `spacing`, `fontSizes`,
  `radii`, `speakerColors`) consumed by StyleSheet.create() or inline styles
- **Rejected:** NativeWind/Tailwind (user prefers plain CSS feel), CSS-in-JS
  libraries (unnecessary dependency)
- **Maps to:** P1 (one codebase), G3 (Android), D23 (plain CSS spirit)
- **Rationale:** user said, "A sounds good to me" — TS theme is the simplest
  cross-platform approach that preserves the design token structure from the
  original CSS plan.

### D32: Export formatters support language substitution with strict mode

> Formatters accept `format(transcript, language?, strict?)`. When `language`
> is provided, segment text is replaced with the cached translation. `strict`
> (default `true`) throws `MissingTranslationError` on missing translations;
> `false` falls back to original text.

- **Chosen:** `strict: boolean = true` parameter on all formatters
- **Maps to:** V3 (no data loss), D27 (export formatters)
- **Rationale:** user said, "i would suggest we add a parameter
  `format(transcript, language?, strict: bool = True)`. if strict == True,
  then the function throws an error on missing translations. if false, then
  it substitutes the original segment — whatever language it happened to be
  in." UI will use pre-export validation to check for gaps before calling
  format, and present a confirmation dialog. For now, declining cancels the
  export; the design supports swapping in alternative behaviors later (e.g.,
  a secondary modal offering to continue with original text).

### D33: @testing-library/jest-native is deprecated — use built-in matchers

> `@testing-library/jest-native` was merged into `@testing-library/react-native`
> v12.4+. Custom matchers auto-register on import. No `setupFilesAfterEnv`
> entry needed.

- **Chosen:** Do not install `@testing-library/jest-native`. Use
  `@testing-library/react-native` only. Correct Jest key is
  `setupFilesAfterEnv` (not `setupFilesAfterSetup`).
- **Maps to:** D24 (testing strategy)
- **Rationale:** Deprecated package. Built-in matchers are zero-config.

### D34: Zustand settings store persisted via `persist` middleware

> Settings were in-memory only — lost on page refresh. Added Zustand
> `persist` middleware with `localStorage`. API key is excluded from
> persistence (security — re-enter per session).

- **Chosen:** `zustand/middleware` `persist` with `createJSONStorage(() => localStorage)`
- **Maps to:** V2 (simplicity), D22 (Zustand)
- **Rationale:** One-line middleware addition. Without it, the settings store
  is only useful for tests. Phase 3+ depends on settings surviving refresh.

---

### Decisions Requiring Rationale

> None — all decisions have documented rationale.
