# Contributing to Learning DNA Plugin

Thank you for your interest in contributing! This guide will help you get started.

## How to Contribute

### Reporting Bugs

- Open an issue on GitHub with a clear title and description
- Include steps to reproduce the problem
- Mention your Claude Code version (`claude --version`)

### Suggesting Features

- Open an issue with the `enhancement` label
- Describe the use case and how it fits with the Learning DNA system
- If proposing a new DNA dimension, explain how it would shape content differently

### Submitting Changes

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Make your changes
4. Test locally with `claude --plugin-dir ./learning-dna-plugin`
5. Commit your changes with a descriptive message
6. Push to your fork and open a pull request

### Development Setup

```bash
git clone https://github.com/your-username/learning-dna-plugin.git
cd learning-dna-plugin

# Test the plugin locally
claude --plugin-dir .
```

### What to Work On

- **Skills**: Improve existing skills or add new ones in `skills/`
- **Agents**: Enhance agent logic in `agents/`
- **DNA dimensions**: Propose new Learning DNA dimensions
- **Language support**: Improve content generation for specific languages
- **Documentation**: Fix typos, improve examples, add guides

## Code Style

- Skills use YAML frontmatter + Markdown
- Keep `SKILL.md` files focused and under 500 lines
- Agent files follow the same frontmatter + markdown pattern
- Use clear section headers and structured output formats

## Pull Request Guidelines

- Keep PRs focused on a single change
- Update the README if your change affects usage
- Test your changes with `claude --plugin-dir` before submitting
- Describe what your PR does and why
