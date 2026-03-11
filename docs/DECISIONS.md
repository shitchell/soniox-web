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
- **Maps to:** G1, V2
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
- **Rationale:** TBD
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
- **Maps to:** G2, V2
- **Tags:** data portability, UX, storage

**Considered:**

| Alternative | Source | Description | Why not? |
|---|---|---|---|
| Defer export/import to v2 | discussed | Smaller v1 scope | user said it "might feel like data loss" without it |
| Keep local data accessible in server mode | discussed | Display message "switch back to local to access" | user said "that feels odd" |
| Auto-migrate on switch (required) | claude considered | No data left behind | user wanted it as an option, not a requirement |

---

### Decisions Requiring Rationale

> The following decisions lack verbatim rationale.
> Would you like to provide it?

- **D11: IndexedDB for storage** — chosen over localStorage/SQLite but rationale
  was not explicitly discussed
