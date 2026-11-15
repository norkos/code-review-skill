# Contributing to AI Code Review Guide

Thank you for your interest in contributing! This document provides guidelines for contributing to this Claude Code Skill project.

## Claude Code Skill Development Guidelines

This project is a Claude Code Skill. Contributors should follow the conventions below.

### Directory Structure

```
code-review-skill/
├── SKILL.md                    # Required: main file (always loaded)
├── README.md
├── CONTRIBUTING.md
├── LICENSE
├── reference/                  # On-demand language/framework guides
│   ├── react.md                # React 19 / Next.js / TanStack Query v5
│   ├── vue.md                  # Vue 3.5 Composition API
│   ├── angular.md              # Angular 17+, Signals, Standalone, RxJS
│   ├── svelte.md               # Svelte 5 / SvelteKit, runes, SSR boundary
│   ├── rust.md                 # Ownership, async, unsafe, cancellation
│   ├── typescript.md           # Type safety, generics, strict mode
│   ├── nestjs.md               # NestJS DI, modules, Guards/Pipes, DTOs
│   ├── python.md               # Type hints, async, testing
│   ├── django.md               # Django / DRF, N+1, serializers, async views
│   ├── fastapi.md              # FastAPI, Depends, Pydantic v2, async
│   ├── java.md                 # Java 17/21, Spring Boot 3, virtual threads
│   ├── java8.md                # Java 8, Spring Boot 2, javax.*, legacy stack
│   ├── kotlin.md               # Kotlin / Android, coroutines, Flow, Compose
│   ├── swift.md                # Swift 5.9+/6, SwiftUI, concurrency, optionals
│   ├── dart.md                 # Dart / Flutter rebuilds, null safety, isolates, state
│   ├── go.md                   # Error handling, goroutines, context
│   ├── csharp.md               # C# / .NET 8, async, EF Core, ASP.NET Core
│   ├── php.md                  # PHP 8.x, types, PDO, security, Composer
│   ├── ruby.md                 # Ruby 3.4+/4.0, Rails 8.x, Active Record/Job
│   ├── c.md                    # Memory safety, UB, error handling
│   ├── cpp.md                  # RAII, move semantics, exception safety
│   ├── zig.md                  # Allocators, error unions, comptime, C interop
│   ├── qt.md                   # Object model, signals/slots, GUI perf
│   ├── css-less-sass.md        # Variables, responsive, performance
│   ├── architecture-review-guide.md  # SOLID, anti-patterns, coupling
│   ├── performance-review-guide.md   # Web Vitals, N+1, complexity
│   ├── security-review-guide.md      # OWASP Top 10, JWT, validation
│   ├── common-bugs-checklist.md      # Quick-reference bug patterns
│   ├── code-quality-universal.md      # Language-agnostic quality anti-patterns
│   ├── code-review-best-practices.md # Communication & process
│   └── cross-cutting/               # Language-agnostic cross-cutting patterns
│       ├── sql-injection-prevention.md # Parameterized queries, 6 languages
│       ├── xss-prevention.md           # Output encoding, CSP, 5 frameworks
│       ├── n-plus-one-queries.md       # N+1 queries, eager loading, 5 languages
│       ├── error-handling-principles.md # Error handling principles, 7 languages
│       └── async-concurrency-patterns.md # Concurrency patterns, 7 languages
├── assets/                     # Templates and quick reference
│   ├── review-checklist.md
│   └── pr-review-template.md
└── scripts/
    └── pr-analyzer.py          # PR complexity analyzer
```

### Frontmatter Requirements

SKILL.md must include YAML frontmatter:

```yaml
---
name: skill-name
description: |
  What the skill does. When it should be triggered.
  Use when [specific usage scenarios].
allowed-tools: ["Read", "Grep", "Glob"]  # Optional: restrict tool access
---
```

#### Required Fields

| Field | Description | Constraints |
|------|------|------|
| `name` | Skill identifier | Lowercase letters, digits, hyphens; max 64 characters |
| `description` | Capabilities and activation conditions | Max 1024 characters; must include "Use when" |

#### Optional Fields

| Field | Description | Example |
|------|------|------|
| `allowed-tools` | Restrict tool access | `["Read", "Grep", "Glob"]` |

### Naming Conventions

**Skill name rules**:
- Use only lowercase letters, digits, and hyphens (kebab-case)
- Max 64 characters
- Avoid underscores and uppercase letters

```
✅ Correct: code-review-skill, typescript-advanced-types
❌ Wrong: CodeReview, code_review, TYPESCRIPT
```

**File naming rules**:
- Reference files use lowercase: `react.md`, `vue.md`
- Multi-word files use hyphens: `common-bugs-checklist.md`

### Writing the Description

The description must contain two parts:

1. **Capability statement**: what exactly the Skill can do
2. **Trigger conditions**: starting with "Use when", describing when it should activate

```yaml
# ✅ Good example
description: |
  Provides comprehensive code review guidance for React 19, Vue 3, Rust,
  TypeScript, Java, Python, and C/C++.
  Helps catch bugs, improve code quality, and give constructive feedback.
  Use when reviewing pull requests, conducting PR reviews, establishing
  review standards, or mentoring developers through code reviews.

# ❌ Bad example (too vague, no trigger conditions)
description: |
  Helps with code review.
```

