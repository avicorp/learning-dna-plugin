---
name: add-quizzes
description: Generate quiz questions from source material, shaped by LearningDNA
disable-model-invocation: true
argument-hint: <topic>
---

# /add-quizzes — Generate Quiz Questions

## Usage
`/learning-dna:add-quizzes <topic>`

Example: `/learning-dna:add-quizzes kubernetes`

## Flow

### Step 1: DNA Gate (MANDATORY)
1. Check if `knowledge/LearningDNA.md` exists — if missing, **STOP. Do not proceed.** Tell the user: "You need to set up your Learning DNA first. Run `/learning-dna:new-topic <topic>` to get started."
2. Read `knowledge/LearningDNA.md` (global DNA)
3. Check if `knowledge/{topic}/LearningDNA.md` exists (per-topic override)
4. Merge: per-topic values override global values → use merged DNA

### Step 2: Read Source Material
- Read all markdown files from `knowledge/{topic}/sources/`
- If no source files exist (or only overview.md), suggest running `/learning-dna:research {topic} <subtopic>` first

### Step 3: Generate Questions Shaped by DNA

**Testing Frequency:**
- `Light` → 3-5 questions per subtopic
- `Standard` → 5-10 questions per subtopic
- `Heavy` → 10-20 questions per subtopic

**Knowledge Level → Difficulty Distribution:**
- `Beginner` → 60% easy, 30% medium, 10% hard
- `Some exposure` → 40% easy, 40% medium, 20% hard
- `Working knowledge` → 20% easy, 40% medium, 40% hard
- `Expert refresher` → 10% easy, 30% medium, 60% hard

**Learning Goal:**
- `Quick refresher` → focus on key facts and definitions
- `Practical skills` → scenario-based questions with practical applications
- `Deep understanding` → conceptual questions requiring synthesis
- `Interview prep` → include scenario and system-design style questions

**Learner Type:**
- `Kids` → gamified, fun language, relatable scenarios
- `Youth` → engaging, school-exam style
- `Professional` → industry-relevant, production scenarios
- `Lifelong` → curiosity-driven, connecting concepts

**Language:**
- Generate all questions, options, and explanations in the specified language

**Quiz JSON Schema:**
```json
{
  "id": "string — unique identifier, e.g., {topic}-{subtopic}-{number}",
  "topicSlug": "string — the topic slug",
  "question": "string — the question text",
  "options": ["string — array of 4 answer choices"],
  "correctIndex": "number — 0-based index of correct answer",
  "explanation": "string — educational explanation of the correct answer",
  "difficulty": "string — easy | medium | hard"
}
```

### Step 4: Present for Review
- Show generated questions to the user grouped by difficulty
- Wait for user approval or modification requests

### Step 5: Write Output
- Write to `knowledge/{topic}/quizzes/quiz-bank.json`
- If the file already has questions, merge new questions (avoid duplicates by checking question text similarity)

### Step 6: Dispatch Agent
- Dispatch the `quiz-improver` agent to review and enhance question quality

### Step 7: Report
- Report counts by difficulty: "Generated X questions: Y easy, Z medium, W hard"
