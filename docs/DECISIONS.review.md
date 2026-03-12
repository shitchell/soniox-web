Here is my structured review of the DECISIONS.md file against the design document.

---

## 1. Internal Consistency

**No contradictions found.** The 14 decisions are mutually consistent. Specifically:

- D6 (dual-mode) aligns with D11 (IndexedDB for local) and D12 (localStorage for settings with config.json defaults)
- D7 (fork-specific deployment) aligns with D12 (config.json defaults updated via CI/CD)
- D9 (shared components) and D10 (thin page shells) reinforce each other and both map to V1
- D14 (export/import in v1) supports the mode-switching described in D6

One minor observation: **D14 maps to V2 (Simplicity)**, but export/import with sync prompts and a persistent sync option is arguably the opposite of simplicity -- it adds UX complexity. A better mapping might be **V3 (No data loss)**, which was defined in the GVP section but is never referenced by any decision. See GVP section below.

---

## 2. Coherency (Rationale Supports Choice)

All decisions have rationale that supports the choice made. A few notes:

- **D5** maps to **V2 (Simplicity)**, but the rationale is about caching to avoid re-translating (cost savings). **G1 (Cheap multilingual transcription)** is already mapped, which is correct. The V2 mapping is defensible (caching is simpler than re-translating each time), but it is a stretch -- the driving motivation was cost, not simplicity.

- **D14** rationale is strong and well-documented with direct quotes. However, as noted above, the V2 mapping is weak. The rationale is entirely about preventing data loss, which aligns with **V3**.

- **D11** maps only to **V2**, but the rationale ("goldilocks between localStorage size limits and sqlite overkill") also supports **G2 (Minimal friction)** since hitting localStorage limits would be a bad user experience.

---

## 3. Gaps (Design Doc Topics Not Tracked in DECISIONS.md)

The following topics appear in the design document but have no corresponding decision:

### 3a. TypeScript throughout
The design doc states "TypeScript throughout" in the Tech Stack section (line 62). No decision captures the choice of TypeScript over plain JavaScript. This is a meaningful architectural choice that affects the entire codebase.

### 3b. Tab-based navigation layout
The design doc specifies a tab layout with four tabs: Live Session, File Upload, Saved Transcripts, and Settings (lines 98-104). The navigation pattern (tabs vs. drawer vs. stack) is an intentional UX decision not tracked.

### 3c. MediaRecorder API for mic capture
The design doc specifies "Real-time mic capture via MediaRecorder API" (line 69). This is a specific technical choice (vs. Web Audio API, getUserMedia raw streams, etc.) that is untracked.

### 3d. WebSocket streaming to Soniox
The design doc specifies "Audio streamed to Soniox WebSocket API" (line 70) and the project structure includes `soniox-ws.ts`. The choice to use WebSocket streaming (vs. chunked HTTP, gRPC-web, etc.) is untracked -- though this may be dictated by the Soniox API rather than being a choice.

### 3e. Segment assembly rules
The design doc specifies segment boundaries: "new Segment whenever the speaker changes or there is a significant pause (>2s gap)" (lines 207-209). The 2-second threshold and segmentation strategy are untracked design decisions.

### 3f. Export formats
The design doc specifies "copy to clipboard, download as .txt/.srt" (line 84) for the Transcript Viewer export. D14 mentions "Export transcripts (JSON)" for data portability. These are two different export features with different purposes, and the .txt/.srt format choice is untracked.

### 3g. Auto-scroll behavior
The design doc specifies "Auto-scroll with manual scroll-back (pauses auto-scroll)" (line 74). This is a specific UX pattern that was presumably discussed but is untracked.

### 3h. Storage abstraction layer interface
The design doc describes `storage/storage.ts` as "Interface -- local and server implement this" (line 136). D6 mentions "a storage abstraction layer that both modes implement" but the decision to use a formal interface/abstraction pattern (vs. conditional logic, dependency injection, etc.) could merit its own decision.

**Assessment:** Items 3a (TypeScript) and 3e (segment assembly rules) are the most significant gaps. The others are arguably implementation details that may not warrant decision tracking, depending on the desired granularity.

---

## 4. GVP Alignment

### 4a. V3 (No data loss) is defined but never mapped

**V3: No data loss** -- "Switching modes or configurations should never strand user data; provide export/import paths" -- is defined in the GVP section but is not referenced by any decision. **D14** (export/import with sync-on-switch) is the obvious candidate and currently maps to G2 and V2 instead. D14's rationale explicitly quotes the user saying it "might feel like data loss," which directly invokes V3.

**Recommendation:** D14 should map to V3 (and possibly replace V2, which is a weak fit as discussed above).

### 4b. D5 mapping to V2 is weak

D5 (translation caching) maps to G1 and V2. The G1 mapping is strong (caching reduces API costs). The V2 (Simplicity) mapping is arguable -- caching adds implementation complexity, even though it simplifies the user experience. If V2 is meant to refer to implementation simplicity, the mapping is wrong. If it refers to UX simplicity (instant language switching), it is defensible but should be clarified.

### 4c. D14 mapping to V2 is incorrect

D14 (export/import in v1) maps to G2 and V2. Adding export/import/sync features to v1 is the opposite of V2 (Simplicity) -- the simpler choice would have been to defer it. The rationale is about preventing data loss (V3) and reducing user friction (G2). **V2 should be replaced with V3.**

### 4d. All other mappings are sound

The remaining mappings check out:
- D1 → G1, V2 (cheapest option that works)
- D2 → G2 (mobile-first for the use case)
- D3 → G1, G2 (features the user needs)
- D4 → G2, V2 (flexible but simple)
- D6 → G2, P2 (user convenience + repo-agnostic)
- D7 → P2 (fork-specific deployment)
- D8 → G3, P1 (Android future + one codebase)
- D9 → V1 (modularity)
- D10 → V1 (modularity)
- D11 → V2 (goldilocks simplicity)
- D12 → P2, G2 (deployment flexibility + user convenience)
- D13 → G2 (minimal friction)

---

## Summary

| Category | Finding | Severity |
|----------|---------|----------|
| GVP | V3 (No data loss) defined but never mapped to any decision | Medium |
| GVP | D14 should map to V3 instead of V2 | Medium |
| GVP | D5 mapping to V2 is weak/ambiguous | Low |
| Gap | TypeScript choice untracked | Medium |
| Gap | Segment assembly rules (2s threshold) untracked | Medium |
| Gap | Tab navigation layout untracked | Low |
| Gap | Export formats (.txt/.srt) untracked | Low |
| Gap | MediaRecorder API choice untracked | Low |
| Gap | Auto-scroll UX pattern untracked | Low |
| Consistency | No contradictions found | -- |
| Coherency | All rationale supports choices made | -- |