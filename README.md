# Learning DNA Plugin for Claude Code

A Claude Code plugin that turns any Claude Code project into a personalized learning platform. Define your learner profile (your "Learning DNA"), pick any topic, and Claude researches, summarizes, and builds quizzes — all adapted to your learning style, age, and language.

## What It Does

1. **You set up your Learning DNA** — answer 8 questions about how you learn best (language, age group, depth, examples, etc.)
2. **You pick a topic** — anything: Kubernetes, biology, music theory, cooking
3. **Claude researches it** — finds sources, lets you approve them, then writes content shaped by your DNA
4. **Claude generates quizzes** — difficulty and style matched to your profile
5. **Claude builds a learning app** — a React web app with your content and quizzes, ready to deploy

No defaults. No one-size-fits-all. Your DNA drives every piece of generated content.

## Requirements

- [Claude Code](https://code.claude.com) v1.0.33 or later

## Installation

### From a marketplace

```bash
# Add the marketplace (once)
/plugin marketplace add avilevy/learning-dna-plugin

# Install the plugin
/plugin install learning-dna
```

### Local development

```bash
git clone https://github.com/avilevy/learning-dna-plugin.git
claude --plugin-dir ./learning-dna-plugin
```

## Usage

### 1. Create your first topic

```
/learning-dna:new-topic kubernetes
```

On first use, you'll be asked 8 questions to build your Learning DNA:

| Dimension | Options |
|-----------|---------|
| Language | English, Spanish, Hebrew, Chinese, Arabic, Other |
| Learner Type | Kids (6-12), Youth (13-22), Professional (23-60), Lifelong (60+) |
| Content Depth | Brief, Standard, Detailed, Comprehensive |
| Example Style | Minimal, Code-focused, Real-world scenarios, Mixed |
| Visualization | Low, Medium, High |
| Testing Frequency | Light (3-5/topic), Standard (5-10), Heavy (10-20) |
| Knowledge Level | Beginner, Some exposure, Working knowledge, Expert refresher |
| Learning Goal | Quick refresher, Practical skills, Deep understanding, Interview prep |

### 2. Research subtopics

```
/learning-dna:research kubernetes networking
```

Claude searches the web, presents sources for your approval, then writes content shaped by your DNA.

### 3. Generate quizzes

```
/learning-dna:add-quizzes kubernetes
```

Generates quiz questions from your researched material — difficulty distribution, question style, and language all driven by your DNA.

### 4. Build the learning app

```
/learning-dna:build-app
```

Scaffolds a React 19 + TypeScript + Vite + Tailwind + Mermaid web app from your knowledge base.

## How DNA Shapes Content

| DNA Setting | Effect |
|-------------|--------|
| `Language: Hebrew` | All content, quizzes, and explanations in Hebrew |
| `Learner Type: Kids` | Simple language, playful tone, everyday analogies, gamified quizzes |
| `Learner Type: Professional` | Industry terminology, production-ready examples |
| `Content Depth: Brief` | Bullet-point summaries, skip theory |
| `Content Depth: Comprehensive` | Full explanations with background theory |
| `Visualization: High` | Mermaid diagram for every major concept |
| `Testing: Heavy` | 10-20 questions per subtopic across all difficulties |
| `Knowledge Level: Expert` | Skip basics, focus on nuances and edge cases |

## Plugin Structure

```
learning-dna-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   ├── new-topic/SKILL.md       # Create topics + DNA setup
│   ├── research/SKILL.md        # Research subtopics from web
│   ├── add-quizzes/SKILL.md     # Generate quizzes
│   └── build-app/SKILL.md       # Build the learning app
├── agents/
│   ├── content-reviewer.md      # Validates content quality
│   ├── quiz-improver.md         # Improves quiz questions
│   └── topic-expander.md        # Suggests subtopics
└── hooks/
    └── hooks.json               # Pre-commit quality checks
```

## What Gets Created in Your Project

When you use the plugin, it creates these in your project directory:

```
your-project/
├── knowledge/
│   ├── LearningDNA.md                  # Your learner profile
│   └── {topic}/
│       ├── LearningDNA.md              # Optional per-topic override
│       ├── sources/
│       │   ├── overview.md
│       │   └── {subtopic}.md
│       └── quizzes/
│           └── quiz-bank.json
└── learning-app/                        # Generated React app
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

[MIT](LICENSE)
