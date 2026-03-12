# Phase 1: Foundation — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Scaffold the Expo project, establish shared types, build the storage
abstraction, settings store, translation provider layer, and export formatter
layer — all with full test coverage.

**Architecture:** Bottom-up foundation. Types first, then storage, then
business logic layers (translation, export). Everything is tested in isolation
before any UI or Soniox integration. Local mode only — server mode comes in
Phase 5.

**Tech Stack:** Expo (React Native) with TypeScript, Zustand, Jest +
jest-expo + @testing-library/react-native, Drizzle ORM (Phase 5), plain CSS.

**Decisions:** See `docs/DECISIONS.md` for full rationale. Key refs: D8 (Expo),
D9 (shared components), D10 (thin pages), D11 (IndexedDB), D15 (translation
plugins), D17 (storage abstraction), D22 (Zustand), D23 (plain CSS), D24
(testing strategy), D27 (export formatters).

**Soniox API reference:** Token shapes are from actual Soniox docs. See
`docs/plans/2026-03-10-initial-design.md` for data model. Inconsistencies noted
in Soniox docs (field naming varies between pages) — our types use the WebSocket
API page as canonical, with comments noting discrepancies.

---

## Task 1: Project Scaffolding

**Files:**
- Create: `package.json`, `tsconfig.json`, `app.json`, `app/_layout.tsx`,
  `app/(tabs)/_layout.tsx`, `jest.config.js`, `styles/theme.css`,
  `styles/common.css`, `config.json`
- Create directories: `components/`, `features/`, `services/`, `storage/`,
  `translation/`, `translation/providers/`, `export/`, `export/formatters/`,
  `types/`, `styles/`, `__tests__/`

**Step 1: Initialize Expo project**

```bash
cd ~/code/git/github.com/shitchell/soniox-web
npx create-expo-app@latest . --template tabs --yes
```

If it complains about existing files, we may need to init in a temp dir and
move. Adjust as needed — the goal is a working Expo tabs project.

**Step 2: Install dependencies**

```bash
npx expo install zustand
npm install --save-dev jest-expo @testing-library/react-native @testing-library/jest-native @types/jest
```

**Step 3: Configure Jest**

Create `jest.config.js`:

```javascript
module.exports = {
  preset: 'jest-expo',
  setupFilesAfterSetup: ['@testing-library/jest-native/extend-expect'],
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg)',
  ],
  moduleFileExtensions: ['ts', 'tsx', 'js', 'jsx', 'json'],
  testMatch: ['**/__tests__/**/*.(test|spec).(ts|tsx|js)'],
};
```

**Step 4: Create directory structure**

```bash
mkdir -p components features/live-session features/file-upload features/transcripts features/settings services storage translation/providers export/formatters types styles __tests__/{types,storage,translation,export,features}
```

**Step 5: Create config.json (static defaults)**

Create `public/config.json`:

```json
{
  "defaultMode": "local",
  "defaultServerUrl": ""
}
```

**Step 6: Create CSS foundation**

Create `styles/theme.css`:

```css
:root {
  /* Colors */
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;
  --color-secondary: #64748b;
  --color-background: #ffffff;
  --color-surface: #f8fafc;
  --color-surface-alt: #f1f5f9;
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-text-inverse: #ffffff;
  --color-border: #e2e8f0;
  --color-error: #dc2626;
  --color-success: #16a34a;
  --color-warning: #d97706;

  /* Speaker colors */
  --color-speaker-1: #2563eb;
  --color-speaker-2: #dc2626;
  --color-speaker-3: #16a34a;
  --color-speaker-4: #d97706;
  --color-speaker-5: #7c3aed;

  /* Typography */
  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;
  --font-size-2xl: 1.5rem;

  /* Spacing */
  --space-xs: 0.25rem;
  --space-sm: 0.5rem;
  --space-md: 1rem;
  --space-lg: 1.5rem;
  --space-xl: 2rem;

  /* Borders */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.07);
}
```

Create `styles/common.css`:

```css
@import './theme.css';

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  font-size: var(--font-size-base);
  color: var(--color-text);
  background-color: var(--color-background);
  line-height: 1.5;
}

/* Utility: visually hidden but accessible */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

**Step 7: Verify the project builds and tests run**

```bash
npx expo export --platform web 2>&1 | tail -5
npx jest --passWithNoTests
```

Expected: both commands succeed.

**Step 8: Commit**

```bash
git add -A
git commit -m "feat: scaffold Expo project with directory structure and CSS foundation"
```

---

## Task 2: Shared TypeScript Types

**Files:**
- Create: `types/transcript.ts`, `types/soniox.ts`
- Test: `__tests__/types/transcript.test.ts`

**Step 1: Write type validation tests**

Create `__tests__/types/transcript.test.ts`:

```typescript
import type { Token, Segment, TranslationCache, Transcript } from '../../types/transcript';

describe('Transcript types', () => {
  it('Token has required fields', () => {
    const token: Token = {
      text: 'Hello',
      startMs: 600,
      endMs: 760,
      speakerId: '1',
      confidence: 0.97,
      isFinal: true,
    };
    expect(token.text).toBe('Hello');
    expect(token.startMs).toBe(600);
    expect(token.endMs).toBe(760);
    expect(token.speakerId).toBe('1');
    expect(token.confidence).toBe(0.97);
    expect(token.isFinal).toBe(true);
  });

  it('Token supports optional translation fields', () => {
    const token: Token = {
      text: 'Hola',
      startMs: undefined,
      endMs: undefined,
      speakerId: '1',
      confidence: 0.95,
      isFinal: true,
      translationStatus: 'translation',
      language: 'es',
      sourceLanguage: 'en',
    };
    expect(token.translationStatus).toBe('translation');
    expect(token.sourceLanguage).toBe('en');
  });

  it('Segment assembles tokens into text', () => {
    const segment: Segment = {
      speakerId: '1',
      startMs: 600,
      endMs: 1200,
      tokens: [],
      text: 'Hello world',
      language: 'en',
    };
    expect(segment.text).toBe('Hello world');
  });

  it('Transcript has all required fields', () => {
    const transcript: Transcript = {
      id: 'abc-123',
      title: 'Test call',
      createdAt: Date.now(),
      durationMs: 60000,
      segments: [],
      translations: {},
      sourceType: 'live',
      settings: {
        diarization: true,
        sourceLanguages: ['en', 'zh'],
        targetLanguage: 'en',
      },
    };
    expect(transcript.sourceType).toBe('live');
    expect(transcript.settings.diarization).toBe(true);
  });

  it('Transcript supports file source type', () => {
    const transcript: Transcript = {
      id: 'def-456',
      title: 'Uploaded file',
      createdAt: Date.now(),
      durationMs: 120000,
      segments: [],
      translations: {},
      sourceType: 'file',
      sourceFileName: 'meeting.mp3',
      settings: {
        diarization: false,
        sourceLanguages: ['en'],
      },
    };
    expect(transcript.sourceType).toBe('file');
    expect(transcript.sourceFileName).toBe('meeting.mp3');
  });

  it('TranslationCache maps language codes to translated text', () => {
    const cache: TranslationCache = {
      en: 'Hello',
      zh: '你好',
      es: 'Hola',
    };
    expect(cache['zh']).toBe('你好');
  });
});
```

**Step 2: Run test to verify it fails**

```bash
npx jest __tests__/types/transcript.test.ts -v
```

Expected: FAIL — cannot find module `../../types/transcript`

**Step 3: Write types/transcript.ts**

```typescript
/**
 * A single word/token from Soniox.
 *
 * Field names are camelCase in our app; Soniox wire format uses snake_case.
 * See types/soniox.ts for the raw wire types.
 */
