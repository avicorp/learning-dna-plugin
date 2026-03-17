# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in this plugin, please report it responsibly.

**Do not open a public issue.**

Instead, email the maintainer directly or use GitHub's private vulnerability reporting feature.

## Scope

This plugin generates markdown content and quiz JSON files. It also instructs Claude to:
- Fetch web content (via WebSearch/WebFetch) with user approval
- Write files to the `knowledge/` and `learning-app/` directories
- Run build commands (`npm run build`)

Please report if you find cases where the plugin could:
- Write files outside the expected directories
- Execute unexpected commands
- Expose sensitive information in generated content
