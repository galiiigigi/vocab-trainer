# Canadian English Vocabulary Trainer

Single-file app: `vocab-trainer.html` (~3440 lines). PWA manifest & icons generated inline (no external files).

## Files
- `vocab-trainer.html` — the entire app (HTML/CSS/JS + inline PWA manifest/icons)
- `CLAUDE.md` — this file (context for Claude)

## Architecture
- Single HTML/CSS/JS file, no build tools
- localStorage persistence (`vocabApp` key)
- Gemini 2.0 Flash API for AI features
- Google Cloud TTS + browser TTS fallback
- GitHub Gist cloud sync via Personal Access Token + REST API
- SM-2 Spaced Repetition System
- All words are user-imported (WORDS_RAW is empty)
- PWA manifest & icons inlined (canvas-generated data URLs, blob manifest)
- No service worker (no offline cache) — keeps single-file simplicity

## Key State
- `state.customWords[]` — all words (no built-in words)
- `state.wordProgress{}` — SRS data per word
- `state.settings` — API keys, theme, voice, thresholds
- `state.ttsUsage` — monthly character tracking
- `state.dailyActivity{}` — heatmap data
- Sync format: v3 with selective export (words/progress/settings/stats/apiKeys) — clipboard + .json file + GitHub Gist
- Gist sync uses 5 save slots (`vocab-slot-1.json` through `vocab-slot-5.json`) in a single gist; old `vocab-sync.json` shown as loadable "Legacy" entry
- API key stripping: `SENSITIVE_KEYS` (`apiKey`, `googleTtsKey`, `githubGistToken`) excluded from sync by default; opt-in via "API Keys" checkbox

## API Calls
- `callGemini(messages, systemInstruction)` — messages use `{role:'user', text:prompt}` format (NOT `{parts:[{text}]}`)
- `callGeminiWithAudio(audioBase64, mimeType, textPrompt, systemInstruction)` — for voice chat
- `googleSpeak(text, rate, isTest)` — Google Cloud TTS with char tracking
- `exportToGist()` / `importFromGist()` / `loadGist(gistId, fileName)` — GitHub Gist cloud sync via REST API
- Monthly char limit default: 900K (of 1M free tier)

## Major Sections (by line ~)
- **App Log** (~242): Console override, showAppLogs(), copyAppLogs()
- **Word DB** (~300): Empty WORDS_RAW, CATEGORIES (16 cats), WORDS build with phonetic field
- **State & Migrations** (~312): localStorage, theme, TTS, SRS settings
- **SRS** (~349): SM-2 algorithm, getDueWords, getSessionWords
- **TTS** (~469): speak(), googleSpeak(), fetchGoogleVoices(), buildVoiceOptions(), testGoogleTTS()
- **API Key Tests** (~606): testGeminiKey() (simple prompt), testTtsKey() (list voices), testGistToken() (GET /user)
- **Navigation** (~631): goTo(), renderScreen() — screens: home, quiz, cards, voice, chat, pretest, addwords, ear, words, stats, settings
- **Home** (~663): Stat boxes (clickable→stats), due words, empty state onboarding, AI feature buttons
- **Quiz** (~703): Timer, answerQuiz, dontKnowQuiz (~847), skipQuiz, markKnownQuiz, endSessionEarly
- **Flashcards** (~941): Flip, rate, skip, markKnown, endSessionEarly
- **Voice Test** (~1056): Mic recognition, fuzzyMatch, endSessionEarly
- **Mark Known** (~1215): markWordAsMastered, markKnownQuiz/Flashcard/Voice
- **Word List** (~1260): Split render (search input fix), updateWordListResults()
- **Duplicates** (~1308): AI-powered findDuplicateWords, per-word delete buttons (~1395), combine/merge
- **Word Detail** (~1440): Modal with phonetic, grammar, SRS info, delete button
- **Stats** (~1478): Stat grid, heatmap, category progress bars
- **Settings** (~1554): Theme, API keys (Gemini + GitHub Gist token), TTS config, voice dropdown, RT thresholds, SRS intervals, word pack import, sync, danger zone, logs
- **API Setup Guides** (~1940): showGeminiSetupGuide, showTtsSetupGuide — modal tutorials for obtaining API keys
- **Sync** (~1729): v3 selective sync — clipboard + .json file + GitHub Gist; API Keys checkbox (opt-in, default off); `stripApiKeys()`, `getExportSettings()`
- **GitHub Gist Sync** (~1990): showGistSetupGuide, findSyncGist, renderSlotPicker, buildGistSyncData, exportToGist, saveToGistSlot, deleteGistSlot, importFromGist, loadGist
- **Word Pack** (~1960): Import JSON, format example with AI prompt
- **rebuildWords()** (~1937): Merges WORDS_RAW + customWords, dedupes by name
- **Ear Training** (~1959): Dictation, minimal pairs, AI listening comprehension
- **Placement Test** (~2244): AI-powered assessment, word pack recommendations
- **Add Words** (~2521): Manual add with duplicate warning (~2603), AI auto-fill, AI suggest
- **Duplicate Warning** (~2603): showDuplicateWarning() — replace/combine/keep both options
- **forceAddWord** (~2638): Handles replace, combine definitions, keep both
- **Gemini API** (~2896): callGemini with dual message format support
- **Voice Chat** (~2988): Recording, sendVoiceChat with Gemini audio
- **AI Chat** (~3044): SCENARIOS array (16 scenarios), startChat, customPrompt support
- **AI Explain** (~3335): aiExplainWord for word detail
- **AI Quiz Feedback** (~3360): aiQuizExplain ("Why was I wrong?")
- **Init** (~3430): DOMContentLoaded → rebuildWords, applyTheme, goTo('home')

