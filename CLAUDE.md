# Canadian English Vocabulary Trainer

Main app: `vocab-trainer.html` (~7143 lines). PWA manifest & icons generated inline. Hosted on GitHub Pages.

## Files
- `vocab-trainer.html` — the entire app (HTML/CSS/JS + inline PWA manifest/icons)
- `sw.js` — minimal Service Worker (notification click handler, PWA install support)
- `CLAUDE.md` — this file (context for Claude)

## Architecture
- Single HTML/CSS/JS file, no build tools
- localStorage persistence (`vocabApp` key)
- **IndexedDB** (`vocabAudio` DB) for Story Mode audio blob cache
- Gemini API (configurable model via `state.settings.geminiModel`) for AI features
- Google Cloud TTS + browser TTS fallback
- GitHub Gist cloud sync via Personal Access Token + REST API
- SM-2 Spaced Repetition System
- All words are user-imported (WORDS_RAW is empty)
- PWA manifest & icons inlined (canvas-generated data URLs, blob manifest)
- Service Worker for notification click handling and PWA install
- Push notifications at 8:00 AM, 12:00 PM, 10:00 PM (requires app to be running)
- Hosted on GitHub Pages: https://galiiigigi.github.io/vocab-trainer/

## Key State
- `state.customWords[]` — all words (no built-in words)
- `state.wordProgress{}` — SRS data per word
- `state.settings` — API keys, theme, voice, thresholds
- `state.ttsUsage` — monthly character tracking
- `state.dailyActivity{}` — heatmap data
- `state.listeningHistory[]` — Story Mode generated passages (id, format, passage, words, vibe, audioKey, createdAt, lastPlayed, plays). LRU cap via `state.settings.storyMaxHistory` (default 200)
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
- **Gemini Model** (~390): `getGeminiModel()`, `getGeminiUrl(key)` — configurable model selector
- **SRS** (~349): SM-2 algorithm, getDueWords, getSessionWords
- **XP & CEFR System** (~570): CEFR_LEVELS, getCEFRLevel, getCEFRProgress, getSkillLevels, awardXP (diminishing returns per word+mode)
- **TTS** (~469): speak(), googleSpeak(), fetchGoogleVoices(), buildVoiceOptions(), testGoogleTTS()
- **cleanDefinition** (~539): Strips word itself from AI-generated definition starts
- **getVocabPoolForPrompt** (~565): Gets up to 60 learner words for AI prompt reinforcement
- **API Key Tests** (~606): testGeminiKey(), testTtsKey(), testGistToken()
- **Navigation** (~631): goTo(), renderScreen()
- **Home** (~663): Stat boxes, due words, empty state, AI feature buttons
- **Quiz** (~703): Timer, answerQuiz, dontKnowQuiz, skipQuiz, markKnownQuiz, endSessionEarly
- **Flashcards** (~941): Flip, rate, skip, markKnown, listeningMode, endSessionEarly
- **Voice Test** (~1056): Mic recognition, fuzzyMatch, endSessionEarly
- **Audio-Only Quiz** (~1133): `startAudioQuiz()`, listen-only quiz with full SRS, review, summary
- **Mark Known** (~1215): markWordAsMastered, markKnownQuiz/Flashcard/Voice
- **Flashcard Listening Mode** (~1344): `startListeningFlashcards()` — audio-only front, flip to reveal
- **Word List** (~1260): Split render (search input fix), updateWordListResults()
- **Duplicates** (~1308): AI-powered findDuplicateWords, per-word delete, combine/merge
- **Word Detail** (~1440): Modal with phonetic, grammar, SRS info, delete button
- **Stats** (~1478): Stat grid, heatmap, category progress bars
- **Settings** (~1554): Theme, API keys, TTS config, Gemini model selector (6 models with descriptions/warnings), sync, danger zone
- **Auto-Play Word List** (~1754): Podcast-style sequential playback (word→pause→definition→pause→example→gap→next)
- **Sync** (~1729): v3 selective sync — clipboard + .json file + GitHub Gist
- **GitHub Gist Sync** (~1990): showGistSetupGuide, findSyncGist, renderSlotPicker, exportToGist, importFromGist
- **API Setup Guides** (~1940): showGeminiSetupGuide, showTtsSetupGuide
- **rebuildWords()** (~1937): Merges WORDS_RAW + customWords, dedupes by name
- **Gemini JSON Importer** (~2573): `parseGeminiImport()` — smart parser for arrays, dicts, wrapped objects
- **Ear Training** (~1959): Dictation, minimal pairs, AI listening comprehension
- **Shadowing Mode** (~3377): `startEarShadowing()` — listen, record, AI pronunciation feedback
- **Substitution Drill** (~3675): `startSubDrill()` — FSI-style sentence substitution with AI evaluation
- **Listen & Respond** (~3870): `startListenRespond()` — Pimsleur-style prompt→think→respond→AI feedback, 10 scenarios
- **Placement Test** (~2244): AI-powered assessment, word pack recommendations
- **Add Words** (~2521): Manual add with duplicate warning, AI auto-fill (multi-meaning), AI suggest
- **Multi-Meaning AutoFill** (~4113): `syncMeaningsToFields()`, `renderMeaningsList()`, `removeMeaning()` — multiple senses per word
- **AI AutoFill** (~4154): `aiAutoFill()` — returns all meanings array, fallback to single definition
- **Duplicate Warning / forceAddWord**: replace/combine/keep both
- **Gemini API** (~2896): callGemini, cleanGeminiText (~4426) strips json prefix/fences
- **Voice Chat** (~2988): Recording, sendVoiceChat with Gemini audio
- **AI Chat** (~3044): SCENARIOS array (16 scenarios), startChat, startFreeChat, customPrompt support
- **Free Chat** (~3660): Open-ended bilingual grammar correction, reviews weak words
- **Language Parent** (~6185): `startLanguageParent()` — Chris Lonsdale method, recasts instead of correcting, simple language, body language cues
- **Word Mixing Drill** (~6215): `startWordMixing()` — combine 2-3 words into sentences, AI evaluates, SRS+XP
- **Brain Soak** (~6372): `startBrainSoak()` — passive immersion, plays sentences on loop with repeat, speed/loop controls
- **Story Mode** (~1280-1670): `openStoryMode()`, `generateStory(format)`, `renderStoryScreen()`, `renderStoryPlayer()`, `playStory()`. Two formats: Podcast monologue + Interview Q&A. 10 vibes per format rotated silently. Non-mastered words only via `pickNonMasteredWords(5)`. Audio cached in IndexedDB → replays = 0 TTS quota. Library UI with LRU-capped history. Entry: Input tab → Passive Immersion section
- **Vocab Map** (~7136 onwards): `startVocabTest()`, `answerVocabTest()`, `computeVocabEstimate()`, `commitVocabTestToLibrary()`, `renderVocabTest()`. Frequency × domain matrix test (7 domains × up to 5 bands ≈ 27 cells). Word bank inline (~330 real words across daily/academic/business/canadian/literary/colloquial/phrasal + 30 fake-word distractors). Lextale-style fake correction (penalizes overclaiming). Output: total estimate + CEFR + 7×5 heatmap. Test results commit into vocab library with proper SRS status (Know→Mastered interval=21d, Unsure→Learning interval=3d, Don't know→New). Entry: Home screen "Vocab Map" button. State: `vtest` global + `state.vmapHistory[]` (capped at 20).
- **Audio Cache** (~914): `openAudioDB()` + `audioDB.{put,get,delete,listKeys}` IndexedDB wrapper, `base64ToBlob()` helper
- **Add Word from Chat** (~3870): showAddWordFromChatModal, addSuggestedWord, addAllSuggestedWords
- **AI Explain** (~3935): aiExplainWord for word detail
- **AI Rephrase** (~5168): `aiRephraseWord()` single + `batchRephraseWords()` batch, with undo
- **AI Quiz Feedback** (~3960): aiQuizExplain ("Why was I wrong?")
- **Notifications** (~3980): requestNotificationPermission, initNotifications, checkNotificationTime
- **Init** (~4010): DOMContentLoaded → rebuildWords, applyTheme, goTo('home'), SW register, initNotifications

## Bottom Nav
Home | Output | Input | Words | Settings (Stats accessible via stat boxes on home)

- **Output tab:** speaking/active production modes (chats, drills, scenarios)
- **Input tab:** listening/passive intake. Sections: Passive Immersion (Brain Soak, Story) / Active Listening (Audio Quiz, Listen Cards, Listening Comprehension) / Ear Training (Dictation, Minimal Pairs)

## Key Patterns
- `formatMultiDef(def)` — numbered definitions for combined/multi-meaning words
- `cleanDefinition(word, def)` — strips word itself from AI definitions
- `cleanGeminiText(text)` — strips `json\n` prefix and code fences from Gemini responses
- `getVocabPoolForPrompt(excludeWord)` — up to 60 learner words for AI reinforcement
- `getGeminiModel()` / `getGeminiUrl(key)` — configurable Gemini model (6 options)
- `syncMeaningsToFields()` — syncs `pendingMeanings[]` to def/example text fields
- `renderMeaningsList()` — displays multi-meaning preview with delete buttons
- `aiRephraseWord(wordId)` / `batchRephraseWords()` — AI rephrase definitions with undo
- `startAudioQuiz()` — audio-only quiz, full SRS integration
- `startListeningFlashcards()` — audio-only front, flip to reveal
- `startAutoPlay()` / `stopAutoPlay()` — podcast-style word list playback
- `startEarShadowing()` — listen + record + AI pronunciation eval
- `startSubDrill()` — FSI substitution drill, AI generates sentence + evaluates substitution
- `startListenRespond(scenarioId)` — Pimsleur listen→think→record→AI eval, 10 workplace scenarios
- `LISTEN_RESPOND_SCENARIOS` — 10 scenarios with difficulty levels (greeting, phone, meeting, interview, etc.)
- `parseGeminiImport()` — smart JSON importer (arrays, dicts, wrapped objects)
- `getDailyWord()` — random due/weak word for home "Word of the Moment"
- `startFreeChat()` — bilingual grammar correction ([Correction] X/O/Tip format)
- `startLanguageParent()` — Chris Lonsdale Language Parent chat (recast, no correction)
- `startWordMixing()` — word combo drill, 2-3 words → sentence → AI eval
- `startBrainSoak()` / `stopBrainSoak()` — passive sentence immersion player
- `brainSoakPlayNext()` — plays sentence twice (normal + slow), auto-advances
- `openStoryMode()` — Story Mode entry (nav to `screen-story`)
- `pickNonMasteredWords(n)` — returns random N words with status !== 'mastered'
- `generateStory(format)` — orchestrator: picks words + vibe, calls Gemini, saves to `state.listeningHistory`, preloads audio
- `fetchAndCacheStoryAudio(text, key)` — fetch Google TTS → save Blob to IndexedDB
- `playStory()` — cache-aware playback (hit = 0 quota; miss = fetch + cache)
- `STORY_FORMATS` / `VIBES_PODCAST` / `VIBES_INTERVIEW` — format registry + vibe pools
- `showAddWordFromChatModal()` — save word during chat with AI auto-fill
- `endSessionEarly(mode)` — End button for quiz, cards, voice, ear
- `dontKnowQuiz()` — reveals answer without guessing, records quality 0
- `awardXP(source, wordId, amount)` — XP with diminishing returns (1st=100%, 2nd=50%, 3rd=25%, 4th+=0% per word+source). Sources: review, newword, milestone-*, freechat, chat:*, langparent, subdrill, shadowing, dictation, listening, listen-respond:*, wordmixing, brainsoak
- `getCEFRLevel()` / `getCEFRProgress()` — current level + progress to next based on total XP
- `getSkillLevels()` — per-skill CEFR estimates (vocab, speaking, listening, drills)
- `CEFR_LEVELS` — A1→C2 with XP thresholds, vocab targets, IELTS equivalents
- Gist sync: `findSyncGist`, `renderSlotPicker`, `exportToGist`, `importFromGist`, 5 save slots
- Word search uses split rendering to avoid input focus loss
- Theme: dark (default) / light via `[data-theme="light"]` CSS

## User Preferences
- Desktop path: `D:\Desktop` (not default Windows path)
- WSL2 environment, browser runs on Windows side
- Prefers minimal file count — PWA inlined, no external manifest/icons/sw.js
- No auto-memory — context saved locally in CLAUDE.md only
