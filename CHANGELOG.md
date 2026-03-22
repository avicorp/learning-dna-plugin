# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-03-22

### Added
- Paragraph-level TTS read buttons — each `<p>` gets a speaker icon on hover, plus a "Read All" button for full-page reading
- TTS speed presets — "Slow & Clear" (0.7x), "Normal" (1x), "Fast" (1.5x), "Speed Reader" (2x) quick-select chips
- Volume control slider for TTS playback
- "Preview Voice" button — hear a sample sentence before selecting a voice
- Per-topic TTS settings persistence in localStorage
- Reading progress visual feedback — sentence highlighting, progress bar, and auto-scroll during TTS playback
- Keyboard shortcuts for TTS — Space (play/pause), Escape (stop), `[`/`]` (speed), N/P (navigate sections)
- DNA-driven TTS defaults — learner type sets default speed, language drives voice ranking, content depth affects pause behavior
- Side menu with progress tracking — collapsible sidebar showing subtopic completion, quiz scores, and overall progress
- Subtopic completion tracking via IntersectionObserver (scroll-to-bottom detection)
- Quiz score display in sidebar with difficulty breakdown

### Changed
- TTS voice ranking now prioritizes "Natural" > "Enhanced" > "Premium" > default system voices
- Build-app skill spec expanded with detailed TTS, keyboard shortcuts, and sidebar specifications

## [1.0.0] - 2026-03-17

### Added
- Initial release of the Learning DNA plugin for Claude Code
- `/learning-dna:new-topic` skill — create topics with mandatory DNA setup
- `/learning-dna:research` skill — research subtopics from web sources
- `/learning-dna:add-quizzes` skill — generate quizzes shaped by DNA
- `/learning-dna:build-app` skill — scaffold React learning app
- `content-reviewer` agent — validates learning materials quality
- `quiz-improver` agent — improves quiz question quality
- `topic-expander` agent — suggests subtopics and learning paths
- Pre-commit review hook with 3-iteration quality loop
- 8-dimension Learning DNA system (Language, Learner Type, Content Depth, Example Style, Visualization, Testing Frequency, Knowledge Level, Learning Goal)