## Bottom Nav
Home | Quiz | Cards | Words | Settings (Stats accessible via stat boxes on home)

## Key Patterns
- `formatMultiDef(def)` — numbered definitions for combined words
- `deleteWordById(wordId)` — universal delete (all words are custom)
- `deleteDupWord(groupIdx, which)` — per-word delete in duplicate review
- `showDuplicateWarning()` + `forceAddWord()` — duplicate check on manual add
- `exportSyncFile()` — downloads .json sync file (mobile-friendly export)
- `importSyncFile(input)` — reads .json file via file picker (mobile-friendly import)
- `stripApiKeys(settings)` — removes SENSITIVE_KEYS from settings for safe export
- `getExportSettings(opt)` — returns full or stripped settings based on apiKeys option
- `findSyncGist(token)` — lists gists, finds one with `vocab-slot-*.json` or `vocab-sync.json`, returns `{ gistId, slots, legacyFile }`
- `renderSlotPicker(mode, gistId, slots, legacyFile)` — renders 5-slot picker UI in `sync-code-area`; export mode shows Save/Overwrite, import mode shows Load/--
- `buildGistSyncData()` — builds sync payload with `_meta: { words, date }` added
- `exportToGist()` — calls `findSyncGist` then renders slot picker in export mode
- `saveToGistSlot(slotNum, gistId, isOccupied)` — saves to `vocab-slot-{N}.json`; confirm on overwrite and API keys; PATCHes existing or POSTs new gist
- `deleteGistSlot(slotNum, gistId, mode)` — deletes a slot file from the gist (sets file to null via PATCH); confirm dialog; refreshes slot picker
- `importFromGist()` — calls `findSyncGist` then renders slot picker in import mode; legacy `vocab-sync.json` shown with orange dashed border
- `loadGist(gistId, fileName)` — downloads and applies a gist file (unchanged, ignores `_meta`)
- `testGeminiKey()` — tests Gemini API key with a minimal prompt
- `testTtsKey()` — tests Google Cloud TTS key via voices list endpoint
- `testGistToken()` — tests GitHub token via GET /user, shows username on success
- `showGeminiSetupGuide()` — in-app tutorial for Gemini API key setup
- `showTtsSetupGuide()` — in-app tutorial for Google Cloud TTS API key setup
- `showGistSetupGuide()` — in-app tutorial for GitHub token setup
- `endSessionEarly(mode)` — End button for quiz, cards, voice, ear
- `dontKnowQuiz()` — reveals answer without guessing, records quality 0
- Word search uses split rendering to avoid input focus loss
- Theme: dark (default) / light via `[data-theme="light"]` CSS

## User Preferences
- Desktop path: `D:\Desktop` (not default Windows path)
- WSL2 environment, browser runs on Windows side
- Prefers minimal file count — PWA inlined, no external manifest/icons/sw.js
- No auto-memory — context saved locally in CLAUDE.md only
