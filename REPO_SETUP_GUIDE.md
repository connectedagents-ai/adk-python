# Repository Setup Guide (GitHub + Claude Code + Codex)

## 1) Recommended Repository Structure

```
repo-root/
  .github/
    ISSUE_TEMPLATE/
      bug_report.md
      feature_request.md
    workflows/
      ci.yml
  docs/
    architecture/
    product/
    ux/
  src/
  tests/
  scripts/
  README.md
  CONTRIBUTING.md
  CODE_OF_CONDUCT.md
  SECURITY.md
  LICENSE
  AGENTS.md
```

### Notes
- **docs/** holds system, product, and UX documentation.
- **scripts/** includes automation (formatting, lint, build).
- **.github/** includes issue templates and CI workflows.

## 2) README Template (Example)

```markdown
# Project Name

## Overview
One-paragraph summary of the product and audience.

## Features
- Key feature 1
- Key feature 2
- Key feature 3

## Quick Start
```bash
# install
# run
```

## Project Structure
- `src/` application code
- `docs/` documentation
- `tests/` test suites

## Contributing
See CONTRIBUTING.md
```

## 3) CONTRIBUTING.md Template (Example)

```markdown
# Contributing

## Development Setup
- Install dependencies
- Run tests

## Branching
- feature/<name>
- fix/<name>

## Pull Requests
- Provide a summary
- Include tests
```

## 4) GitHub Issue Templates (Example)

### .github/ISSUE_TEMPLATE/bug_report.md
```markdown
---
name: Bug report
about: Report a bug
---

## Summary

## Steps to Reproduce

## Expected

## Actual
```

### .github/ISSUE_TEMPLATE/feature_request.md
```markdown
---
name: Feature request
about: Suggest a feature
---

## Summary

## Why This Is Needed
```

## 5) CI Workflow (Example)

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install
        run: ./scripts/install.sh
      - name: Test
        run: ./scripts/test.sh
```

## 6) Claude Code Setup

### Recommended Files
- **AGENTS.md**: Instructions and guardrails for Claude Code.
- **docs/architecture/**: System architecture docs.
- **docs/ux/**: Interaction guidelines and tone.

### Example AGENTS.md Content
```markdown
# Claude Code Instructions
- Follow repository conventions.
- Prefer minimal, testable changes.
- Document assumptions.
```

## 7) Codex Setup

### Recommended Files
- **AGENTS.md** or **CODEX.md** to describe workflow rules.
- **docs/product/** for feature intent and requirements.

### Example CODEX.md
```markdown
# Codex Instructions
- Keep diffs small and atomic.
- Reference relevant docs in PRs.
- Do not introduce new dependencies without approval.
```

## 8) Example Repository Checklist

- [ ] README.md
- [ ] CONTRIBUTING.md
- [ ] CODE_OF_CONDUCT.md
- [ ] SECURITY.md
- [ ] LICENSE
- [ ] .github/ISSUE_TEMPLATE/
- [ ] .github/workflows/ci.yml
- [ ] docs/architecture/
- [ ] docs/product/
- [ ] docs/ux/
```