export interface Token {
  text: string;
  startMs: number | undefined;
  endMs: number | undefined;
  speakerId: string;
  confidence: number;
  isFinal: boolean;
  /** "none" | "original" | "translation" — only present when translation is enabled */
  translationStatus?: 'none' | 'original' | 'translation';
  /** Language code of this token (e.g., "en", "zh") */
  language?: string;
  /** Original language before translation — only on translated tokens */
  sourceLanguage?: string;
}

/** A contiguous block of speech from one speaker. */
export interface Segment {
  speakerId: string;
  startMs: number;
  endMs: number;
  tokens: Token[];
  /** Assembled text from tokens */
  text: string;
  /** Detected source language */
  language?: string;
}

/** Cached translations for a segment, keyed by target language code. */
export interface TranslationCache {
  [targetLanguage: string]: string;
}

/** A full transcript, storable in local or server storage. */
export interface Transcript {
  id: string;
  title: string;
  createdAt: number;
  durationMs: number;
  segments: Segment[];
  translations: {
    [segmentIndex: number]: TranslationCache;
  };
  sourceType: 'live' | 'file';
  sourceFileName?: string;
  settings: {
    diarization: boolean;
    sourceLanguages: string[];
    targetLanguage?: string;
  };
}
```

**Step 4: Write types/soniox.ts**

```typescript
/**
 * Raw Soniox WebSocket API types — snake_case, matching the wire format.
 *
 * Source: https://soniox.com/docs/stt/api-reference/websocket-api
 *
 * NOTE: Soniox docs have inconsistencies between pages:
 * - Response timestamp fields: "final_audio_proc_ms" vs "audio_final_proc_ms"
 * - error_code: shown as number (503) on WS page, string on RT page
 * We use the WebSocket API page as canonical.
 */

/** Configuration sent as the first WebSocket message. */
export interface SonioxConfig {
  api_key: string;
  model: string;
  audio_format: string;
  sample_rate?: number;
  num_channels?: number;
  language_hints?: string[];
  language_hints_strict?: boolean;
  context?: SonioxContext;
  enable_speaker_diarization?: boolean;
  enable_language_identification?: boolean;
  enable_endpoint_detection?: boolean;
  max_endpoint_delay_ms?: number;
  client_reference_id?: string;
  translation?: SonioxTranslationConfig;
}

export interface SonioxContext {
  general?: Array<{ key: string; value: string }>;
  text?: string;
  terms?: string[];
  translation_terms?: Array<{ source: string; target: string }>;
}

export type SonioxTranslationConfig =
  | { type: 'one_way'; target_language: string }
  | { type: 'two_way'; language_a: string; language_b: string };

/** A single token in a Soniox response. */
export interface SonioxToken {
  text: string;
  start_ms?: number;
  end_ms?: number;
  confidence: number;
  is_final: boolean;
  speaker?: string;
  translation_status?: 'none' | 'original' | 'translation';
  language?: string;
  source_language?: string;
}

/** A response message from the Soniox WebSocket. */
export interface SonioxResponse {
  tokens: SonioxToken[];
  final_audio_proc_ms?: number;
  total_audio_proc_ms?: number;
  finished?: boolean;
  error_code?: number;
  error_message?: string;
}
```

**Step 5: Run tests to verify they pass**

```bash
npx jest __tests__/types/transcript.test.ts -v
```

Expected: all 6 tests PASS

**Step 6: Commit**

```bash
git add types/ __tests__/types/
git commit -m "feat: add shared TypeScript types for transcripts and Soniox API"
```

---

## Task 3: Storage Abstraction + IndexedDB Implementation

**Files:**
- Create: `storage/types.ts`, `storage/local-web.ts`
- Test: `__tests__/storage/types.test.ts`, `__tests__/storage/local-web.test.ts`

**Step 1: Write storage interface tests**

Create `__tests__/storage/types.test.ts`:

```typescript
import type { StorageAdapter } from '../../storage/types';
import type { Transcript } from '../../types/transcript';

describe('StorageAdapter interface', () => {
  it('can be implemented with all required methods', () => {
    const mock: StorageAdapter = {
      getTranscript: async (_id: string) => null,
      getAllTranscripts: async () => [],
      saveTranscript: async (_t: Transcript) => {},
      deleteTranscript: async (_id: string) => {},
      getTranslation: async (_transcriptId: string, _segmentIndex: number, _lang: string) => null,
      saveTranslation: async (_transcriptId: string, _segmentIndex: number, _lang: string, _text: string) => {},
      getAllTranslations: async (_transcriptId: string) => ({}),
      exportAll: async () => [],
      importAll: async (_transcripts: Transcript[]) => {},
    };
    expect(mock.getTranscript).toBeDefined();
    expect(mock.exportAll).toBeDefined();
    expect(mock.importAll).toBeDefined();
  });
});
```

**Step 2: Run test to verify it fails**

```bash
npx jest __tests__/storage/types.test.ts -v
```

Expected: FAIL — cannot find module

**Step 3: Write storage/types.ts**

```typescript
import type { Transcript, TranslationCache } from '../types/transcript';

/**
 * Storage adapter interface. Implemented by:
 * - local-web.ts (IndexedDB, for web)
 * - local-native.ts (expo-sqlite, for Android — Phase 2+)
 * - server.ts (backend API, for server mode — Phase 5)
 */
export interface StorageAdapter {
  getTranscript(id: string): Promise<Transcript | null>;
  getAllTranscripts(): Promise<Transcript[]>;
  saveTranscript(transcript: Transcript): Promise<void>;
  deleteTranscript(id: string): Promise<void>;

