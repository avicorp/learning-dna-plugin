---
name: research
description: Research and summarize a subtopic using web sources, shaped by LearningDNA
disable-model-invocation: true
argument-hint: <topic> <subtopic>
---

# /research — Research and Summarize a Subtopic

## Usage
`/learning-dna:research <topic> <subtopic>`

Example: `/learning-dna:research kubernetes networking`

## Flow

### Step 1: DNA Gate (MANDATORY)
1. Check if `knowledge/LearningDNA.md` exists — if missing, **STOP. Do not proceed.** Tell the user: "You need to set up your Learning DNA first. Run `/learning-dna:new-topic <topic>` to get started."
2. Read `knowledge/LearningDNA.md` (global DNA)
3. Check if `knowledge/{topic}/LearningDNA.md` exists (per-topic override)
4. Merge: per-topic values override global values → use merged DNA for all content generation

### Step 2: Verify Topic Exists
- Check that `knowledge/{topic}/` exists
- If not, suggest running `/learning-dna:new-topic {topic}` first

### Step 3: Web Research
1. Use WebSearch to find 10-15 high-quality sources: official docs, engineering blogs, books, academic papers
2. Present the sources to the user in a numbered list with title, URL, and brief description
3. **Wait for user approval** — never auto-approve sources. Ask: "Which sources should I use? (e.g., 1,3,5-8 or 'all')"

### Step 4: Fetch and Process Sources
1. Use WebFetch on approved sources
2. Extract key information, concepts, and examples

### Step 5: Structure Content Shaped by DNA
Write markdown following these DNA-driven rules:

**Content Depth:**
- `Brief` → 2-3 paragraphs per section, key takeaways only, bullet-point summaries
- `Standard` → balanced explanation with examples
- `Detailed` → thorough explanations with context and nuance
- `Comprehensive` → full explanations, background theory, trade-off analysis, edge cases

**Example Style:**
- `Minimal` → concepts only, few examples
- `Code-focused` → include code samples in every section
- `Real-world scenarios` → practical, production-oriented examples
- `Mixed` → blend of code and real-world examples

**Visualization:**
- `Low` → diagrams only for complex architectures
- `Medium` → diagrams for key concepts
- `High` → mermaid diagram for every major concept

**Learner Type:**
- `Kids` → simple language, playful tone, everyday analogies, short sections
- `Youth` → engaging, school/university oriented, relatable scenarios
- `Professional` → industry terminology, career-focused, production-ready
- `Lifelong` → curiosity-driven, clear structure, connecting to broad experience

**Knowledge Level:**
- `Beginner` → explain prerequisites, define all terms, build from basics
- `Some exposure` → brief refresher of basics, focus on building understanding
- `Working knowledge` → skip basics, focus on practical application
- `Expert refresher` → skip fundamentals, focus on edge cases and advanced patterns

**Language:**
- Generate all content in the specified language

**Markdown structure:**
- H1 title
- H2 numbered sections
- H3 subsections
- Mermaid diagrams in triple backticks with `mermaid` language tag
- `---` separators between major sections
- `## References` section at the end with source URLs

### Step 6: Write Output
- Write to `knowledge/{topic}/sources/{subtopic}.md`
- Update `knowledge/{topic}/sources/overview.md` to link the new subtopic

### Step 7: Dispatch Agents
- Dispatch the `topic-expander` agent to suggest what to research next
- The agent should analyze existing subtopics and suggest gaps

### Key Constraints
- Never auto-approve sources — always wait for user selection
- Always include `## References` with URLs
- Content style must match DNA settings — this is the most visible effect of the DNA system
