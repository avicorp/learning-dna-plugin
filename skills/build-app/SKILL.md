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

### Step 2: Application Structure Setup
> **Only runs when `learning-app/` does not exist.** If the app already exists, skip to Step 3.

Set up the foundational project before any components are built:

1. Initialize a React 19 + TypeScript + Vite 6 + Tailwind CSS 4 project in `learning-app/`
2. Add Mermaid 11 for diagram rendering and React Router for routing
3. Create the full directory structure:
   ```
   learning-app/
     src/
       components/    # Reusable components (TTSReader, Layout, Sidebar, Navbar, Footer)
       contexts/      # React Context providers
       hooks/         # Custom hooks (useTTS, useIndexedDB, useProgress)
       pages/         # Page components (Welcome, TopicHome, TopicDetail, Quiz, QuizStatus)
       data/          # Generated JSON data files
       types/         # TypeScript type definitions
       lib/           # Utility functions (IndexedDB loader)
     scripts/         # Parse scripts for knowledge → JSON
   ```
4. Create `src/types/index.ts` with all TypeScript interfaces upfront (see Step 3 for the full type definitions)
5. Set up React Router with placeholder routes:
   - `/` — Welcome page
   - `/quiz-status` — Quiz status page
   - `/:topic` — Topic home page
   - `/:topic/:subtopic` — Topic detail page
   - `/:topic/quiz` — Quiz page
6. Install all dependencies, configure Vite and Tailwind

### Step 3: Data Model — React Context
Create a `LearningContext` in `src/contexts/LearningContext.tsx` that replaces all localStorage-based state management.

#### TypeScript Interfaces (in `src/types/index.ts`)
```typescript
interface AppState {
  topics: Topic[];
  topicProgress: Record<string, TopicProgress>;
}

interface Topic {
  slug: string;
  name: string;
  subtopics: string[];
}

interface TopicProgress {
  learningStatus: boolean;
  quizStatus: boolean;
  lastScore: QuizScore | null;
  subtopicStatus: Record<string, 'completed' | 'in-progress' | 'not-started'>;
  lastVisited: string | null;
}

interface QuizScore {
  total: number;
  correct: number;
  breakdown: {
    easy: [number, number];
    medium: [number, number];
    hard: [number, number];
  };
}
```

#### LearningProvider
- A `LearningProvider` component that wraps the entire app (in `main.tsx` or `App.tsx`)
- Loads initial state from IndexedDB on mount (via the data loader from Step 4)
- Persists state changes back to IndexedDB on update
- Exposes context methods:
  - `markSubtopicComplete(topicSlug: string, subtopic: string): void`
  - `updateQuizScore(topicSlug: string, score: QuizScore): void`
  - `setLastVisited(topicSlug: string, subtopic: string): void`
  - `getTopicProgress(topicSlug: string): TopicProgress`
- Export a `useLearning` hook for components to consume the context

#### Async-Safety Rules
- All context mutation methods MUST guard against being called before IndexedDB data has loaded. Check `isLoaded` before allowing mutations to prevent overwriting persisted state with defaults.
- Components that call context mutations on mount (e.g., `markSubtopicInProgress` in `useEffect`) MUST wait for `isLoaded === true` before calling. This prevents race conditions where the async IndexedDB load hasn't completed yet and mutations create fresh default objects that overwrite persisted data.

### Step 4: Data Loader — IndexedDB
Create `src/lib/indexedDB.ts` — a utility module for persistent storage using IndexedDB.

#### Database Schema
- **Database name:** `learningDNA`
- **Object stores:**
  - `topicProgress` — keyed by `topicSlug`, stores `TopicProgress` objects
  - `quizResults` — keyed by `topicSlug`, stores quiz attempt history arrays
  - `ttsSettings` — keyed by `topicSlug` (or `"global"` for shared defaults), stores TTS preferences

