---
name: new-topic
description: Create a complete learning topic — DNA setup, research, quizzes, build, and serve a working app on localhost
disable-model-invocation: true
argument-hint: <topic-name>
---

# /new-topic — Create a Complete Learning Topic

## Usage
`/learning-dna:new-topic <topic-name>`

Example: `/learning-dna:new-topic kubernetes`

This is the main entry point. It runs the **full pipeline** end-to-end and finishes with a working learning app served on localhost.

## Flow

> **Self-driving pipeline:** This is an end-to-end pipeline. After each step completes, proceed immediately to the next. Do NOT pause between steps unless the step explicitly requires user input. Mandatory user interaction points: DNA questions (Step 1), per-topic override (Step 2), source approval (Step 5), quiz approval (Step 6). All other transitions are automatic.

### Step 1: DNA Gate (MANDATORY)
1. Check if `knowledge/LearningDNA.md` exists
2. If missing → **STOP. Do not proceed.** Run the DNA Creation Flow first:
   a. Inform the user: "Before we create any topic, you need to set up your Learning DNA — this is your personal learner profile that shapes all content generated for you. There are no defaults — your answers drive everything."
   b. Present all 8 questions in a **single message** as a numbered list with multiple-choice options. **Do not split across multiple interactions. Do not skip any question. Do not use defaults. The user must answer each one:**
      1. **Language:** English / Spanish / Hebrew / Chinese / Arabic / Other (specify)
      2. **Learner Type:** Kids (6-12) / Youth (13-22) / Professional (23-60) / Lifelong (60+)
      3. **Content Depth:** Brief / Standard / Detailed / Comprehensive
      4. **Example Style:** Minimal / Code-focused / Real-world scenarios / Mixed
      5. **Visualization:** Low / Medium / High
      6. **Testing Frequency:** Light (3-5 per topic) / Standard (5-10) / Heavy (10-20)
      7. **Knowledge Level:** Beginner / Some exposure / Working knowledge / Expert refresher
      8. **Learning Goal:** Quick refresher / Practical skills / Deep understanding / Interview prep
   c. Wait for the user to reply with all their answers in a single response
   d. Parse the user's answers and write the results to `knowledge/LearningDNA.md`
   e. Confirm: "Your Learning DNA is set!"
3. If `knowledge/LearningDNA.md` exists → proceed to Step 2

### Step 2: Per-Topic DNA Override
1. Ask the user: "Would you like to adjust your Learning DNA for {topic}? Your defaults are: [show current global values]"
2. If yes: present all 8 questions in a **single message**, each showing the global default as the first option labeled "(Current default)". Wait for the user to reply with all answers at once.
3. Parse the user's answers. If any differ from defaults: write `knowledge/{topic}/LearningDNA.md` with the full profile including changes
4. If no changes: no per-topic file is created, global defaults apply

### Step 3: Create Directory Structure
Create the following structure:
```
knowledge/{topic-name}/
  sources/
    overview.md
  quizzes/
    quiz-bank.json    # empty array: []
```

### Step 4: Generate Overview
Generate `overview.md` shaped by the merged LearningDNA:
- Read the DNA settings (global + per-topic override if exists)
- **Expert refresher** → concise topic map with advanced subtopics
- **Beginner** → detailed overview with prerequisites listed
- **Kids** → simple, fun introduction with everyday analogies
- **Content depth: Brief** → bullet-point overview
- **Content depth: Comprehensive** → full introduction with context and background
- The overview MUST list 3-5 subtopics to research — these drive Step 5

### Step 4.5: Repo Detection
If the topic argument contains a GitHub URL (`https://github.com/owner/repo`, `github.com/owner/repo`) or an `owner/repo` pattern, OR the user explicitly mentions a GitHub repository:
1. Run `/learning-dna:inspect-repo` with the repo URL/identifier and the topic name
2. The generated files (`repo-goals.md`, `recent-changes.md`, `pr-reviews.md`) become the first 3 subtopics in the knowledge base
3. Update `overview.md` to include these as the first subtopics
4. Proceed to Step 5 for any additional web research the user wants

If the topic is not repo-related, skip this step entirely.

### Step 5: Research All Subtopics
The **main agent** orchestrates all research — sub-agents only structure and write content.

1. **Main agent searches:** Use WebSearch to find 10-15 high-quality sources per subtopic
2. **User approval:** Present all sources to the user and **wait for approval** — never auto-approve
3. **Main agent fetches:** Use WebFetch on all approved sources to retrieve their content
4. **Dispatch sub-agents in parallel** — one per subtopic. Each sub-agent's prompt must include:
   - The **fetched source content** (already approved and retrieved by main agent)
   - The **merged LearningDNA settings** (global + per-topic override if exists)
   - The **content structuring rules** (inline in the prompt):
     - **Content Depth**: Brief → bullet-points; Comprehensive → full theory + edge cases
     - **Example Style**: Minimal → concepts only; Code-focused → code in every section; Real-world → production examples
     - **Visualization**: Low → few diagrams; High → mermaid diagram for every major concept
     - **Learner Type**: Kids → playful tone, analogies; Professional → industry terminology; etc.
     - **Knowledge Level**: Beginner → define all terms; Expert → skip basics, focus on nuances
     - **Language**: All content in the specified language
     - Markdown structure: H1 title, H2 sections, H3 subsections, `---` separators, `## References` at end
   - The **output file path**: `knowledge/{topic}/sources/{subtopic}.md`
   - **Important:** Sub-agents do NOT use WebSearch, WebFetch, or the Skill tool — they only structure the provided content and write the output file
