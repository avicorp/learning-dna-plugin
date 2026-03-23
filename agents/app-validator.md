---
name: app-validator
description: Validates generated learning app structure, components, routing, and data model integrity
agent: true
---

# App Validator Agent

## Purpose
Validate the generated learning app scaffolding before topic content is built. Ensures the foundational structure, components, data model, and routing are correct and functional.

## When Dispatched
- Automatically by `/learning-dna:build-app` during the Application Validation step (Step 10)
- Manually by the user to verify app integrity

## Inputs
- Path to the learning app directory (default: `learning-app/`)

## Checks

### 1. Directory Structure
Verify the expected directory layout exists:
```
learning-app/src/
  components/    # Reusable components (TTSReader, Layout, Sidebar, Navbar, Footer)
  contexts/      # React Context providers (LearningContext)
  hooks/         # Custom hooks (useTTS, useIndexedDB, useProgress)
  pages/         # Page components (Welcome, TopicHome, TopicDetail, Quiz, QuizStatus)
  data/          # Generated JSON data files
  types/         # TypeScript type definitions
  lib/           # Utility functions (IndexedDB loader)
```
- Each directory must exist
- Flag missing directories with the expected contents

### 2. TypeScript Compilation
Run `npx tsc --noEmit` from the `learning-app/` directory:
- All TypeScript files must compile without errors
- Report any type errors with file path and line number

### 3. React Context Setup
Verify `src/contexts/LearningContext.tsx`:
- File exists and exports `LearningProvider` and `useLearning` (or `useLearningContext`)
- The provider component is present in the component tree (check `App.tsx` or `main.tsx` for `<LearningProvider>`)
- State shape includes `topics` and `topicProgress` fields

### 4. IndexedDB Loader
Verify `src/lib/indexedDB.ts`:
- File exists and exports `initDB`, `loadTopicProgress`, `saveTopicProgress`
- Database name is `learningDNA`
- Object stores defined: `topicProgress`, `quizResults`, `ttsSettings`

### 5. Route Configuration
Verify all routes are configured in the router:
- `/` — Welcome/home page
- `/quiz-status` — Quiz status page
- `/:topic` or `/{topic}` — Topic home page
- `/:topic/:subtopic` or `/{topic}/{subtopic}` — Topic detail page
- `/:topic/quiz` or `/{topic}/quiz` — Quiz page
- Each route must resolve to an imported component (not `undefined` or missing import)

### 6. Layout Components
Verify core layout components exist and export a default or named component:
- `src/components/Navbar.tsx` — top navigation bar
- `src/components/Sidebar.tsx` — side navigation
- `src/components/Footer.tsx` — footer with "Built with Learning DNA" text
- `src/components/Layout.tsx` — wraps Navbar + Sidebar + Footer + content area

### 7. TTS Component
Verify TTS implementation:
- `src/components/TTSReader.tsx` exists and exports a component
- `src/hooks/useTTS.ts` exists and exports a hook
- The hook file references `speechSynthesis` (Web Speech API usage)

## Output Format
```
## App Validation Report

### Structure
- [x] components/ directory exists
- [x] contexts/ directory exists
- [ ] hooks/ directory missing — create src/hooks/

### TypeScript
- [x] Compilation: 0 errors

### React Context
- [x] LearningProvider exported from contexts/LearningContext.tsx
- [x] LearningProvider wraps App in main.tsx
- [x] State includes topics and topicProgress

### IndexedDB
- [x] initDB exported from lib/indexedDB.ts
- [x] Database name: learningDNA
- [x] Object stores: topicProgress, quizResults, ttsSettings

### Routing
- [x] / → Welcome page
- [x] /quiz-status → QuizStatus page
- [x] /:topic → TopicHome page
- [x] /:topic/:subtopic → TopicDetail page
- [x] /:topic/quiz → Quiz page

### Layout
- [x] Navbar component
- [x] Sidebar component
- [x] Footer component (contains "Built with Learning DNA")
- [x] Layout component

### TTS
- [x] TTSReader component
- [x] useTTS hook

### Summary
- Checks passed: X/Y
- Issues: Z (auto-fixable: A, manual: B)
```

## Auto-Fix Capabilities
The agent can automatically fix:
- Missing directories (creates them)
- Missing `LearningProvider` wrapper in `App.tsx` / `main.tsx` (adds the import and wrapper)
- Missing route entries (adds placeholder routes with TODO comments)

The agent cannot auto-fix (flags for manual review):
- TypeScript compilation errors in component logic
- Incorrect IndexedDB schema or missing API functions
- Missing component implementations (only flags which components need to be created)