#### API Functions
```typescript
initDB(): Promise<IDBDatabase>
loadTopicProgress(slug: string): Promise<TopicProgress | null>
saveTopicProgress(slug: string, progress: TopicProgress): Promise<void>
loadQuizResults(slug: string): Promise<QuizScore[]>
saveQuizResult(slug: string, score: QuizScore): Promise<void>
loadTTSSettings(slug: string): Promise<TTSSettings | null>
saveTTSSettings(slug: string, settings: TTSSettings): Promise<void>
```

#### React Hook
Create `src/hooks/useIndexedDB.ts` — a React-friendly wrapper around the IndexedDB API functions. The hook handles async initialization and provides loading/error states.

#### Migration from localStorage
On first load, check if any `learningDNA-*` localStorage keys exist (from v1.x generated apps):
- Migrate `learningDNA-progress-{topicSlug}` → `topicProgress` store
- Migrate `learningDNA-tts-settings-{topicSlug}` → `ttsSettings` store
- Migrate `learningDNA-tts-settings` → `ttsSettings` store under key `"global"`
- After successful migration, delete the old localStorage keys

### Step 5: Application Layout
Create the layout with three zones: top navbar, side navigation, and footer.

#### Top Navbar (`src/components/Navbar.tsx`)
- Fixed position at the top of the viewport
- App title/logo on the left side
- Navigation buttons on the right:
  - **"Topics"** button — navigates to the welcome/home page (`/`)
  - **"Quiz"** button — navigates to the quiz status page (`/quiz-status`)
- **Responsive:** on mobile (<1024px), navigation buttons collapse into a hamburger menu

#### Side Navigation Bar (`src/components/Sidebar.tsx`)
- Collapsible sidebar on the left side, visible on all pages
- Toggle button (hamburger icon `☰`) always visible in the top-left corner of the layout
- **Desktop (≥1024px):** sidebar open by default, pushes main content to the right
- **Mobile (<1024px):** sidebar collapsed by default, overlays content with a semi-transparent backdrop when opened

##### Sidebar Contents
- **Topic title** displayed at the top of the sidebar
- **Subtopic list** — each subtopic rendered as a navigation link with:
  - **Status icon:** checkmark icon (completed), half-circle icon (in progress), empty circle icon (not started)
  - **Completion tracking:** a subtopic is marked "completed" when the user scrolls to the bottom of its content page (use an `IntersectionObserver` on a sentinel `<div>` placed at the end of the content). **Dependency note:** The `IntersectionObserver` effect MUST include `isLoaded` in its dependency array when gated on context readiness, so the observer is created after data loads.
  - **Active highlight:** the current subtopic is highlighted with the app's accent color
- **Quiz section** — link to the quiz page showing:
  - Score badge with best score (e.g., "8/10") or "Not attempted" if never taken
  - Difficulty breakdown if attempted (e.g., "Easy: 5/5, Medium: 2/3, Hard: 1/2")
- **Overall progress bar** at the bottom — percentage calculated as: (completed subtopics + quiz attempted) / (total subtopics + 1)

##### Progress Data Source
All progress data is read from the `LearningContext` (which persists to IndexedDB). On page load: restore sidebar open/closed state, scroll to last-visited subtopic.

