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

  #### TTS Read Buttons (headings + paragraphs)
  - Each heading (h1, h2, h3) must be wrapped in a `div` with `display: flex; justify-content: space-between; align-items: center`
  - Each heading wrapper includes a TTS button (speaker icon) on the opposite side
  - TTS uses the Web Speech API (`window.speechSynthesis`)
  - **Heading button:** reads the heading text + all section content until the next heading (section-level reading)
  - **Paragraph button:** each `<p>` element gets a small speaker icon button that appears on hover (opacity transition). Clicking it reads only that single paragraph. The button is smaller than the heading button to maintain visual hierarchy
  - **"Read All" button:** a floating button at the top of the topic detail page (next to the topic title) that reads the entire page content sequentially from top to bottom. Shows a stop button while reading
  - Set utterance `lang` to match the LearningDNA language (e.g., `"he"` for Hebrew, `"en"` for English)
  - Toggle state: playing shows stop icon, stopped shows play icon

  #### TTS Settings Popup
  Opened from a three-dots `⋮` button placed next to the TTS play buttons; clicking the button opens a popup menu with the settings, clicking outside or pressing Escape closes it:
  - **Speed presets:** quick-select buttons displayed as a row of chips above the speed slider:
    - "Slow & Clear" (0.7x) / "Normal" (1x) / "Fast" (1.5x) / "Speed Reader" (2x)
    - Selecting a preset updates the slider; moving the slider deselects the active preset
  - **Speed control:** slider from 0.5x to 2x (default 1x), maps to `utterance.rate`
  - **Volume control:** slider from 0 to 1 (default 1), maps to `utterance.volume`
  - **Voice selection:** dropdown populated from `speechSynthesis.getVoices()`, filtered to voices matching the LearningDNA language. Voice ranking priority: "Natural" > "Enhanced" > "Premium" > default system voices
  - **"Preview Voice" button:** next to the voice dropdown — speaks a short sample sentence in the selected voice and language (e.g., "This is how I sound" localized to the configured language). Useful for comparing voices before committing to one
  - **Pitch control:** slider from 0.5 to 1.5 (default 1), maps to `utterance.pitch`
  - **Settings persistence:**
    - Per-topic settings stored in `localStorage` under key `learningDNA-tts-settings-{topicSlug}`
    - Falls back to global `learningDNA-tts-settings` if no per-topic settings exist
    - On first load (no saved settings), initialize defaults from the LearningDNA profile (see DNA-driven defaults below)

  #### Reading Progress Visual Feedback
  - **Sentence highlighting:** during TTS playback, highlight the currently-spoken sentence with a light yellow background (`bg-yellow-100`). Use the `SpeechSynthesisUtterance` `boundary` event to track word position and map it to the corresponding sentence in the DOM
  - **Progress bar:** a thin progress bar (3px height, accent color) appears below the heading or paragraph being read, showing percentage of content spoken
  - **Auto-scroll:** smoothly scroll the viewport to keep the currently-read sentence visible using `scrollIntoView({ behavior: 'smooth', block: 'center' })`
  - The `useTTS` hook must expose `currentWordIndex`, `currentSentenceElement`, and `progress` (0-1) state for components to consume

  #### Keyboard Shortcuts for TTS
  - **Space** — play/pause current section (only when focus is NOT in an input, textarea, or contenteditable element)
  - **Escape** — stop playback entirely
  - **`]` / `[`** — increase / decrease speed by 0.1x (clamped to 0.5x–2x range)
  - **`N`** — skip to next section or paragraph and start reading
  - **`P`** — skip to previous section or paragraph and start reading
  - **`?` icon** in the TTS settings popup — opens a tooltip showing all available keyboard shortcuts
  - Implementation: global `keydown` event listener managed by the `useTTS` hook, automatically disabled when focus is in form elements

  #### DNA-Driven TTS Defaults
  On first load (no localStorage settings), the `useTTS` hook reads the LearningDNA profile embedded in the app data and sets intelligent defaults:
  - **Learner Type → Speed:**
    - Kids (6-12) → default speed 0.85x, prefer child-friendly/clear voices
    - Youth (13-22) → default speed 1.0x
    - Professional (23-60) → default speed 1.1x
    - Lifelong (60+) → default speed 0.9x, prefer clear/natural voices
  - **Language** → auto-select the best matching voice using the ranking: "Natural" > "Enhanced" > "Premium" > default
  - **Content Depth → Reading behavior:**
    - Brief → auto-pause 1 second between sections during "Read All"
    - Comprehensive → continuous reading without extra pauses
  - Users can always override these defaults — DNA only sets the initial starting values

  #### TTS Implementation Approach
  - `useRef` + `useEffect` on the markdown container to post-process rendered headings and paragraphs, injecting TTS buttons
  - A shared `useTTS` hook to manage: voice settings, playback state, progress tracking, keyboard shortcuts, DNA defaults initialization, and localStorage persistence

