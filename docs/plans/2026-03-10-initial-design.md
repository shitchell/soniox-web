# Soniox Web — Initial Design

## Purpose

A mobile-first (also works on desktop) web app for real-time multilingual
transcription and translation, powered by the Soniox API. Built with Expo
(React Native) so it can also be shipped as an Android app later.

Designed for a Chinese speaker who regularly calls people in different languages
and wants live, speaker-attributed, translatable transcripts.

## Architecture

### Static Frontend + Optional Backend

The app is always a static frontend. The only variable is where the API key
comes from:

- **Self-hosted mode**: a small backend provides the API key via `/api/config`.
  The user never manages a key.
- **GitHub Pages mode**: user enters their API key in Settings, stored in
  localStorage. JS connects directly to Soniox WebSocket.

Detection is automatic: app loads, checks if `/api/config` exists. If yes, use
server key. If 404, prompt user for key.

A fork-specific GitHub deployment pipeline can configure the app to point at a
specific backend server without polluting the main repo.

### Tech Stack

- **Expo (React Native)** — web + Android from one codebase
- **Expo Router** — file-based navigation with tab layout
- **TypeScript** throughout
- **IndexedDB** — local storage for transcripts and translation cache

## Features

### Live Session
- Real-time mic capture via MediaRecorder API
- Audio streamed to Soniox WebSocket API
- Live transcript with speaker diarization (Speaker 1, Speaker 2, etc.)
- Optional real-time translation with configurable target language
- Auto-scroll with manual scroll-back (pauses auto-scroll)

### File Upload
- Drag/drop or file picker for audio/video files
- Sent to Soniox async API for batch transcription
- Progress indicator, transitions to Transcript Viewer when done

### Transcript Viewer
- Language toggle: "Original" or any translated language
- Speaker-labeled segments with timestamps
- Translations cached — switching languages is instant after first translation
- Export: copy to clipboard, download as .txt/.srt

### Settings
- API key field (only in GitHub Pages / no-server mode)
- Default source/target languages
- Display preferences (font size, show/hide timestamps)
- Diarization and translation toggles (defaults)

## Project Structure

```
soniox-web/
├── app/                        # Expo Router pages (thin shells)
│   ├── (tabs)/
│   │   ├── live.tsx            # Live Session tab
│   │   ├── upload.tsx          # File Upload tab
│   │   ├── transcripts.tsx     # Saved Transcripts list
│   │   └── settings.tsx        # Settings tab
│   ├── transcript/[id].tsx     # Transcript Viewer detail
│   └── _layout.tsx             # Root layout + tab nav
│
├── components/                 # Shared UI primitives
│   ├── SpeakerBubble.tsx
│   ├── TranscriptScroller.tsx
│   ├── LanguagePicker.tsx
│   ├── RecordButton.tsx
│   ├── ToggleChip.tsx
│   ├── FileDropZone.tsx
│   └── ...
│
├── features/                   # Feature logic (hooks + state, no UI)
│   ├── live-session/
│   │   ├── useLiveSession.ts
│   │   └── sessionStore.ts
│   ├── file-upload/
│   │   ├── useFileTranscribe.ts
│   │   └── uploadStore.ts
│   ├── transcripts/
│   │   ├── useTranscripts.ts
│   │   └── transcriptStore.ts
│   └── settings/
│       └── useSettings.ts
│
├── services/                   # Soniox API layer
│   ├── soniox-ws.ts            # WebSocket streaming client
│   ├── soniox-async.ts         # File upload / batch API
│   └── soniox-config.ts        # Key management (localStorage vs server)
│
├── storage/                    # Persistence layer
│   └── db.ts                   # IndexedDB for transcripts + translation cache
│
└── types/                      # Shared TypeScript types
    ├── transcript.ts
    └── soniox.ts
```

Pages in `app/` are thin shells — they compose `components/` and wire up
`features/` hooks. Components live outside feature modules so they can be freely
reused or moved between screens.

## Data Model

```typescript
// A single word/token from Soniox
interface Token {
  text: string;
  startMs: number;
  endMs: number;
  speakerId: number;
  confidence: number;
}

// A contiguous block from one speaker
interface Segment {
  speakerId: number;
  startMs: number;
  endMs: number;
  tokens: Token[];
  text: string;                    // assembled from tokens
  language?: string;               // detected source language
}

// Cached translations for a segment
interface TranslationCache {
  [targetLanguage: string]: string; // e.g. { "en": "Hello", "zh": "你好" }
}

// A full transcript (saved to IndexedDB)
interface Transcript {
  id: string;
  title: string;                   // auto-generated or user-editable
  createdAt: number;
  durationMs: number;
  segments: Segment[];
  translations: {
    [segmentIndex: number]: TranslationCache;
  };
  sourceType: 'live' | 'file';
  sourceFileName?: string;         // for file uploads
  settings: {                      // snapshot of settings at recording time
    diarization: boolean;
    sourceLanguages: string[];
    targetLanguage?: string;
  };
}
```

## Translation Caching Strategy

- **During real-time**: if live translation is on, translations arrive alongside
  the original text and are cached immediately.
- **After the fact**: user picks a language from the dropdown, we translate only
  untranslated segments, then cache the results.
- **Switching languages**: instant for any previously translated language — no
  API calls needed.

## Segment Assembly

Soniox streams tokens one at a time. We buffer tokens by speaker ID and create
a new Segment whenever:
- The speaker changes
- There is a significant pause (>2s gap)

## Cost Estimate

At Soniox API real-time rates (~$0.12/hr), ~20 hours/month of calls costs
roughly $2.40/month — compared to $20/month for the Soniox Pro app.
