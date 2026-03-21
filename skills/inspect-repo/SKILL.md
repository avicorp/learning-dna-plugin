---
name: inspect-repo
description: Inspect a GitHub repo and create a knowledge base from its README, recent commits, and PR reviews
disable-model-invocation: true
argument-hint: <github-repo-url-or-owner/repo> [topic-name]
---

# /inspect-repo — Inspect a GitHub Repo

## Usage
`/learning-dna:inspect-repo <github-repo-url-or-owner/repo> [topic-name]`

Examples:
- `/learning-dna:inspect-repo https://github.com/anthropics/claude-code`
- `/learning-dna:inspect-repo anthropics/claude-code`
- `/learning-dna:inspect-repo anthropics/claude-code my-claude-study`

## Flow

> **Self-driving pipeline:** After each step completes, proceed immediately to the next. Do NOT pause between steps. The only user interaction point is the DNA gate (Step 1). All other transitions are automatic.

### Step 1: DNA Gate (MANDATORY)
1. Check if `knowledge/LearningDNA.md` exists — if missing, **STOP. Do not proceed.** Tell the user: "You need to set up your Learning DNA first. Run `/learning-dna:new-topic <topic>` to get started."
2. Read `knowledge/LearningDNA.md` (global DNA)
3. Check if `knowledge/{topic}/LearningDNA.md` exists (per-topic override)
4. Merge: per-topic values override global values → use merged DNA for all content generation

### Step 2: Parse Input & Validate
1. Accept any of these formats:
   - `https://github.com/owner/repo`
   - `github.com/owner/repo`
   - `owner/repo`
2. Extract `{owner}` and `{repo}` from the input
3. Derive topic name: use the provided `[topic-name]` argument, or default to the repo name (e.g., `claude-code`)
4. Validate the repo exists:
   ```bash
   gh api repos/{owner}/{repo} --jq '.full_name'
   ```
   - If 404 → **STOP.** Tell the user: "Repository {owner}/{repo} not found. Check the URL and your GitHub authentication (`gh auth status`)."
5. Create directory: `knowledge/{topic}/sources/`

### Step 3: Fetch Repo Docs → `repo-goals.md`
1. Fetch README content:
   ```bash
   gh api repos/{owner}/{repo}/readme --jq '.content' | base64 --decode
   ```
2. Fetch repo metadata:
   ```bash
   gh api repos/{owner}/{repo} --jq '{description, topics, homepage, language}'
   ```
3. Fetch root-level markdown files (max 5) via tree API:
   ```bash
   gh api "repos/{owner}/{repo}/git/trees/HEAD" --jq '.tree[] | select(.path | test("\\.(md|MD)$")) | select(.path | test("/") | not) | .path' | head -5
   ```
   For each discovered file (excluding README which is already fetched):
   ```bash
   gh api "repos/{owner}/{repo}/contents/{filename}" --jq '.content' | base64 --decode
   ```
4. Dispatch `repo-analyst` sub-agent with:
   - All pre-fetched content (README, metadata, extra markdown files)
   - The merged LearningDNA settings
   - Output file path: `knowledge/{topic}/sources/repo-goals.md`
   - Instruction: write about purpose, architecture overview, key technologies, target audience
   - **Sub-agent must NOT call GitHub API or any external tools — only process provided data and write the file**

### Step 4: Fetch Recent Commits → `recent-changes.md`
1. Compute the date 2 weeks ago (ISO 8601 format)
2. Probe commit count:
   ```bash
   gh api "repos/{owner}/{repo}/commits?since={2-weeks-ago}&per_page=100" --jq 'length'
   ```
3. **Safeguard branching:**
   - **< 50 commits:** Fetch commit list with messages, dates, and authors:
     ```bash
     gh api "repos/{owner}/{repo}/commits?since={2-weeks-ago}&per_page=50" --jq '.[] | {sha: .sha[0:7], message: .commit.message, date: .commit.author.date, author: .commit.author.name}'
     ```
   - **50+ commits:** Use compare API for diff stat summary. Log to the user: "Large repo — using batched summary mode."
     ```bash
     gh api "repos/{owner}/{repo}/compare/{2-weeks-ago-sha}...HEAD" --jq '{total_commits: .total_commits, files: [.files[:20][] | {filename, changes, additions, deletions}]}'
     ```
     No per-commit inspection. Never paginate beyond page 1.