### Progressive Disclosure

Claude loads supporting files only when needed, not all at once.

#### File Responsibilities

| File | When loaded | Content |
|------|----------|------|
| `SKILL.md` | Always | Core principles, quick index, when to use |
| `reference/*.md` | On demand | Detailed language/framework guides |
| `assets/*.md` | When explicitly needed | Templates, checklists |
| `scripts/*.py` | When explicitly referenced | Utility scripts |

#### Content Organization

**SKILL.md** (keep to ~200 lines):
- Overview: 2-3 sentences describing the purpose
- Core principles and methodology
- Language/framework index table (linking to reference/)
- When to use this Skill

**reference/*.md** (detailed content):
- Complete code examples
- All best practices
- Review Checklist
- Edge cases and pitfalls

### Referencing Files

When referencing other files from SKILL.md:

```markdown
# ✅ Correct: use Markdown link syntax
| **React** | [React Guide](reference/react.md) | Hooks, React 19, RSC |
| **Vue 3** | [Vue Guide](reference/vue.md) | Composition API |

See the [React Guide](reference/react.md) for the full guide.

# ❌ Wrong: use code formatting
See the `reference/react.md` file.
```

**Path rules**:
- Use relative paths (relative to the Skill directory)
- Use forward slashes `/`, not backslashes
- No `./` prefix needed

### Conventions

**Severity**: review comments use the labeling scheme from SKILL.md "Technique 4", with three tiers from red to green indicating priority:

- 🔴 `[blocking]` - Must fix before merge
- 🟡 `[important]` - Should fix; discuss if you disagree
- 🟢 `[nit]` - Optional improvement, not blocking

Use these labels in new reference guides; do not invent equivalent names (such as critical/warning/suggestion).

**Language policy**: all content in this repository is written in English. New guides, edits, code comments, and examples must be in English.

---

## Types of Contributions

### Adding a New Language

1. Create a new file in `reference/` (e.g. `go.md`)
2. Follow this structure:

```markdown
# [Language] Code Review Guide

> One-sentence summary of what this guide covers.

## Table of Contents
- [Topic 1](#topic-1)
- [Topic 2](#topic-2)
- [Review Checklist](#review-checklist)

---

## Topic 1

### Subtopic

```[language]
// ❌ Bad pattern - explain why it's bad
bad_code_example()

// ✅ Good pattern - explain why it's better
good_code_example()
```

---

## Review Checklist

### Category 1
- [ ] Check item 1
- [ ] Check item 2
```

3. Add a link to the index table in `SKILL.md`
4. Update the statistics in `README.md`

### Adding Framework Patterns

1. Cite official documentation
2. Include version numbers (e.g. "React 19", "Vue 3.5+")
3. Provide runnable code examples
4. Add matching checklist items

### Improving Existing Content

- Fix spelling or grammar mistakes
- Update outdated patterns (note version changes)
- Add edge-case examples
- Improve clarity of code examples

---

## Code Example Guidelines

### Format

```markdown
// ❌ Problem description - explain why this is bad
problematic_code()

// ✅ Recommended approach - explain why this is better
recommended_code()
```

### Quality Standards

- Examples should be based on real scenarios, not contrived
- Show both the problem and the solution
- Keep examples concise and focused
- Include necessary context (imports, etc.)

---

## Submission Process

### Reporting Issues

- Use GitHub Issues to report problems or suggestions
- Provide a clear description and examples
- Mention the relevant language/framework

### Pull Request Process

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/add-go-support`
3. Make your changes
4. Commit (see commit format below)
5. Push to your fork: `git push origin feature/add-go-support`
6. Open a Pull Request

### Commit Message Format

```
type: short description

Detailed explanation (if needed)

- Specific change 1
- Specific change 2
```

**Types**:
- `feat`: new feature or content
- `fix`: bug fix
- `docs`: documentation-only change
- `refactor`: refactoring (no behavior change)
- `chore`: maintenance work

**Example**:
```
feat: add Go code review guide

- Add reference/go.md
- Cover error handling, concurrency, interface design
- Update SKILL.md index table
```

---

## Skill Design Principles

### Single Responsibility

Each Skill focuses on one core capability. This Skill focuses on **code review** and should not expand into:
- Code generation
- Project scaffolding
- Deployment configuration

### Versioning

- Note framework/language versions in reference files
- Describe version changes in commit messages when updating
- Update outdated content rather than deleting it (unless fully deprecated)

### Content Quality

- All recommendations should be backed by sources (official docs, established best practices)
- Avoid subjective preferences (such as code style); focus on objective issues
- Prioritize common pitfalls and security issues

---

## FAQ

### Q: How do I test my changes?

Copy the modified Skill into `~/.claude/skills/` and test it in Claude Code:
```bash
cp -r code-review-skill ~/.claude/skills/code-review-skill
```

### Q: Should I update SKILL.md or a reference file?

- **SKILL.md**: only change the index table or core principles
- **reference/*.md**: add/update specific language or framework content

### Q: How should outdated content be handled?

1. Note the version change (e.g. "React 18 → React 19")
2. Keep the old-version content if people still use it
3. Update the related checklist items

---

## Questions

If you have any questions, feel free to ask in GitHub Issues.
