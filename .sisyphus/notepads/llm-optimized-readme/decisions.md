# Decisions

This file records architectural choices, design decisions, and their rationales for the llm-optimized-readme task.

## 2026-02-02T10:13:28Z Task: init-notepad

Initialized notepad for llm-optimized-readme. This file will document why certain approaches were chosen over alternatives.

## 2026-02-02T10:15:00Z Task: Create README Structure and Frontmatter

### Decision: Three-Section README Structure

Chose a three-section hierarchy to serve different audiences:

1. **## TL;DR** - For humans who need the 30-second overview
   - One-line summary of the stack
   - Key URLs for quick access
   - Minimal verification commands
   - No deep context, just orientation

2. **## For Humans** - Complete operational reference
   - Full architecture flow
   - All URLs, files, namespaces
   - Secrets (names only, no values)
   - Complete verification commands
   - Operational notes and gotchas
   - Preserved all existing content from original README

3. **## For LLM Agents** - Machine-parseable onboarding data
   - Template variables block ({{APP_NAME}}, {{GITHUB_USER}}, etc.)
   - Decision/inputs block with structured data
   - Preserved the original one-paragraph LLM instruction as a code block
   - Key data points extracted for easy parsing

### Rationale

- **Separation of concerns:** Humans and LLMs consume information differently
- **Template variables:** Enable programmatic substitution when onboarding new apps
- **Preserved existing wisdom:** The original LLM paragraph contained dense, valuable context; kept it intact
- **Copy/paste friendly:** URLs, commands, and paths are inline and unambiguous

### Structure Pattern

```
# Title

## TL;DR
## For Humans
## For LLM Agents
```

This pattern scales well and keeps each section focused on its audience.
