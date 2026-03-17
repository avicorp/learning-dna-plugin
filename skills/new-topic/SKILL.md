---
name: new-topic
description: Create a new learning topic with directory structure and optional per-topic LearningDNA override
disable-model-invocation: true
argument-hint: <topic-name>
---

# /new-topic — Create a New Learning Topic

## Usage
`/learning-dna:new-topic <topic-name>`

Example: `/learning-dna:new-topic kubernetes`

## Flow

### Step 1: DNA Gate (MANDATORY)
1. Check if `knowledge/LearningDNA.md` exists
2. If missing → **STOP. Do not proceed.** Run the DNA Creation Flow first:
   a. Inform the user: "Before we create any topic, you need to set up your Learning DNA — this is your personal learner profile that shapes all content generated for you. There are no defaults — your answers drive everything."
   b. Ask 8 questions one at a time using AskUserQuestion with multiple-choice options. **Do not skip any question. Do not use defaults. The user must answer each one:**
      - **Language:** English / Spanish / Hebrew / Chinese / Arabic / Other (specify)
      - **Learner Type:** Kids (6-12) / Youth (13-22) / Professional (23-60) / Lifelong (60+)
      - **Content Depth:** Brief / Standard / Detailed / Comprehensive
      - **Example Style:** Minimal / Code-focused / Real-world scenarios / Mixed
      - **Visualization:** Low / Medium / High
      - **Testing Frequency:** Light (3-5 per topic) / Standard (5-10) / Heavy (10-20)
      - **Knowledge Level:** Beginner / Some exposure / Working knowledge / Expert refresher
      - **Learning Goal:** Quick refresher / Practical skills / Deep understanding / Interview prep
   c. Write the user's responses to `knowledge/LearningDNA.md`
   d. Confirm: "Your Learning DNA is set! You can update it anytime or create topic-specific overrides."
3. If `knowledge/LearningDNA.md` exists → proceed to Step 2

### Step 2: Create Directory Structure
Create the following structure:
```
knowledge/{topic-name}/
  sources/
    overview.md
  quizzes/
    quiz-bank.json    # empty array: []
```

### Step 3: Generate overview.md
Generate `overview.md` shaped by the merged LearningDNA:
- Read `knowledge/LearningDNA.md` for the current DNA settings
- **Expert refresher** → concise topic map with advanced subtopics
- **Beginner** → detailed overview with prerequisites listed
- **Kids** → simple, fun introduction with everyday analogies
- **Content depth: Brief** → bullet-point overview
- **Content depth: Comprehensive** → full introduction with context and background
- The overview should list potential subtopics to research next

### Step 4: Per-Topic DNA Override
1. Ask the user: "Would you like to adjust your Learning DNA for {topic}? Your defaults are: [show current global values]"
2. If yes: ask the same 8 questions, but each shows the global default as the first option labeled "(Current default)"
3. If the user changes any answers: write `knowledge/{topic}/LearningDNA.md` with the full profile including changes
4. If no changes: no per-topic file is created, global defaults apply

### Step 5: Suggest Next Steps
Tell the user: "Topic '{topic}' is ready! Next steps:"
- `/learning-dna:research {topic} <subtopic>` — to research and summarize a subtopic
- `/learning-dna:add-quizzes {topic}` — to generate quiz questions (after researching subtopics)