  getTranslation(
    transcriptId: string,
    segmentIndex: number,
    language: string,
  ): Promise<string | null>;
  saveTranslation(
    transcriptId: string,
    segmentIndex: number,
    language: string,
    text: string,
  ): Promise<void>;
  getAllTranslations(
    transcriptId: string,
  ): Promise<{ [segmentIndex: number]: TranslationCache }>;

  exportAll(): Promise<Transcript[]>;
  importAll(transcripts: Transcript[]): Promise<void>;
}
```

**Step 4: Run test to verify it passes**

```bash
npx jest __tests__/storage/types.test.ts -v
```

Expected: PASS

**Step 5: Write IndexedDB implementation tests**

Create `__tests__/storage/local-web.test.ts`:

```typescript
/**
 * NOTE: These tests use fake-indexeddb to simulate IndexedDB in Node.
 * Install: npm install --save-dev fake-indexeddb
 */
import 'fake-indexeddb/auto';
import { IndexedDBStorage } from '../../storage/local-web';
import type { Transcript } from '../../types/transcript';

const makeTranscript = (overrides: Partial<Transcript> = {}): Transcript => ({
  id: 'test-1',
  title: 'Test transcript',
  createdAt: Date.now(),
  durationMs: 60000,
  segments: [
    {
      speakerId: '1',
      startMs: 0,
      endMs: 5000,
      tokens: [],
      text: 'Hello world',
      language: 'en',
    },
  ],
  translations: {},
  sourceType: 'live',
  settings: {
    diarization: true,
    sourceLanguages: ['en'],
  },
  ...overrides,
});

describe('IndexedDBStorage', () => {
  let storage: IndexedDBStorage;

  beforeEach(async () => {
    storage = new IndexedDBStorage();
    await storage.init();
  });

  afterEach(async () => {
    await storage.clear();
  });

  describe('transcripts', () => {
    it('saves and retrieves a transcript', async () => {
      const t = makeTranscript();
      await storage.saveTranscript(t);
      const retrieved = await storage.getTranscript('test-1');
      expect(retrieved).toEqual(t);
    });

    it('returns null for nonexistent transcript', async () => {
      const result = await storage.getTranscript('nonexistent');
      expect(result).toBeNull();
    });

    it('lists all transcripts', async () => {
      await storage.saveTranscript(makeTranscript({ id: 'a' }));
      await storage.saveTranscript(makeTranscript({ id: 'b' }));
      const all = await storage.getAllTranscripts();
      expect(all).toHaveLength(2);
      expect(all.map((t) => t.id).sort()).toEqual(['a', 'b']);
    });

    it('overwrites transcript with same ID', async () => {
      await storage.saveTranscript(makeTranscript({ id: 'a', title: 'v1' }));
      await storage.saveTranscript(makeTranscript({ id: 'a', title: 'v2' }));
      const all = await storage.getAllTranscripts();
      expect(all).toHaveLength(1);
      expect(all[0].title).toBe('v2');
    });

    it('deletes a transcript', async () => {
      await storage.saveTranscript(makeTranscript({ id: 'a' }));
      await storage.deleteTranscript('a');
      const result = await storage.getTranscript('a');
      expect(result).toBeNull();
    });
  });

  describe('translations', () => {
    it('saves and retrieves a translation', async () => {
      await storage.saveTranslation('t1', 0, 'es', 'Hola mundo');
      const result = await storage.getTranslation('t1', 0, 'es');
      expect(result).toBe('Hola mundo');
    });

    it('returns null for nonexistent translation', async () => {
      const result = await storage.getTranslation('t1', 0, 'fr');
      expect(result).toBeNull();
    });

    it('gets all translations for a transcript', async () => {
      await storage.saveTranslation('t1', 0, 'es', 'Hola');
      await storage.saveTranslation('t1', 0, 'fr', 'Bonjour');
      await storage.saveTranslation('t1', 1, 'es', 'Mundo');
      const all = await storage.getAllTranslations('t1');
      expect(all[0]).toEqual({ es: 'Hola', fr: 'Bonjour' });
      expect(all[1]).toEqual({ es: 'Mundo' });
    });
  });

  describe('export/import', () => {
    it('exports all transcripts', async () => {
      await storage.saveTranscript(makeTranscript({ id: 'a' }));
      await storage.saveTranscript(makeTranscript({ id: 'b' }));
      const exported = await storage.exportAll();
      expect(exported).toHaveLength(2);
    });

    it('imports transcripts (overwrites existing)', async () => {
      await storage.saveTranscript(makeTranscript({ id: 'a', title: 'old' }));
      const imports = [
        makeTranscript({ id: 'a', title: 'new' }),
        makeTranscript({ id: 'b', title: 'added' }),
      ];
      await storage.importAll(imports);
      const all = await storage.getAllTranscripts();
      expect(all).toHaveLength(2);
      expect(all.find((t) => t.id === 'a')?.title).toBe('new');
    });
  });
});
```

**Step 6: Install fake-indexeddb and run test to verify it fails**

```bash
npm install --save-dev fake-indexeddb
npx jest __tests__/storage/local-web.test.ts -v
```

Expected: FAIL — cannot find module `../../storage/local-web`

**Step 7: Write storage/local-web.ts**

```typescript
import type { StorageAdapter } from './types';
import type { Transcript, TranslationCache } from '../types/transcript';

const DB_NAME = 'soniox-web';
const DB_VERSION = 1;
const TRANSCRIPTS_STORE = 'transcripts';
const TRANSLATIONS_STORE = 'translations';

export class IndexedDBStorage implements StorageAdapter {
  private db: IDBDatabase | null = null;

