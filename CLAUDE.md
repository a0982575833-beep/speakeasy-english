# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

SpeakEasy — an AI English speaking coach PWA for a beginner-level English learner (Roy, a TMGM client manager in Taipei). The AI tutor persona is "NingNing". The entire application is a **single self-contained file: `index.html`** (CSS + markup + vanilla JS, no framework, no dependencies, no build step).

## Development

There is no package.json, build, lint, or test tooling. Development is: edit `index.html`, reload the browser.

```bash
# Serve locally (required — Web Speech API mic only works on localhost or HTTPS)
python -m http.server 8080
```

- **Deploy**: GitHub Pages serves the `main` branch at https://a0982575833-beep.github.io/speakeasy-english/ — pushing to `main` is deploying.
- **Which file to edit**: `index.html` is the canonical deployed app. `SpeakEasy_2026-04-03.html` is a stale source copy (lags behind; last synced at v2.3) — do not edit it expecting changes to ship.
- **Service worker**: `service-worker.js` uses cache-first for assets (`CACHE_NAME = 'speakeasy-v1'`) and network-only for `googleapis.com`. Bump the cache name if cached assets must be invalidated for existing installs.
- **Commit convention**: version-prefixed messages, e.g. `v2.4: Add VoxCPM2 local TTS integration...`. Bug fixes use `Fix: ...`.
- **Testing** is manual, in-browser. Voice features require Chrome or Safari (Firefox has limited Web Speech API support).

## Architecture of index.html

One `<script>` block (~line 335 onward) organized by `/* ===== SECTION ===== */` comment banners. Key conventions:

- `$` / `$$` — `getElementById` / `querySelectorAll` shorthands.
- `S` — the global state object, hydrated from localStorage at load. All keys use the `K = 'se_'` prefix; persist via `sv(key, value)`.
- Five tabs (`chat`, `daily`, `shadow`, `words`, `stats`) are absolutely-positioned divs toggled by `switchTab()`; the app is a fixed-height mobile shell with no routing.
- All user-facing strings are bilingual: English + Traditional Chinese (e.g. toasts like `'URL updated'`, `'Gentle — major errors only 溫和模式'`). Follow this in any new UI text.

### Gemini API integration

- Calls `gemini-2.5-flash:generateContent` directly from the browser with a **user-supplied API key** stored in localStorage (`se_key`) — there is no backend. Free-tier keys are expected (15 RPM), so 429/quota errors get friendly bilingual messages in `gem()`.
- Chat uses SSE streaming (`streamGenerateContent?alt=sse`) with a non-streaming `gem()` fallback; context window is capped at the last 20 conversation turns.
- The tutor persona and correction behavior live in `getSysPrompt()` (dynamic — it embeds the current gentle/strict mode).

### The emoji reply protocol (critical)

The system prompt instructs the model to embed structured data in replies using emoji markers, which `dispMsg()` parses with regexes:

- `📝 ...` — Chinese translation line
- `⚠️CORRECTION:` block with `❌` (error) / `✅` (fix) / `💡` (why, EN) / `📝` (why, CN) — rendered as correction cards
- `📚VOCAB: word (pos) = Chinese | example` — rendered as save-word buttons

If you change the system prompt's output format, you **must** update the matching parse/strip regexes in `sendMsg()` (strips markers before TTS) and `dispMsg()` (extracts cards), and vice versa.

### TTS: 3-tier fallback in speak()

1. **VoxCPM2** — optional local TTS server (default `http://localhost:8809`, configurable in settings; availability auto-checked every 30s)
2. **Gemini TTS** — `gemini-2.5-flash-preview-tts`, voice "Kore"; returns raw PCM converted via `pcmToWav()`
3. **Browser** `speechSynthesis`

Playback rate comes from the user's speed setting (0.7x / 1.0x / 1.3x). Voice *input* uses the Web Speech API (`SpeechRecognition`), hold-to-talk with auto-restart while held, with an EN/中 language toggle.

### Other features

- **Daily scenarios**: 3 per day generated via Gemini (Work/Social/Daily categories), cached in localStorage by date.
- **Shadowing**: fixed built-in sentence list; listen → repeat → compare.
- **Gamification**: XP (`addXP`), day streak, and per-mistake counters in `S.stats`.

## Repo Files

- `index.html` — the app (deployed)
- `manifest.json`, `service-worker.js`, `icons/` — PWA install/offline support
- `README_2026-04-03.md` — user-facing bilingual usage guide
- `Zeabur部署指南_2026-04-03.md` — alternative Zeabur deployment guide
- `進度_2026-04-03.md` — progress/status notes (outdated: says v1.5; git history is at v2.4+ — trust `git log` for current feature state)
