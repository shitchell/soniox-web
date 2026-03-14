# Soniox Web — Claude Context

## What Is This

Real-time multilingual transcription and translation web app powered by the
Soniox API. Built with Expo (React Native) for web + future Android. Designed
for a friend who speaks Chinese and needs live call transcription with speaker
diarization and translation.

## Current Status

**Phase 1 (Foundation) is ready for execution. No code has been written yet.**

The repo contains only design docs and an implementation plan. The plan has been
reviewed by 3 agents and all issues have been fixed.

## Required Reading (in order)

Before doing any work, read these files:

1. `docs/plans/2026-03-10-initial-design.md` — Architecture, data model,
   project structure, deployment modes
2. `docs/DECISIONS.md` — All 34 decisions with rationale (D1-D34). Skim the
   GVP section at top, then read decision titles to orient yourself. Read full
   entries when relevant to your current task.
3. `docs/plans/2026-03-12-phase1-foundation.md` — **The implementation plan.**
   7 tasks, TDD-style, with copy-paste-ready code. This is what you're executing.

## Executing Phase 1

Use the `superpowers:subagent-driven-development` skill to execute the plan at
`docs/plans/2026-03-12-phase1-foundation.md`.

The plan has 7 tasks:
1. Project scaffolding (Expo init, Jest config, TS theme, directories)
2. Shared TypeScript types (Token, Segment, Transcript, Soniox wire types)
3. Storage abstraction + IndexedDB implementation
4. Translation provider layer + test providers (pig latin, leet speak)
5. Export formatter layer (JSON, text, markdown, SRT, VTT)
6. Settings store (Zustand with persist middleware)
7. Full test suite run + final commit

Each task follows TDD: write tests first, verify they fail, write implementation,
verify they pass, commit.

## Key Technical Decisions

- **No CSS files** — React Native doesn't support them. Use `styles/theme.ts`
  (TypeScript constants) + `StyleSheet.create()`. See D31.
- **No `@testing-library/jest-native`** — Deprecated. Matchers are built into
  `@testing-library/react-native` v12.4+. See D33.
- **Zustand `|zustand` in transformIgnorePatterns** — Required or Jest chokes
  on ESM imports. See D33.
- **Export formatters have `strict` mode** — `format(transcript, language?,
  strict?)`. Throws `MissingTranslationError` when strict=true and a segment
  lacks a translation. See D32.
- **`exportAll()` includes translations** — Merges from the translations store
  into each transcript. See V3 (no data loss).
- **Settings persist to localStorage** — Via Zustand `persist` middleware. API
  key excluded for security. See D34.
- **`speakerId` is string** — Soniox returns strings, not numbers. See D30.

## Repo Structure (after Phase 1)

```
soniox-web/
├── app/                        # Expo Router pages (thin shells)
│   ├── (tabs)/
│   │   ├── live.tsx
│   │   ├── upload.tsx
│   │   ├── transcripts.tsx
│   │   └── settings.tsx
│   ├── transcript/[id].tsx
│   └── _layout.tsx
├── components/                 # Shared UI primitives (Phase 4)
├── features/
│   └── settings/
│       └── settingsStore.ts    # Zustand store with persist
├── services/                   # Soniox API layer (Phase 2)
├── storage/
│   ├── types.ts                # StorageAdapter interface
│   └── local-web.ts            # IndexedDB implementation
├── translation/
│   ├── types.ts                # TranslationProvider interface
│   ├── registry.ts             # TranslationRegistry class
│   ├── index.ts                # Barrel export + pre-populated registry
│   └── providers/
│       ├── pig-latin.ts        # Test provider
│       └── leet-speak.ts       # Test provider with config fields
├── export/
│   ├── types.ts                # ExportFormatter interface + MissingTranslationError
│   ├── registry.ts             # ExportRegistry class
│   ├── index.ts                # Barrel export + pre-populated registry
│   └── formatters/
│       ├── json.ts
│       ├── text.ts
│       ├── markdown.ts
│       ├── srt.ts
│       └── vtt.ts
├── types/
│   ├── transcript.ts           # Token, Segment, Transcript, TranslationCache
│   └── soniox.ts               # Raw Soniox WebSocket API types
├── styles/
│   └── theme.ts                # Colors, spacing, fonts, radii constants
├── __tests__/                  # Mirrors source structure
├── docs/
│   ├── plans/
│   ├── DECISIONS.md
│   └── DECISIONS.review*.md
└── jest.config.js
```
