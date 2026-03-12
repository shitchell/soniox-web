## 1. Project Overview

**Soniox Web** is a real-time multilingual transcription and translation web application built for a specific user -- the developer's friend who regularly participates in calls involving multiple languages (primarily Chinese and English). The app is designed to run during phone calls, transcribing speech in real time with speaker diarization, and providing on-demand translation between any of Soniox's 60+ supported languages.

**Who it's for:** Initially one person (the developer's friend), with a future goal of distributing it more broadly to friends via an Android app, potentially with ad revenue.

**What it does (v1 feature set):**
- Real-time microphone transcription during calls
- File upload for processing pre-recorded audio/video
- Speaker diarization (identifying who said what)
- Configurable source/target language translation (not hardcoded to Chinese/English)
- Transcript saving with cached translations (view any saved transcript in any language; translations are cached per-segment in IndexedDB so they only cost one API call)
- Export/import of transcripts as JSON for data portability
- Sync prompt when switching between local and server modes

**Architecture -- dual mode:**
- **Local mode:** Fully client-side, user provides their own Soniox API key, data stored in IndexedDB, deployable as a static site (e.g., GitHub Pages)
- **Server mode:** Full backend with authentication, user accounts, server-managed API key, server-side storage. Unlocked by pointing the app at a backend URL.
- The user selects the mode via a radio button in Settings. When a backend is configured as the default, users see a login screen with a "use locally without an account" escape hatch.

**Tech stack:**
- **Expo (React Native) with Expo Router** -- single codebase targeting web (primary) and Android (future)
- **Mobile-first responsive design** (also works on desktop)
- **IndexedDB** (via a `storage/db.ts` abstraction) for transcripts and translation caches
- **localStorage** for settings, with a statically served `config.json` providing defaults (overridable by CI/CD)

**Project structure:**
- `app/` -- thin page shells that compose components and wire up feature hooks
- `components/` -- shared UI component library at project root (not inside feature modules)
- `features/` -- business logic as hooks and stores

**Deployment model:** The main repo is generic and server-agnostic. Specific deployments (e.g., pointing at the developer's server) are done via forks with CI pipelines that update `config.json` defaults.

**Cost:** Soniox API at ~$0.12/hr real-time, estimated ~$2.40/month at 20 hours of use.

---

## 2. Inferences

| # | What the document says | What I inferred |
|---|------------------------|-----------------|
| 1 | "she speaks Chinese" and calls involve "different languages" | The primary use case is Chinese-English bilingual calls, though the app is generalized beyond this. |
| 2 | Soniox API at ~$0.12/hr | The Soniox API handles both transcription and translation in a single call (bundled), since other providers were rejected for lacking bundled diarization/translation. |
| 3 | "features/ for logic" with "hooks and stores" | State management uses React hooks and likely simple context/store patterns (not Redux/Zustand), though no specific state library is named. |
| 4 | Expo with Expo Router | The web version is served via Expo's web export, not a separate React web build. |
| 5 | "Server mode" with auth, user accounts, server-side storage | There is a backend component to be built (API server), though no backend technology is specified. |
| 6 | `storage/db.ts` abstraction | Both local and server modes implement the same storage interface, allowing the app to swap transparently between IndexedDB and server-side API calls. |
| 7 | "potential ad revenue" for Android app | The Android distribution is aspirational/future and not part of the current build scope. |
| 8 | Export/import as JSON | The JSON format contains transcript segments with speaker labels, timestamps, original text, and cached translations. |
| 9 | "config.json or similar that we can update through CI/CD" | The config.json is served as a static asset alongside the built app, fetched at runtime. |
| 10 | Login screen with "use locally" link | Server mode requires some form of authentication (signup/login), but no auth provider or method is specified. |

---

## 3. Ambiguities

1. **What "Server mode" actually provides beyond storage and auth.** D6 says server mode includes "server-side API calls" (proxying Soniox requests through the backend), but it's unclear whether this is just for hiding the API key or if the server does additional processing (e.g., server-side transcription, queuing).

2. **Translation source.** It's unclear whether translation is done by the Soniox API itself or by a separate translation service. The decision about caching translations (D5) implies translation is a distinct API call that costs money, but D3 lists translation as part of the Soniox feature set.

3. **"Stores" in D10.** The document mentions "hooks and stores" but never specifies what "stores" means -- Zustand? React Context? Plain module-level state? This is left open.

4. **Backend technology.** Server mode implies a full backend, but no decisions are recorded about what language, framework, or database it uses.

5. **Scope of "v1".** D14 mentions "v1" for export/import, but there is no explicit definition of what constitutes v1 versus later versions for other features.

6. **IndexedDB in Expo/React Native.** IndexedDB is a browser API. The document doesn't address how storage works when the app is compiled as a native Android app -- whether IndexedDB is available in that context (it is in Expo web, but not in native).

7. **"Sync local data to server" (D14).** The sync mechanism is described at a UX level (button that grays out after use) but not at a technical level -- is it a one-time bulk upload? Does it handle conflicts?

8. **Real-time transcription audio source.** The app is described as running "during phone calls" on mobile, but it's unclear how the app captures call audio. On Android, recording phone calls requires special permissions and is restricted on many devices. The document only mentions "mic transcription."

---

## 4. Questions

1. **Backend stack:** What technology will the server component use? Node/Express? Python/FastAPI? Is there a database preference for server-side storage?

2. **Authentication method:** What auth mechanism will server mode use? OAuth? Email/password? Magic links? A specific auth provider (Firebase Auth, Supabase, etc.)?

3. **Audio capture on mobile:** How will the app capture audio from phone calls on Android? System audio capture is heavily restricted. Is the user expected to use speakerphone with the mic, or is there a different approach?

4. **Soniox API integration pattern:** Does the app open a WebSocket to Soniox for real-time streaming, or use chunked HTTP? How are API keys secured in local mode (just stored in localStorage/memory)?

5. **Translation API:** Is translation handled by Soniox's own API, or by a separate service (e.g., Google Translate, DeepL)? What happens when viewing a transcript in a new language -- is it a real-time Soniox API call or a different translation endpoint?

6. **Offline behavior:** What happens when the app loses connectivity mid-transcription? Is there any buffering or recovery?

7. **Testing strategy:** No decisions about testing approach, frameworks, or coverage expectations.

8. **Styling/design system:** No decisions about CSS approach (StyleSheet, NativeWind/Tailwind, styled-components, etc.) or design system.

9. **Data model specifics:** What does a transcript record look like? What fields does a segment have? How are speakers identified and persisted?

10. **File upload processing:** For pre-recorded files, does processing happen client-side (streaming to Soniox) or server-side? What file formats and size limits are supported?

11. **Multi-user in server mode:** Can multiple users share a server deployment? Is there any notion of shared transcripts or is each user's data isolated?

12. **Android distribution:** APK sideloading? Google Play Store? This affects signing, review processes, and ad SDK integration.

13. **Expo SDK version and managed vs. bare workflow:** Is this using Expo's managed workflow, or will it need to eject for native modules (e.g., audio recording)?