  async init(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(DB_NAME, DB_VERSION);

      request.onupgradeneeded = () => {
        const db = request.result;
        if (!db.objectStoreNames.contains(TRANSCRIPTS_STORE)) {
          db.createObjectStore(TRANSCRIPTS_STORE, { keyPath: 'id' });
        }
        if (!db.objectStoreNames.contains(TRANSLATIONS_STORE)) {
          const store = db.createObjectStore(TRANSLATIONS_STORE, {
            keyPath: ['transcriptId', 'segmentIndex', 'language'],
          });
          store.createIndex('byTranscript', 'transcriptId', { unique: false });
        }
      };

      request.onsuccess = () => {
        this.db = request.result;
        resolve();
      };

      request.onerror = () => reject(request.error);
    });
  }

  /** For testing: clear all data and close the database. */
  async clear(): Promise<void> {
    if (this.db) {
      this.db.close();
      this.db = null;
    }
    return new Promise((resolve, reject) => {
      const request = indexedDB.deleteDatabase(DB_NAME);
      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  private getDb(): IDBDatabase {
    if (!this.db) throw new Error('IndexedDBStorage not initialized. Call init() first.');
    return this.db;
  }

  private tx(stores: string | string[], mode: IDBTransactionMode): IDBTransaction {
    return this.getDb().transaction(stores, mode);
  }

  async getTranscript(id: string): Promise<Transcript | null> {
    return new Promise((resolve, reject) => {
      const tx = this.tx(TRANSCRIPTS_STORE, 'readonly');
      const request = tx.objectStore(TRANSCRIPTS_STORE).get(id);
      request.onsuccess = () => resolve(request.result ?? null);
      request.onerror = () => reject(request.error);
    });
  }

  async getAllTranscripts(): Promise<Transcript[]> {
    return new Promise((resolve, reject) => {
      const tx = this.tx(TRANSCRIPTS_STORE, 'readonly');
      const request = tx.objectStore(TRANSCRIPTS_STORE).getAll();
      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async saveTranscript(transcript: Transcript): Promise<void> {
    return new Promise((resolve, reject) => {
      const tx = this.tx(TRANSCRIPTS_STORE, 'readwrite');
      const request = tx.objectStore(TRANSCRIPTS_STORE).put(transcript);
      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async deleteTranscript(id: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const tx = this.tx(TRANSCRIPTS_STORE, 'readwrite');
      const request = tx.objectStore(TRANSCRIPTS_STORE).delete(id);
      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async getTranslation(
    transcriptId: string,
    segmentIndex: number,
    language: string,
  ): Promise<string | null> {
    return new Promise((resolve, reject) => {
      const tx = this.tx(TRANSLATIONS_STORE, 'readonly');
      const request = tx
        .objectStore(TRANSLATIONS_STORE)
        .get([transcriptId, segmentIndex, language]);
      request.onsuccess = () => resolve(request.result?.text ?? null);
      request.onerror = () => reject(request.error);
    });
  }

  async saveTranslation(
    transcriptId: string,
    segmentIndex: number,
    language: string,
    text: string,
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const tx = this.tx(TRANSLATIONS_STORE, 'readwrite');
      const request = tx.objectStore(TRANSLATIONS_STORE).put({
        transcriptId,
        segmentIndex,
        language,
        text,
      });
      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async getAllTranslations(
    transcriptId: string,
  ): Promise<{ [segmentIndex: number]: TranslationCache }> {
    return new Promise((resolve, reject) => {
      const tx = this.tx(TRANSLATIONS_STORE, 'readonly');
      const index = tx.objectStore(TRANSLATIONS_STORE).index('byTranscript');
      const request = index.getAll(transcriptId);
      request.onsuccess = () => {
        const result: { [segmentIndex: number]: TranslationCache } = {};
        for (const row of request.result) {
          if (!result[row.segmentIndex]) {
            result[row.segmentIndex] = {};
          }
          result[row.segmentIndex][row.language] = row.text;
        }
        resolve(result);
      };
      request.onerror = () => reject(request.error);
    });
  }

  async exportAll(): Promise<Transcript[]> {
    return this.getAllTranscripts();
  }

  async importAll(transcripts: Transcript[]): Promise<void> {
    for (const t of transcripts) {
      await this.saveTranscript(t);
    }
  }
}
```

**Step 8: Run tests to verify they pass**

```bash
npx jest __tests__/storage/ -v
```

Expected: all tests PASS

**Step 9: Commit**

```bash
git add storage/ __tests__/storage/
git commit -m "feat: add storage abstraction interface and IndexedDB implementation"
```

---

## Task 4: Translation Provider Layer + Test Providers

**Files:**
- Create: `translation/types.ts`, `translation/registry.ts`,
  `translation/providers/pig-latin.ts`, `translation/providers/leet-speak.ts`
- Test: `__tests__/translation/pig-latin.test.ts`,
  `__tests__/translation/leet-speak.test.ts`,
  `__tests__/translation/registry.test.ts`

**Step 1: Write translation provider interface test**

Create `__tests__/translation/registry.test.ts`:

```typescript
import { TranslationRegistry } from '../../translation/registry';
import type { TranslationProvider } from '../../translation/types';

const mockProvider: TranslationProvider = {
  id: 'mock',
  name: 'Mock Translator',
  configFields: [],
  translate: async (text: string) => `mock:${text}`,
};

describe('TranslationRegistry', () => {
  let registry: TranslationRegistry;

  beforeEach(() => {
    registry = new TranslationRegistry();
  });

  it('registers and retrieves a provider', () => {
    registry.register(mockProvider);
    expect(registry.get('mock')).toBe(mockProvider);
  });

  it('lists all registered providers', () => {
    registry.register(mockProvider);
    registry.register({ ...mockProvider, id: 'mock2', name: 'Mock 2' });
    const all = registry.listAll();
    expect(all).toHaveLength(2);
    expect(all.map((p) => p.id).sort()).toEqual(['mock', 'mock2']);
  });

  it('returns undefined for unregistered provider', () => {
    expect(registry.get('nonexistent')).toBeUndefined();
  });

  it('overwrites provider with same id', () => {
    registry.register(mockProvider);
    const updated = { ...mockProvider, name: 'Updated' };
    registry.register(updated);
    expect(registry.get('mock')?.name).toBe('Updated');
    expect(registry.listAll()).toHaveLength(1);
  });
});
```

**Step 2: Run test to verify it fails**

```bash
npx jest __tests__/translation/registry.test.ts -v
```

Expected: FAIL

**Step 3: Write translation/types.ts**

```typescript
/**
 * Configuration field definition for a translation provider.
 * Each field becomes a form input in Settings when the provider is selected.
 */
export interface ConfigField {
  key: string;
  label: string;
  type: 'text' | 'number' | 'boolean' | 'textarea';
  default?: string | number | boolean;
  description?: string;
}

/**
 * Translation provider interface. All providers (API services, local
 * transforms, LLMs) implement this.
 */
export interface TranslationProvider {
  /** Unique identifier, e.g. "pig-latin", "google-translate" */
  id: string;
  /** Display name shown in Settings dropdown */
  name: string;
  /** Configuration fields this provider requires */
  configFields: ConfigField[];
  /**
   * Translate text from one language to another.
   * @param text - Source text to translate
   * @param targetLanguage - Target language code (e.g., "en", "zh")
   * @param sourceLanguage - Optional source language code
   * @param config - Provider-specific configuration values
   */
  translate(
    text: string,
    targetLanguage: string,
    sourceLanguage?: string,
    config?: Record<string, string | number | boolean>,
  ): Promise<string>;
}
```

**Step 4: Write translation/registry.ts**

```typescript
import type { TranslationProvider } from './types';

export class TranslationRegistry {
  private providers = new Map<string, TranslationProvider>();

  register(provider: TranslationProvider): void {
    this.providers.set(provider.id, provider);
  }

  get(id: string): TranslationProvider | undefined {
    return this.providers.get(id);
  }

  listAll(): TranslationProvider[] {
    return Array.from(this.providers.values());
  }
}
```

**Step 5: Run registry test to verify it passes**

```bash
npx jest __tests__/translation/registry.test.ts -v
```

Expected: PASS

**Step 6: Write pig latin tests**

Create `__tests__/translation/pig-latin.test.ts`:

```typescript
import { pigLatinProvider } from '../../translation/providers/pig-latin';

describe('Pig Latin provider', () => {
  it('has correct metadata', () => {
    expect(pigLatinProvider.id).toBe('pig-latin');
    expect(pigLatinProvider.name).toBe('Pig Latin');
    expect(pigLatinProvider.configFields).toEqual([]);
  });

  it('translates words starting with consonant', async () => {
    const result = await pigLatinProvider.translate('hello', 'pig-latin');
    expect(result).toBe('ellohay');
  });

  it('translates words starting with vowel', async () => {
    const result = await pigLatinProvider.translate('apple', 'pig-latin');
    expect(result).toBe('appleyay');
  });

  it('translates multiple words', async () => {
    const result = await pigLatinProvider.translate('hello world', 'pig-latin');
    expect(result).toBe('ellohay orldway');
  });

  it('handles consonant clusters', async () => {
    const result = await pigLatinProvider.translate('string', 'pig-latin');
    expect(result).toBe('ingstray');
  });

  it('preserves capitalization of first letter', async () => {
    const result = await pigLatinProvider.translate('Hello', 'pig-latin');
    expect(result).toBe('Ellohay');
  });

  it('handles empty string', async () => {
    const result = await pigLatinProvider.translate('', 'pig-latin');
    expect(result).toBe('');
  });

  it('preserves punctuation', async () => {
    const result = await pigLatinProvider.translate('hello!', 'pig-latin');
    expect(result).toBe('ellohay!');
  });
});
```

**Step 7: Run test to verify it fails**

```bash
npx jest __tests__/translation/pig-latin.test.ts -v
```

Expected: FAIL

**Step 8: Write translation/providers/pig-latin.ts**

```typescript
import type { TranslationProvider } from '../types';

const VOWELS = new Set(['a', 'e', 'i', 'o', 'u']);

function toPigLatin(word: string): string {
  if (word.length === 0) return '';

  // Separate trailing punctuation
  const punctMatch = word.match(/^([a-zA-Z]+)([^a-zA-Z]*)$/);
  if (!punctMatch) return word;

  const [, letters, punctuation] = punctMatch;
  const isCapitalized = letters[0] === letters[0].toUpperCase();
  const lower = letters.toLowerCase();

  let result: string;
  if (VOWELS.has(lower[0])) {
    result = lower + 'yay';
  } else {
    // Find the consonant cluster
    let i = 0;
    while (i < lower.length && !VOWELS.has(lower[i])) i++;
    result = lower.slice(i) + lower.slice(0, i) + 'ay';
  }

  if (isCapitalized) {
    result = result[0].toUpperCase() + result.slice(1);
  }

  return result + punctuation;
}

export const pigLatinProvider: TranslationProvider = {
  id: 'pig-latin',
  name: 'Pig Latin',
  configFields: [],
  async translate(text: string): Promise<string> {
    if (!text) return '';
    return text.split(' ').map(toPigLatin).join(' ');
  },
};
```

**Step 9: Run pig latin tests**

```bash
npx jest __tests__/translation/pig-latin.test.ts -v
```

Expected: PASS

**Step 10: Write leet speak tests**

Create `__tests__/translation/leet-speak.test.ts`:

```typescript
import { leetSpeakProvider } from '../../translation/providers/leet-speak';

describe('Leet Speak provider', () => {
  it('has correct metadata', () => {
    expect(leetSpeakProvider.id).toBe('leet-speak');
    expect(leetSpeakProvider.name).toBe('1337 Speak');
  });

  it('has configurable fields', () => {
    const fields = leetSpeakProvider.configFields;
    const skipField = fields.find((f) => f.key === 'skipLetters');
    expect(skipField).toBeDefined();
    expect(skipField?.type).toBe('text');

    const customField = fields.find((f) => f.key === 'customReplacements');
    expect(customField).toBeDefined();
    expect(customField?.type).toBe('textarea');
  });

  it('applies default replacements', async () => {
    const result = await leetSpeakProvider.translate('leet', '1337');
    expect(result).toBe('1337');
  });

  it('replaces common letters by default', async () => {
    const result = await leetSpeakProvider.translate('hello', '1337');
    // h->h, e->3, l->1, l->1, o->0
    expect(result).toBe('h3110');
  });

  it('skips letters specified in config', async () => {
    const result = await leetSpeakProvider.translate('eat', '1337', undefined, {
      skipLetters: 'e',
    });
    // e stays, a->4, t->7
    expect(result).toBe('e47');
  });

  it('applies custom replacements from config', async () => {
    const result = await leetSpeakProvider.translate('hello', '1337', undefined, {
      customReplacements: 'h=|-|',
    });
    expect(result).toBe('|-|3110');
  });

  it('handles empty string', async () => {
    const result = await leetSpeakProvider.translate('', '1337');
    expect(result).toBe('');
  });

  it('preserves characters without replacements', async () => {
    const result = await leetSpeakProvider.translate('xyz', '1337');
    expect(result).toBe('xyz');
  });
});
```

**Step 11: Run test to verify it fails**

```bash
npx jest __tests__/translation/leet-speak.test.ts -v
```

Expected: FAIL

**Step 12: Write translation/providers/leet-speak.ts**

```typescript
import type { TranslationProvider } from '../types';

const DEFAULT_REPLACEMENTS: Record<string, string> = {
  a: '4',
  e: '3',
  i: '1',
  l: '1',
  o: '0',
  s: '5',
  t: '7',
};

function parseCustomReplacements(input: string): Record<string, string> {
  const result: Record<string, string> = {};
  if (!input) return result;
  for (const pair of input.split(',')) {
    const [from, to] = pair.trim().split('=');
    if (from && to) {
      result[from.toLowerCase()] = to;
    }
  }
  return result;
}

function parseSkipLetters(input: string): Set<string> {
  if (!input) return new Set();
  return new Set(input.toLowerCase().split('').filter((c) => c.trim()));
}

export const leetSpeakProvider: TranslationProvider = {
  id: 'leet-speak',
  name: '1337 Speak',
  configFields: [
    {
      key: 'skipLetters',
      label: 'Letters to skip',
      type: 'text',
      default: '',
      description: 'Letters that should NOT be replaced (e.g., "ae" skips a and e)',
    },
    {
      key: 'customReplacements',
      label: 'Custom replacements',
      type: 'textarea',
      default: '',
      description: 'Comma-separated replacements (e.g., "h=|-|,w=\\/\\/"). Overrides defaults.',
    },
  ],
  async translate(
    text: string,
    _targetLanguage: string,
    _sourceLanguage?: string,
    config?: Record<string, string | number | boolean>,
  ): Promise<string> {
    if (!text) return '';

    const skipSet = parseSkipLetters(String(config?.skipLetters ?? ''));
    const customMap = parseCustomReplacements(String(config?.customReplacements ?? ''));
    const replacements = { ...DEFAULT_REPLACEMENTS, ...customMap };

    return text
      .split('')
      .map((char) => {
        const lower = char.toLowerCase();
        if (skipSet.has(lower)) return char;
        return replacements[lower] ?? char;
      })
      .join('');
  },
};
```

**Step 13: Run all translation tests**

```bash
npx jest __tests__/translation/ -v
```

Expected: all tests PASS

**Step 14: Commit**

```bash
git add translation/ __tests__/translation/
git commit -m "feat: add translation provider layer with pig latin and 1337 speak providers"
```

---

## Task 5: Export Formatter Layer

**Files:**
- Create: `export/types.ts`, `export/registry.ts`,
  `export/formatters/json.ts`, `export/formatters/text.ts`,
  `export/formatters/markdown.ts`, `export/formatters/srt.ts`,
  `export/formatters/vtt.ts`
- Test: `__tests__/export/registry.test.ts`,
  `__tests__/export/formatters.test.ts`

**Step 1: Write export formatter tests**

Create `__tests__/export/formatters.test.ts`:

```typescript
import { jsonFormatter } from '../../export/formatters/json';
import { textFormatter } from '../../export/formatters/text';
import { markdownFormatter } from '../../export/formatters/markdown';
import { srtFormatter } from '../../export/formatters/srt';
import { vttFormatter } from '../../export/formatters/vtt';
import type { Transcript } from '../../types/transcript';

const transcript: Transcript = {
  id: 'test-1',
  title: 'Test Call',
  createdAt: 1710000000000,
  durationMs: 10000,
  segments: [
    {
      speakerId: '1',
      startMs: 0,
      endMs: 3000,
      tokens: [],
      text: 'Hello, how are you?',
      language: 'en',
    },
    {
      speakerId: '2',
      startMs: 3500,
      endMs: 7000,
      tokens: [],
      text: 'I am doing great, thanks!',
      language: 'en',
    },
  ],
  translations: {},
  sourceType: 'live',
  settings: {
    diarization: true,
    sourceLanguages: ['en'],
  },
};

describe('JSON formatter', () => {
  it('has correct metadata', () => {
    expect(jsonFormatter.id).toBe('json');
    expect(jsonFormatter.extension).toBe('.json');
  });

  it('outputs valid JSON matching the transcript', () => {
    const output = jsonFormatter.format(transcript);
    const parsed = JSON.parse(output);
    expect(parsed.id).toBe('test-1');
    expect(parsed.segments).toHaveLength(2);
  });
});

describe('Text formatter', () => {
  it('has correct metadata', () => {
    expect(textFormatter.id).toBe('text');
    expect(textFormatter.extension).toBe('.txt');
  });

  it('outputs speaker-labeled lines', () => {
    const output = textFormatter.format(transcript);
    expect(output).toContain('Speaker 1: Hello, how are you?');
    expect(output).toContain('Speaker 2: I am doing great, thanks!');
  });
});

describe('Markdown formatter', () => {
  it('has correct metadata', () => {
    expect(markdownFormatter.id).toBe('markdown');
    expect(markdownFormatter.extension).toBe('.md');
  });

  it('includes title as heading', () => {
    const output = markdownFormatter.format(transcript);
    expect(output).toContain('# Test Call');
  });

  it('includes speaker-labeled lines', () => {
    const output = markdownFormatter.format(transcript);
    expect(output).toContain('**Speaker 1:** Hello, how are you?');
    expect(output).toContain('**Speaker 2:** I am doing great, thanks!');
  });
});

describe('SRT formatter', () => {
  it('has correct metadata', () => {
    expect(srtFormatter.id).toBe('srt');
    expect(srtFormatter.extension).toBe('.srt');
  });

  it('outputs numbered entries with SRT timestamps', () => {
    const output = srtFormatter.format(transcript);
    expect(output).toContain('1\n00:00:00,000 --> 00:00:03,000');
    expect(output).toContain('2\n00:00:03,500 --> 00:00:07,000');
    expect(output).toContain('Speaker 1: Hello, how are you?');
  });
});

describe('VTT formatter', () => {
  it('has correct metadata', () => {
    expect(vttFormatter.id).toBe('vtt');
    expect(vttFormatter.extension).toBe('.vtt');
  });

  it('starts with WEBVTT header', () => {
    const output = vttFormatter.format(transcript);
    expect(output.startsWith('WEBVTT')).toBe(true);
  });

  it('outputs entries with VTT timestamps', () => {
    const output = vttFormatter.format(transcript);
    expect(output).toContain('00:00:00.000 --> 00:00:03.000');
    expect(output).toContain('Speaker 1: Hello, how are you?');
  });
});
```

**Step 2: Run test to verify it fails**

```bash
npx jest __tests__/export/formatters.test.ts -v
```

Expected: FAIL

**Step 3: Write export/types.ts**

```typescript
import type { Transcript } from '../types/transcript';

export interface ExportFormatter {
  /** Unique identifier, e.g. "json", "srt" */
  id: string;
  /** Display name shown in export dropdown */
  name: string;
  /** File extension including dot, e.g. ".json", ".srt" */
  extension: string;
  /** MIME type for download, e.g. "application/json" */
  mimeType: string;
  /** Format a transcript into a string. */
  format(transcript: Transcript, language?: string): string;
}
```

**Step 4: Write export/registry.ts**

```typescript
import type { ExportFormatter } from './types';

export class ExportRegistry {
  private formatters = new Map<string, ExportFormatter>();

  register(formatter: ExportFormatter): void {
    this.formatters.set(formatter.id, formatter);
  }

  get(id: string): ExportFormatter | undefined {
    return this.formatters.get(id);
  }

  listAll(): ExportFormatter[] {
    return Array.from(this.formatters.values());
  }
}
```

**Step 5: Write all five formatters**

`export/formatters/json.ts`:

```typescript
import type { ExportFormatter } from '../types';

export const jsonFormatter: ExportFormatter = {
  id: 'json',
  name: 'JSON',
  extension: '.json',
  mimeType: 'application/json',
  format(transcript) {
    return JSON.stringify(transcript, null, 2);
  },
};
```

`export/formatters/text.ts`:

```typescript
import type { ExportFormatter } from '../types';

export const textFormatter: ExportFormatter = {
  id: 'text',
  name: 'Plain Text',
  extension: '.txt',
  mimeType: 'text/plain',
  format(transcript) {
    return transcript.segments
      .map((seg) => `Speaker ${seg.speakerId}: ${seg.text}`)
      .join('\n');
  },
};
```

`export/formatters/markdown.ts`:

```typescript
import type { ExportFormatter } from '../types';

export const markdownFormatter: ExportFormatter = {
  id: 'markdown',
  name: 'Markdown',
  extension: '.md',
  mimeType: 'text/markdown',
  format(transcript) {
    const date = new Date(transcript.createdAt).toISOString().split('T')[0];
    const lines = [
      `# ${transcript.title}`,
      '',
      `*${date} \u2014 ${formatDuration(transcript.durationMs)}*`,
      '',
    ];
    for (const seg of transcript.segments) {
      lines.push(`**Speaker ${seg.speakerId}:** ${seg.text}`);
      lines.push('');
    }
    return lines.join('\n').trim();
  },
};

function formatDuration(ms: number): string {
  const totalSec = Math.floor(ms / 1000);
  const min = Math.floor(totalSec / 60);
  const sec = totalSec % 60;
  return `${min}m ${sec}s`;
}
```

`export/formatters/srt.ts`:

```typescript
import type { ExportFormatter } from '../types';

function msToSrt(ms: number): string {
  const hours = Math.floor(ms / 3600000);
  const minutes = Math.floor((ms % 3600000) / 60000);
  const seconds = Math.floor((ms % 60000) / 1000);
  const millis = ms % 1000;
  return (
    String(hours).padStart(2, '0') +
    ':' +
    String(minutes).padStart(2, '0') +
    ':' +
    String(seconds).padStart(2, '0') +
    ',' +
    String(millis).padStart(3, '0')
  );
}

export const srtFormatter: ExportFormatter = {
  id: 'srt',
  name: 'SRT (Subtitles)',
  extension: '.srt',
  mimeType: 'text/srt',
  format(transcript) {
    return transcript.segments
      .map((seg, i) => {
        const start = msToSrt(seg.startMs);
        const end = msToSrt(seg.endMs);
        return `${i + 1}\n${start} --> ${end}\nSpeaker ${seg.speakerId}: ${seg.text}`;
      })
      .join('\n\n');
  },
};
```

`export/formatters/vtt.ts`:

```typescript
import type { ExportFormatter } from '../types';

function msToVtt(ms: number): string {
  const hours = Math.floor(ms / 3600000);
  const minutes = Math.floor((ms % 3600000) / 60000);
  const seconds = Math.floor((ms % 60000) / 1000);
  const millis = ms % 1000;
  return (
    String(hours).padStart(2, '0') +
    ':' +
    String(minutes).padStart(2, '0') +
    ':' +
    String(seconds).padStart(2, '0') +
    '.' +
    String(millis).padStart(3, '0')
  );
}

export const vttFormatter: ExportFormatter = {
  id: 'vtt',
  name: 'WebVTT (Subtitles)',
  extension: '.vtt',
  mimeType: 'text/vtt',
  format(transcript) {
    const cues = transcript.segments
      .map((seg) => {
        const start = msToVtt(seg.startMs);
        const end = msToVtt(seg.endMs);
        return `${start} --> ${end}\nSpeaker ${seg.speakerId}: ${seg.text}`;
      })
      .join('\n\n');
    return `WEBVTT\n\n${cues}`;
  },
};
```

**Step 6: Run all export tests**

```bash
npx jest __tests__/export/ -v
```

Expected: all tests PASS

**Step 7: Write and run registry test**

Create `__tests__/export/registry.test.ts`:

```typescript
import { ExportRegistry } from '../../export/registry';
import { jsonFormatter } from '../../export/formatters/json';
import { textFormatter } from '../../export/formatters/text';

describe('ExportRegistry', () => {
  it('registers and retrieves formatters', () => {
    const registry = new ExportRegistry();
    registry.register(jsonFormatter);
    registry.register(textFormatter);
    expect(registry.get('json')).toBe(jsonFormatter);
    expect(registry.get('text')).toBe(textFormatter);
    expect(registry.listAll()).toHaveLength(2);
  });
});
```

```bash
npx jest __tests__/export/registry.test.ts -v
```

Expected: PASS

**Step 8: Commit**

```bash
git add export/ __tests__/export/
git commit -m "feat: add export formatter layer with json, text, markdown, srt, and vtt formatters"
```

---

## Task 6: Settings Store (Zustand)

**Files:**
- Create: `features/settings/settingsStore.ts`
- Test: `__tests__/features/settings/settingsStore.test.ts`

**Step 1: Write settings store tests**

Create `__tests__/features/settings/settingsStore.test.ts`:

```typescript
import { useSettingsStore } from '../../../features/settings/settingsStore';

describe('settingsStore', () => {
  beforeEach(() => {
    useSettingsStore.getState().reset();
  });

  it('defaults to local mode', () => {
    const state = useSettingsStore.getState();
    expect(state.mode).toBe('local');
    expect(state.serverUrl).toBe('');
  });

  it('sets mode to server with URL', () => {
    useSettingsStore.getState().setMode('server', 'https://example.com');
    const state = useSettingsStore.getState();
    expect(state.mode).toBe('server');
    expect(state.serverUrl).toBe('https://example.com');
  });

  it('stores Soniox API key (local mode)', () => {
    useSettingsStore.getState().setSonioxApiKey('test-key-123');
    expect(useSettingsStore.getState().sonioxApiKey).toBe('test-key-123');
  });

  it('toggles diarization default', () => {
    expect(useSettingsStore.getState().diarizationEnabled).toBe(true);
    useSettingsStore.getState().setDiarizationEnabled(false);
    expect(useSettingsStore.getState().diarizationEnabled).toBe(false);
  });

  it('stores translation provider selection and per-provider config', () => {
    useSettingsStore.getState().setTranslationProvider('leet-speak');
    useSettingsStore.getState().setProviderConfig('leet-speak', {
      skipLetters: 'ae',
      customReplacements: 'h=|-|',
    });
    expect(useSettingsStore.getState().translationProviderId).toBe('leet-speak');
    expect(useSettingsStore.getState().providerConfigs['leet-speak']).toEqual({
      skipLetters: 'ae',
      customReplacements: 'h=|-|',
    });
  });

  it('remembers config when switching between providers', () => {
    useSettingsStore.getState().setProviderConfig('leet-speak', { skipLetters: 'a' });
    useSettingsStore.getState().setProviderConfig('pig-latin', {});
    useSettingsStore.getState().setTranslationProvider('pig-latin');

    // Switch back — config should still be there
    useSettingsStore.getState().setTranslationProvider('leet-speak');
    expect(useSettingsStore.getState().providerConfigs['leet-speak']).toEqual({
      skipLetters: 'a',
    });
  });

  it('stores default languages', () => {
    useSettingsStore.getState().setSourceLanguages(['en', 'zh']);
    useSettingsStore.getState().setTargetLanguage('en');
    const state = useSettingsStore.getState();
    expect(state.sourceLanguages).toEqual(['en', 'zh']);
    expect(state.targetLanguage).toBe('en');
  });

  it('stores display preferences', () => {
    useSettingsStore.getState().setFontSize('lg');
    useSettingsStore.getState().setShowTimestamps(false);
    const state = useSettingsStore.getState();
    expect(state.fontSize).toBe('lg');
    expect(state.showTimestamps).toBe(false);
  });
});
```

**Step 2: Run test to verify it fails**

```bash
npx jest __tests__/features/settings/ -v
```

Expected: FAIL

**Step 3: Write features/settings/settingsStore.ts**

```typescript
import { create } from 'zustand';

export type AppMode = 'local' | 'server';
export type FontSize = 'sm' | 'base' | 'lg' | 'xl';

interface ProviderConfigs {
  [providerId: string]: Record<string, string | number | boolean>;
}

interface SettingsState {
  // Mode
  mode: AppMode;
  serverUrl: string;

  // Soniox
  sonioxApiKey: string;
  diarizationEnabled: boolean;
  liveTranslationEnabled: boolean;

  // Languages
  sourceLanguages: string[];
  targetLanguage: string;

  // Translation provider
  translationProviderId: string;
  providerConfigs: ProviderConfigs;

  // Display
  fontSize: FontSize;
  showTimestamps: boolean;

  // Actions
  setMode: (mode: AppMode, serverUrl?: string) => void;
  setSonioxApiKey: (key: string) => void;
  setDiarizationEnabled: (enabled: boolean) => void;
  setLiveTranslationEnabled: (enabled: boolean) => void;
  setSourceLanguages: (languages: string[]) => void;
  setTargetLanguage: (language: string) => void;
  setTranslationProvider: (providerId: string) => void;
  setProviderConfig: (
    providerId: string,
    config: Record<string, string | number | boolean>,
  ) => void;
  setFontSize: (size: FontSize) => void;
  setShowTimestamps: (show: boolean) => void;
  reset: () => void;
}

const initialState = {
  mode: 'local' as AppMode,
  serverUrl: '',
  sonioxApiKey: '',
  diarizationEnabled: true,
  liveTranslationEnabled: false,
  sourceLanguages: [] as string[],
  targetLanguage: '',
  translationProviderId: '',
  providerConfigs: {} as ProviderConfigs,
  fontSize: 'base' as FontSize,
  showTimestamps: true,
};

export const useSettingsStore = create<SettingsState>((set) => ({
  ...initialState,

  setMode: (mode, serverUrl) =>
    set({ mode, ...(serverUrl !== undefined ? { serverUrl } : {}) }),

  setSonioxApiKey: (sonioxApiKey) => set({ sonioxApiKey }),

  setDiarizationEnabled: (diarizationEnabled) => set({ diarizationEnabled }),

  setLiveTranslationEnabled: (liveTranslationEnabled) =>
    set({ liveTranslationEnabled }),

  setSourceLanguages: (sourceLanguages) => set({ sourceLanguages }),

  setTargetLanguage: (targetLanguage) => set({ targetLanguage }),

  setTranslationProvider: (translationProviderId) =>
    set({ translationProviderId }),

  setProviderConfig: (providerId, config) =>
    set((state) => ({
      providerConfigs: { ...state.providerConfigs, [providerId]: config },
    })),

  setFontSize: (fontSize) => set({ fontSize }),

  setShowTimestamps: (showTimestamps) => set({ showTimestamps }),

  reset: () => set(initialState),
}));
```

**Step 4: Run tests to verify they pass**

```bash
npx jest __tests__/features/settings/ -v
```

Expected: all tests PASS

**Step 5: Commit**

```bash
git add features/settings/ __tests__/features/settings/
git commit -m "feat: add Zustand settings store with mode, language, translation provider, and display prefs"
```

---

## Task 7: Run Full Test Suite + Final Phase 1 Commit

**Step 1: Run all tests**

```bash
npx jest --verbose
```

Expected: all tests PASS (types, storage, translation, export, settings)

**Step 2: Verify build still works**

```bash
npx expo export --platform web 2>&1 | tail -5
```

Expected: success

**Step 3: Final commit if any files were missed**

```bash
git status
# If any unstaged files:
git add -A
git commit -m "chore: phase 1 foundation complete — types, storage, translation, export, settings"
```

**Step 4: Push**

```bash
git push
```

---

## Remaining Phases (separate plan documents)

### Phase 2: Soniox Integration
- Soniox WebSocket client (token streaming, config, error handling)
- Soniox async client (file upload API)
- Segment assembly logic (token → segment with speaker changes and pause detection)
- Token converter (Soniox snake_case → our camelCase types)

### Phase 3: Feature Logic
- Transcript store (Zustand + storage adapter CRUD)
- Live session feature (audio capture → Soniox → segment assembly → storage)
- File upload feature (file picker → Soniox async → storage)

### Phase 4: UI
- CSS page styles
- Shared components (SpeakerBubble, TranscriptScroller, LanguagePicker,
  RecordButton, ToggleChip, FileDropZone)
- Tab layout and all pages (thin shells composing components + hooks)
- Login screen with "use locally" escape hatch

### Phase 5: Backend
- Node.js/Express server scaffolding
- Drizzle ORM setup (schema, migrations for SQLite + Postgres)
- Auth endpoints (register, login, JWT)
- Transcript CRUD API endpoints
- Translation cache API endpoints
- Share link endpoints
- Server storage adapter (frontend implementation calling backend API)

### Phase 6: Integration & Polish
- Mode switching with sync prompts (D25 sync semantics)
- config.json loading and defaults
- Data portability (export/import UI)
- Share links UI
- Responsive CSS polish
- Error handling and edge cases