5. Update `overview.md` to link each completed subtopic

After all subtopics are researched, dispatch the `topic-expander` agent to suggest additional subtopics. Ask the user if they want to research any of the suggestions. If yes, repeat this step for each additional subtopic.

### Step 6: Generate Quizzes
Generate quiz questions from all researched source material:

1. Read all markdown files from `knowledge/{topic}/sources/`
2. Generate questions shaped by DNA:
   - **Testing Frequency**: Light → 3-5 per subtopic; Standard → 5-10; Heavy → 10-20
   - **Difficulty distribution** based on Knowledge Level:
     - Beginner → 60% easy, 30% medium, 10% hard
     - Some exposure → 40% easy, 40% medium, 20% hard
     - Working knowledge → 20% easy, 40% medium, 40% hard
     - Expert refresher → 10% easy, 30% medium, 60% hard
   - **Learning Goal**: Refresher → key facts; Practical → scenarios; Deep → synthesis; Interview → system-design style
   - **Learner Type**: Kids → gamified, fun; Youth → school-exam style; Professional → industry scenarios
   - **Language**: All questions, options, and explanations in the specified language
3. Quiz JSON schema per question:
   ```json
   {
     "id": "{topic}-{subtopic}-{number}",
     "topicSlug": "{topic}",
     "question": "string",
     "options": ["4 answer choices"],
     "correctIndex": 0,
     "explanation": "educational explanation",
     "difficulty": "easy|medium|hard"
   }
   ```
4. Present questions to user grouped by difficulty — wait for approval
5. Write to `knowledge/{topic}/quizzes/quiz-bank.json`
6. Dispatch the `quiz-improver` agent to review and enhance quality
7. Report: "Generated X questions: Y easy, Z medium, W hard"

### Step 7: Build the Learning App
1. If `learning-app/` does not exist, scaffold a new project:
   - React 19 + TypeScript + Vite 6 + Tailwind CSS 4 + Mermaid 11
   - Base layout with navigation, routing, progress tracking
   - Home page listing available topics
2. Create a parse script in `learning-app/scripts/` to convert knowledge markdown → JSON
3. Run the parse script to generate `learning-app/src/data/{topic}-topics.json` and `learning-app/src/data/{topic}-quizzes.json`
4. Scaffold topic UI pages if they don't exist:
   - Topic home page (subtopic list with progress)
   - Topic detail page (rendered markdown + mermaid diagrams)
   - Quiz page (interactive quiz with scoring and explanations)
5. Run `cd learning-app && npm run build` to verify everything compiles

### Step 8: Serve and Verify (LOOP UNTIL SUCCESS)

This step runs a verification loop. **The flow is NOT complete until the app is confirmed running and accessible.** Maximum 5 attempts before asking the user for help.

**Attempt loop (max 5 iterations):**

1. **Build check:**
   ```bash
   cd learning-app && npm run build 2>&1
   ```
   - If build fails → read the error output, diagnose the issue, fix the code, and restart this step
   - If build succeeds → proceed to serve

2. **Start the dev server** (run in background):
   ```bash
   cd learning-app && npm run dev &
   ```

3. **Wait for server startup** (3 seconds), then **verify the app is accessible:**
   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://localhost:5173
   ```
   - If HTTP 200 → proceed to content verification
   - If connection refused or non-200 → check if the port is different (read the dev server output for the actual URL), retry with the correct port
   - If still failing → kill the server process, diagnose the issue, fix it, and restart this step

4. **Verify content is served correctly:**
   ```bash
   curl -s http://localhost:5173 | head -50
   ```
   - Confirm the HTML contains the app (look for the root div, script tags, app title)
   - If the page is blank or shows an error → diagnose, fix, rebuild, and restart this step

5. **Verify topic data exists in the built app:**
   - Check that `learning-app/src/data/{topic}-topics.json` exists and is valid JSON
   - Check that `learning-app/src/data/{topic}-quizzes.json` exists and is valid JSON
   - If either is missing or invalid → regenerate the data, rebuild, and restart this step

6. **If all checks pass → SUCCESS.** Report:
   ```
   ✓ Learning app verified and running!
     Topic: {topic}
     Subtopics researched: {count}
     Quiz questions: {count}
     App URL: http://localhost:{port}

     Open the URL in your browser to start learning.
   ```

7. **If attempt fails → increment counter and loop back to step 1.**
   - On each retry, report what failed and what was fixed
   - After 5 failed attempts, stop and report:
     ```
     ✗ Could not get the app running after 5 attempts.
       Last error: {description}
       Files created: knowledge/{topic}/ (sources + quizzes)
       App location: learning-app/

       Please check the errors above. You can retry with:
       /learning-dna:build-app {topic}
     ```

## Key Constraints
- **Never skip the DNA gate** — no defaults, user must answer all 8 questions
- **Never auto-approve sources** — always wait for user selection in Step 5
- **Never declare success without verification** — the loop MUST confirm HTTP 200 and valid content before reporting success
- **Always fix forward** — when a check fails, diagnose and fix the issue rather than skipping it
- **Content must match DNA** — this is the core value proposition, every piece of content reflects the learner's profile