- **MarkdownContent component** must also implement the TTS heading and paragraph behavior described above
- Quiz page
- Layout component with **footer**:
  - Footer text: "Built with Learning DNA 🧬" (always in English, regardless of the app's language)
  - "Learning DNA" must link to `https://github.com/avicorp/learning-dna-plugin`
  - Link opens in new tab: `target="_blank" rel="noopener noreferrer"`
- Routes configuration
- TypeScript types
- **Side menu with progress tracking:**

  #### Sidebar Layout
  - Collapsible sidebar on the left side of topic detail pages and quiz pages
  - Toggle button (hamburger icon `☰`) always visible in the top-left corner of the layout
  - **Desktop (≥1024px):** sidebar open by default, pushes main content to the right
  - **Mobile (<1024px):** sidebar collapsed by default, overlays content with a semi-transparent backdrop when opened

  #### Sidebar Contents
  - **Topic title** displayed at the top of the sidebar
  - **Subtopic list** — each subtopic rendered as a navigation link with:
    - **Status icon:** checkmark icon (completed), half-circle icon (in progress), empty circle icon (not started)
    - **Completion tracking:** a subtopic is marked "completed" when the user scrolls to the bottom of its content page (use an `IntersectionObserver` on a sentinel `<div>` placed at the end of the content)
    - **Active highlight:** the current subtopic is highlighted with the app's accent color
  - **Quiz section** — link to the quiz page showing:
    - Score badge with best score (e.g., "8/10") or "Not attempted" if never taken
    - Difficulty breakdown if attempted (e.g., "Easy: 5/5, Medium: 2/3, Hard: 1/2")
  - **Overall progress bar** at the bottom of the sidebar — shows percentage calculated as: (completed subtopics + quiz attempted) / (total subtopics + 1)

  #### Progress Persistence
  - Stored in `localStorage` under key `learningDNA-progress-{topicSlug}`
  - Data shape:
    ```json
    {
      "subtopics": { "networking": "completed", "storage": "in-progress", "security": "not-started" },
      "quizBestScore": { "total": 10, "correct": 8 },
      "lastVisited": "networking"
    }
    ```
  - On page load: restore sidebar open/closed state, scroll to last-visited subtopic

### Step 4: Build Verification
Run the build to verify everything compiles:
```bash
cd learning-app && npm run build
```

**Footer assertion** — after a successful build, verify the footer exists and is correct:
1. Grep the built output (`learning-app/dist/`) or source components for the exact text `Built with Learning DNA`
2. Confirm the link `https://github.com/avicorp/learning-dna-plugin` is present with `target="_blank"`
3. If either check fails, fix the Layout component and rebuild before proceeding

### Step 5: Report
Report what was built:
- Which topics were processed
- Which parse scripts were created/run
- Which UI pages were scaffolded
- Build success/failure status
