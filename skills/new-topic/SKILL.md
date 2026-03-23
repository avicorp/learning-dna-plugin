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
2. If missing → **STOP. Do not proceed.** Run the DNA Creation Flow using AskUserQuestion:

   a. Inform the user: "Before we create any topic, you need to set up your Learning DNA — this is your personal learner profile that shapes all content generated for you."

   b. **First AskUserQuestion call** — collect 4 answers at once:
      | # | header | question | options |
      |---|--------|----------|---------|
      | 1 | Language | What language should your learning content be in? | English — "Content in English" / Spanish — "Contenido en español" / Hebrew — "תוכן בעברית" / Arabic — "المحتوى بالعربية" |
      | 2 | Learner | Who is this for? | Kids (6-12) — "Simple language, fun analogies" / Youth (13-22) — "Engaging, school-oriented" / Professional (23-60) — "Industry terminology, career-focused" / Lifelong (60+) — "Curiosity-driven, clear structure" |
      | 3 | Depth | How deep should the content go? | Brief — "Bullet-points, key takeaways only" / Standard — "Balanced explanation with examples" / Detailed — "Thorough coverage with extra context" / Comprehensive — "Full theory, edge cases, deep dives" |
      | 4 | Examples | What kind of examples do you prefer? | Minimal — "Concepts only, few examples" / Code-focused — "Code samples in every section" / Real-world — "Practical, production scenarios" / Mixed — "Blend of code and real-world" |

   c. **Second AskUserQuestion call** — collect remaining 4 answers:
      | # | header | question | options |
      |---|--------|----------|---------|
      | 5 | Visuals | How many diagrams and visuals? | Low — "Diagrams only for complex topics" / Medium — "Diagrams for key concepts" / High — "Mermaid diagram for every major concept" |
      | 6 | Testing | How many quiz questions per topic? | Light — "3-5 questions per subtopic" / Standard — "5-10 questions per subtopic" / Heavy — "10-20 questions per subtopic" |
      | 7 | Level | What's your current knowledge level? | Beginner — "New to this, explain everything" / Some exposure — "Know the basics, build from there" / Working knowledge — "Skip basics, focus on application" / Expert refresher — "Advanced patterns and edge cases only" |
      | 8 | Goal | What's your learning goal? | Quick refresher — "Key facts and reminders" / Practical skills — "Hands-on, scenario-based" / Deep understanding — "Truly understand the why and how" / Interview prep — "System-design style, explain while coding" |

   d. Write all 8 answers to `knowledge/LearningDNA.md`
   e. Confirm: "Your Learning DNA is set!"

3. If `knowledge/LearningDNA.md` exists → proceed to Step 2

### Step 2: Per-Topic DNA Override
1. Use AskUserQuestion to ask: question="Would you like to adjust your Learning DNA for {topic}?", header="Override", options: "Keep defaults — {show current global summary}" / "Customize — Adjust settings for this topic"
2. If "Keep defaults" → no per-topic file is created, proceed to Step 3
3. If "Customize" → run the same two AskUserQuestion calls from Step 1, but mark each current global value with "(Current default)" at the end of its label. Parse the answers and write `knowledge/{topic}/LearningDNA.md` with the full profile including any changes.

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
1. Read and follow the full flow defined in `skills/inspect-repo/SKILL.md` (starting from Step 2 — the DNA gate is already done). **Do NOT use the Skill tool** to invoke it — `inspect-repo` has `disable-model-invocation: true`, so it must be read and executed inline.
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
Follow the full `build-app` skill specification (`skills/build-app/SKILL.md`), which uses a structure-first approach:
1. If `learning-app/` does not exist, set up the application structure, data model (React Context), data loader (IndexedDB), layout (navbar + sidebar + footer), and TTS component **before** scaffolding any topic pages (Steps 2-6 of build-app)
2. Parse content to JSON and scaffold topic UI pages (Steps 7-8 of build-app)
3. Create or update the welcome/home page to list all topics including the new one (Step 9 of build-app)
4. Run application validation to ensure the scaffolding is correct (Step 10 of build-app)
5. Run `cd learning-app && npm run build` to verify everything compiles (Step 11 of build-app)

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
   - Verify the welcome page at `/` lists the topic (and all other existing topics if any)
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

     Want to share it online? Deploy for free:
     /learning-dna:deploy
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
