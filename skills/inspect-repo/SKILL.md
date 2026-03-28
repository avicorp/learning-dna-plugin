---
name: inspect-repo
description: Inspect a GitHub repo and create a knowledge base from its README, recent commits, and PR reviews
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

### Step 2: Parse Input & Clone
1. Accept any of these formats:
   - `https://github.com/owner/repo`
   - `github.com/owner/repo`
   - `owner/repo`
2. Extract `{owner}` and `{repo}` from the input
3. Derive topic name: use the provided `[topic-name]` argument, or default to the repo name (e.g., `claude-code`)
4. Clone the repo into `repo-inspection/{repo}`:
   ```bash
   git clone --depth 100 https://github.com/{owner}/{repo}.git repo-inspection/{repo}
   ```
   - `--depth 100` keeps the clone shallow — enough history for analysis without downloading everything
   - If the clone fails → **STOP.** Tell the user: "Could not clone {owner}/{repo}. Check the URL and that the repo is accessible."
5. Create directory: `knowledge/{topic}/sources/`

### Step 3: Read Repo Docs → `repo-goals.md`
All reads are from the local clone at `repo-inspection/{repo}/`.

1. Read the README file (look for `README.md`, `readme.md`, `README`, `README.rst`):
   ```bash
   ls repo-inspection/{repo}/README* repo-inspection/{repo}/readme* 2>/dev/null
   ```
   Then use the Read tool on the found file.

2. Read root-level markdown files (max 5, excluding README):
   ```bash
   ls repo-inspection/{repo}/*.md 2>/dev/null | head -5
   ```
   Use the Read tool on each discovered file (CONTRIBUTING.md, CHANGELOG.md, etc.)

3. Read `package.json`, `Cargo.toml`, `go.mod`, `pyproject.toml`, or similar manifest to extract dependencies and metadata:
   ```bash
   ls repo-inspection/{repo}/package.json repo-inspection/{repo}/Cargo.toml repo-inspection/{repo}/go.mod repo-inspection/{repo}/pyproject.toml 2>/dev/null
   ```

4. Get a directory tree overview (depth 2):
   ```bash
   find repo-inspection/{repo} -maxdepth 2 -type f | grep -v '.git/' | head -80
   ```

5. Dispatch `repo-analyst` sub-agent with:
   - All content read above (README, markdown files, manifest, directory tree)
   - The merged LearningDNA settings
   - Output file path: `knowledge/{topic}/sources/repo-goals.md`
   - Instruction: write about purpose, architecture overview, key technologies, target audience
   - **Sub-agent must NOT read files or run commands — only process provided data and write the output file**

### Step 4: Read Git History → `recent-changes.md`
All git commands run inside the local clone.

1. Count recent commits (last 2 weeks):
   ```bash
   git -C repo-inspection/{repo} log --since="2 weeks ago" --oneline | wc -l
   ```

2. **Safeguard branching:**
   - **< 50 commits:** Get full commit details:
     ```bash
     git -C repo-inspection/{repo} log --since="2 weeks ago" --format="%h %ad %an: %s" --date=short
     ```
   - **50+ commits:** Get a summary with file-level stats. Log to the user: "Large repo — using summary mode."
     ```bash
     git -C repo-inspection/{repo} log --since="2 weeks ago" --format="%h %ad %an: %s" --date=short | head -20
     git -C repo-inspection/{repo} diff --stat HEAD~50
     ```

3. Get the diff stat for the period to see which files changed most:
   ```bash
   git -C repo-inspection/{repo} log --since="2 weeks ago" --pretty=format: --name-only | sort | uniq -c | sort -rn | head -20
   ```

4. Dispatch `repo-analyst` sub-agent with:
   - All commit history and file change data collected above
   - The merged LearningDNA settings
   - Output file path: `knowledge/{topic}/sources/recent-changes.md`
   - Instruction: write about focus areas, new capabilities (feat/add commits), fixes (fix/bug commits), lessons learned
   - **Sub-agent must NOT read files or run commands**

### Step 5: Read PR Reviews → `pr-reviews.md`
PR reviews are GitHub-specific and not available in the local clone — use `gh` CLI for this step only.

1. Fetch recently merged PRs (max 20):
   ```bash
   gh pr list --repo {owner}/{repo} --state merged --limit 20 --json number,title,author,mergedAt,body
   ```

2. For each PR (max 15), fetch reviews and comments:
   ```bash
   gh pr view {pr_number} --repo {owner}/{repo} --json reviews,comments
   ```

3. Dispatch `repo-analyst` sub-agent with:
   - All PR and review data collected above
   - The merged LearningDNA settings
   - Output file path: `knowledge/{topic}/sources/pr-reviews.md`
   - Instruction: write about main reviewers, recurring feedback themes, most important architectural/quality knowledge
   - **Sub-agent must NOT run commands or call APIs**

> **Parallel dispatch:** The three `repo-analyst` sub-agent calls (from Steps 3, 4, and 5) can run **in parallel** since they only process already-collected data.

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

### Step 8: Cleanup & Report
1. Remove the cloned repo to save disk space:
   ```bash
   rm -rf repo-inspection/{repo}
   ```

2. Print a summary:
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

     Next steps:
       - /learning-dna:add-quizzes {topic}  — generate quiz questions from this knowledge
       - /learning-dna:build-app {topic}    — build a learning app
       - /learning-dna:research {topic} <subtopic> — add web research on a specific area
   ```

## Key Constraints
- **Clone, don't API** — use the local clone for all repo content and git history; only use `gh` CLI for PR reviews (which aren't in the clone)
- **Never skip the DNA gate** — content must be shaped by the learner's profile
- **Sub-agents never run commands** — all data is collected by the main agent and passed to sub-agents as text
- **Shallow clone** — `--depth 100` keeps it fast; do not fetch full history
- **50-commit safeguard** — large repos get summary mode, not per-commit inspection
- **Cleanup** — always remove the cloned repo after inspection is complete