4. Dispatch `repo-analyst` sub-agent with:
   - All pre-fetched commit data
   - The merged LearningDNA settings
   - Output file path: `knowledge/{topic}/sources/recent-changes.md`
   - Instruction: write about focus areas, new capabilities (feat/add commits), fixes (fix/bug commits), lessons learned
   - **Sub-agent must NOT call GitHub API or any external tools**

### Step 5: Fetch PR Reviews → `pr-reviews.md`
1. Fetch merged PRs (last 2 weeks, max 30):
   ```bash
   gh api "repos/{owner}/{repo}/pulls?state=closed&sort=updated&direction=desc&per_page=30" --jq '[.[] | select(.merged_at != null) | {number, title, user: .user.login, merged_at, body}]'
   ```
2. For each PR (max 20 from the results), fetch reviews and review comments:
   ```bash
   gh api "repos/{owner}/{repo}/pulls/{pr_number}/reviews" --jq '[.[] | {user: .user.login, state, body}]'
   ```
   ```bash
   gh api "repos/{owner}/{repo}/pulls/{pr_number}/comments?per_page=50" --jq '[.[] | {user: .user.login, body, path, diff_hunk}]'
   ```
   - Single page only per PR — never paginate
3. Dispatch `repo-analyst` sub-agent with:
   - All pre-fetched PR and review data
   - The merged LearningDNA settings
   - Output file path: `knowledge/{topic}/sources/pr-reviews.md`
   - Instruction: write about main reviewers, recurring feedback themes, most important architectural/quality knowledge
   - **Sub-agent must NOT call GitHub API or any external tools**

> **Steps 3, 4, and 5 — API fetching is sequential** (to track the call budget), but the three `repo-analyst` sub-agent dispatches at the end can run **in parallel** since they only process pre-fetched data.

### Step 6: Generate Overview
Write or update `knowledge/{topic}/sources/overview.md`:
```markdown
# {Topic Name} — Repository Knowledge Base

Overview of knowledge extracted from [{owner}/{repo}](https://github.com/{owner}/{repo}).

## Subtopics
1. [Repo Goals & Architecture](repo-goals.md) — purpose, design, and technologies
2. [Recent Changes](recent-changes.md) — what's been happening in the last 2 weeks
3. [PR Review Insights](pr-reviews.md) — code review patterns and quality themes
```

Shape the overview language and style according to the merged LearningDNA.

### Step 7: Dispatch Content Reviewer
Dispatch the `content-reviewer` agent to validate all 3 source files:
- `knowledge/{topic}/sources/repo-goals.md`
- `knowledge/{topic}/sources/recent-changes.md`
- `knowledge/{topic}/sources/pr-reviews.md`

Include the merged LearningDNA in the agent prompt. If the reviewer finds auto-fixable issues, apply them.

### Step 8: Report
Print a summary:
```
Repo inspection complete for {owner}/{repo}!

  Files created:
    - knowledge/{topic}/sources/repo-goals.md
    - knowledge/{topic}/sources/recent-changes.md
    - knowledge/{topic}/sources/pr-reviews.md
    - knowledge/{topic}/sources/overview.md

  Stats:
    - Commits analyzed: {count}
    - PRs inspected: {count}
    - GitHub API calls used: {count}/50

  Next steps:
    - /learning-dna:add-quizzes {topic}  — generate quiz questions from this knowledge
    - /learning-dna:build-app {topic}    — build a learning app
    - /learning-dna:research {topic} <subtopic> — add web research on a specific area
```

## Performance Constraint
**Total `gh api` calls must not exceed 50.** Track the running count throughout the flow. If approaching the limit, stop fetching and summarize what has been collected so far. Sub-agents do text processing only — no API calls.

## Key Constraints
- **Never skip the DNA gate** — content must be shaped by the learner's profile
- **Sub-agents never call APIs** — all GitHub data is pre-fetched by the main agent
- **Never paginate beyond page 1** — single-page fetches only
- **50-commit safeguard** — large repos get batched summary mode, not per-commit inspection
- **API budget of 50 calls** — stop and summarize if approaching the limit