#### Footer (`src/components/Footer.tsx`)
- Footer text: "Built with Learning DNA" (always in English, regardless of the app's language)
- "Learning DNA" must link to `https://github.com/avicorp/learning-dna-plugin`
- Link opens in new tab: `target="_blank" rel="noopener noreferrer"`

#### Layout Wrapper (`src/components/Layout.tsx`)
Composes Navbar + Sidebar + Footer + main content area. All pages render inside this layout.

### Step 6: TTS Component
Extract TTS into a reusable component specification. Create `src/components/TTSReader.tsx` and `src/hooks/useTTS.ts`.

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
- **Settings persistence:** per-topic settings stored in IndexedDB under the `ttsSettings` object store keyed by `topicSlug`. Falls back to the `"global"` key if no per-topic settings exist. On first load (no saved settings), initialize defaults from the LearningDNA profile (see DNA-driven defaults below)

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
On first load (no IndexedDB settings), the `useTTS` hook reads the LearningDNA profile embedded in the app data and sets intelligent defaults:
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
- A shared `useTTS` hook to manage: voice settings, playback state, progress tracking, keyboard shortcuts, DNA defaults initialization, and IndexedDB persistence
- The TTSReader component can be used on any page that displays text content, not just topic detail pages

### Step 7: Parse Content to JSON
For each topic with content:
1. Check if a parse script exists in `learning-app/scripts/` for this topic
2. If not, create one:
   - Read all markdown files from `knowledge/{topic}/sources/`
   - Parse H1/H2/H3 structure, mermaid diagrams, content sections
   - Output JSON to `learning-app/src/data/{topic}-topics.json`
3. If quiz bank exists at `knowledge/{topic}/quizzes/quiz-bank.json`:
   - Copy/transform to `learning-app/src/data/{topic}-quizzes.json`
4. Run the parse script: `npx tsx learning-app/scripts/parse-{topic}.ts`

### Step 8: Scaffold Topic UI Pages (if needed)
If the topic lacks UI pages in `learning-app/src/pages/`, scaffold them following existing patterns:
- **Topic home page** — subtopic list with progress from `LearningContext`
- **Topic detail page** — rendered markdown + mermaid diagrams + TTSReader component
- **MarkdownContent component** must implement the TTS heading and paragraph behavior described in Step 6
- **Quiz page** — interactive quiz with scoring, explanations, and difficulty breakdown
- Routes configuration updated to include the new topic

### Step 9: Welcome / Home Page
Create a welcome page at route `/` and a quiz status page at `/quiz-status`.

#### Welcome Page (`src/pages/Welcome.tsx`)
- Displays a grid/list of all available knowledge bases (topics)
- Each topic card shows:
  - Topic name
  - Number of subtopics
  - Overall progress percentage (from `LearningContext`)
  - Quiz status badge (score or "Not attempted")
  - Last visited date
- Clicking a topic card navigates to that topic's home page (`/:topic`)
- The "Topics" button in the top navbar always navigates here

#### Quiz Status Page (`src/pages/QuizStatus.tsx`)
- Route: `/quiz-status`
- Lists all topics with their quiz status
- For each topic: topic name, number of questions, last score, best score, difficulty breakdown
- "Take Quiz" or "Retake Quiz" button per topic, linking to `/:topic/quiz`

### Step 10: Application Validation
> **Gate:** Do not proceed to building topic content until all checks pass.

Dispatch the `app-validator` agent to validate the scaffolding. The validation checks:

1. **Build check:** Run `npm run build` — must complete with zero errors
2. **Component check:** Verify all core components exist and export correctly:
   - `src/components/Layout.tsx` (with Navbar, Sidebar, Footer)
   - `src/components/TTSReader.tsx`
   - `src/contexts/LearningContext.tsx`
   - `src/lib/indexedDB.ts`
3. **Routing check:** Verify all routes are configured:
   - `/` — Welcome page
   - `/quiz-status` — Quiz status page
   - `/:topic` — Topic home page
   - `/:topic/:subtopic` — Topic detail page
   - `/:topic/quiz` — Quiz page
4. **Data model check:** Verify the `LearningProvider` wraps the App component, and that IndexedDB initialization runs without errors

If any check fails, fix the issue before proceeding to Step 11.

### Step 11: Build Verification
Run the build to verify everything compiles:
```bash
cd learning-app && npm run build
```

**Footer assertion** — after a successful build, verify the footer exists and is correct:
1. Grep the built output (`learning-app/dist/`) or source components for the exact text `Built with Learning DNA`
2. Confirm the link `https://github.com/avicorp/learning-dna-plugin` is present with `target="_blank"`
3. If either check fails, fix the Layout component and rebuild before proceeding

### Step 12: Report
Report what was built:
- Which topics were processed
- Which parse scripts were created/run
- Which UI pages were scaffolded
- Whether the app structure was newly created (Step 2) or already existed
- Application validation results (Step 10)
- Build success/failure status
