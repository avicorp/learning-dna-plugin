---
name: build-app
description: Rebuild learning app data from knowledge/ content and scaffold UI if needed
disable-model-invocation: true
argument-hint: "[topic]"
---

# /build-app — Rebuild Learning App Data

## Usage
`/learning-dna:build-app` — rebuild all topics
`/learning-dna:build-app <topic>` — rebuild a specific topic

## Flow

### Step 1: Scan Knowledge Directory
- Scan `knowledge/` for topics with content (directories containing `sources/` with markdown files)
- If a specific topic is provided, only process that topic
- Skip directories without source content

### Step 2: Parse Content to JSON
For each topic with content:
1. Check if a parse script exists in `learning-app/scripts/` for this topic
2. If not, create one:
   - Read all markdown files from `knowledge/{topic}/sources/`
   - Parse H1/H2/H3 structure, mermaid diagrams, content sections
   - Output JSON to `learning-app/src/data/{topic}-topics.json`
3. If quiz bank exists at `knowledge/{topic}/quizzes/quiz-bank.json`:
   - Copy/transform to `learning-app/src/data/{topic}-quizzes.json`
4. Run the parse script: `npx tsx learning-app/scripts/parse-{topic}.ts`

### Step 3: Scaffold UI Pages (if needed)
If the learning app does not exist yet, scaffold it:
- Initialize a new React 19 + TypeScript + Vite 6 + Tailwind CSS 4 project in `learning-app/`
- Add Mermaid 11 for diagram rendering
- Create the base layout, routing, and navigation

If the topic lacks UI pages in `learning-app/src/`, scaffold following existing patterns:
- Topic home page
- Topic detail page with **TTS (Text-to-Speech) support**:
  - Each heading (h1, h2, h3) must be wrapped in a `div` with `display: flex; justify-content: space-between; align-items: center`
  - Each heading wrapper includes a TTS button (speaker icon) on the opposite side
  - TTS uses the Web Speech API (`window.speechSynthesis`)
  - On click: reads the heading text + all section content until the next heading
  - Set utterance `lang` to match the LearningDNA language (e.g., `"he"` for Hebrew, `"en"` for English)
  - Toggle state: playing shows stop icon, stopped shows play icon
  - **TTS settings popup** (opened from a three-dots `⋮` button placed next to the TTS play buttons; clicking the button opens a popup menu with the settings, clicking outside or pressing Escape closes it):
    - **Speed control:** slider from 0.5x to 2x (default 1x), maps to `utterance.rate`
    - **Voice selection:** dropdown populated from `speechSynthesis.getVoices()`, filtered to voices matching the LearningDNA language. Prefer natural/premium voices when available (e.g., voices with "Natural", "Enhanced", or "Premium" in name)
    - **Pitch control:** slider from 0.5 to 1.5 (default 1), maps to `utterance.pitch`
    - Settings persist in `localStorage` under key `learningDNA-tts-settings`
    - On first load, auto-select the best available voice for the language (prefer natural-sounding voices)
  - Implementation approach: `useRef` + `useEffect` on the markdown container to post-process rendered headings and inject TTS buttons; a shared `useTTS` hook to manage voice settings, playback state, and localStorage persistence
- **MarkdownContent component** must also implement the TTS heading behavior described above
- Quiz page
- Layout component with **footer**:
  - Footer text: "נבנה עם Learning DNA 🧬" (or equivalent in the user's LearningDNA language)
  - The text must link to `https://github.com/avicorp/learning-dna-plugin`
  - Link opens in new tab: `target="_blank" rel="noopener noreferrer"`
- Routes configuration
- TypeScript types
- Progress tracking utilities

### Step 4: Build Verification
Run the build to verify everything compiles:
```bash
cd learning-app && npm run build
```

### Step 5: Report
Report what was built:
- Which topics were processed
- Which parse scripts were created/run
- Which UI pages were scaffolded
- Build success/failure status
