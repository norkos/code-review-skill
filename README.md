<div align="center">

<h1>&#128269; Code Review Skill</h1>

<p>
  <strong>A comprehensive, modular code review skill for Claude Code</strong>
</p>

<p>
  <a href="https://github.com/awesome-skills/code-review-skill/blob/main/LICENSE">
    <img src="https://img.shields.io/badge/License-MIT-22c55e?style=flat-square" alt="License: MIT"/>
  </a>
  <img src="https://img.shields.io/badge/Claude_Code-Skill-7c3aed?style=flat-square&logo=anthropic&logoColor=white" alt="Claude Code Skill"/>
  <img src="https://img.shields.io/badge/Total_Lines-21%2C000%2B-3b82f6?style=flat-square" alt="21000+ lines"/>
  <img src="https://img.shields.io/badge/Languages-20%2B-f59e0b?style=flat-square" alt="20+ languages"/>
  <img src="https://img.shields.io/badge/PRs-Welcome-ec4899?style=flat-square" alt="PRs Welcome"/>
</p>

<p>
  <a href="./CONTRIBUTING.md">Contributing</a>
</p>

</div>

---

### What is this?

**Code Review Skill** is a production-ready skill for [Claude Code](https://claude.ai/code) that transforms AI-assisted code review from vague suggestions into a **structured, consistent, and expert-level** process.

It covers **20+ languages and frameworks** with over **21,000 lines** of carefully curated review guidelines — loaded progressively to minimize context window usage.

---

### &#10024; Key Features

- **Progressive Disclosure** — Core skill is ~220 lines; language guides (~200–1,100 lines each) load only when needed.
- **Four-Phase Review Process** — Structured workflow from understanding scope to delivering clear feedback.
- **Severity Labeling** — Every finding is categorized: `blocking` · `important` · `nit` · `suggestion` · `learning` · `praise`
- **Security-First** — Dedicated security checklists per language ecosystem.
- **Collaborative Tone** — Questions over commands, suggestions over mandates.
- **Automation Awareness** — Clearly separates what human review should catch vs. what linters handle.

---

### &#127760; Supported Languages & Frameworks

<table>
  <thead>
    <tr>
      <th>Category</th>
      <th>Technology</th>
      <th>Guide</th>
      <th>Lines</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="6"><strong>Frontend</strong></td>
      <td>&#9883;&#65039; React 19 / Next.js / TanStack Query v5</td>
      <td><code>reference/react.md</code></td>
      <td>~870</td>
    </tr>
    <tr>
      <td>&#128154; Vue 3.5 + Composition API</td>
      <td><code>reference/vue.md</code></td>
      <td>~920</td>
    </tr>
    <tr>
      <td>&#128302; Angular 17+ / Signals / Zoneless</td>
      <td><code>reference/angular.md</code></td>
      <td>~790</td>
    </tr>
    <tr>
      <td>&#128293; Svelte 5 / SvelteKit</td>
      <td><code>reference/svelte.md</code></td>
      <td>~1,060</td>
    </tr>
    <tr>
      <td>&#127912; CSS / Less / Sass</td>
      <td><code>reference/css-less-sass.md</code></td>
      <td>~660</td>
    </tr>
    <tr>
      <td>&#128311; TypeScript</td>
      <td><code>reference/typescript.md</code></td>
      <td>~1,020</td>
    </tr>
    <tr>
      <td rowspan="11"><strong>Backend</strong></td>
      <td>&#9749; Java 17/21 + Spring Boot 3</td>
      <td><code>reference/java.md</code></td>
      <td>~410</td>
    </tr>
    <tr>
      <td>&#9749; Java 8 + Spring Boot 2</td>
      <td><code>reference/java8.md</code></td>
      <td>~580</td>
    </tr>
    <tr>
      <td>&#9889; FastAPI</td>
      <td><code>reference/fastapi.md</code></td>
      <td>~590</td>
    </tr>
    <tr>
      <td>PHP 8.x</td>
      <td><code>reference/php.md</code></td>
      <td>~700</td>
    </tr>
    <tr>
      <td>&#128142; Ruby 3.4+/4.0 + Rails 8.x</td>
      <td><code>reference/ruby.md</code></td>
      <td>~960</td>
    </tr>
    <tr>
      <td>&#128230; NestJS</td>
      <td><code>reference/nestjs.md</code></td>
      <td>~590</td>
    </tr>
    <tr>
      <td>&#128013; Django / DRF</td>
      <td><code>reference/django.md</code></td>
      <td>~1,030</td>
    </tr>
    <tr>
      <td>&#128057; Go</td>
      <td><code>reference/go.md</code></td>
      <td>~990</td>
    </tr>
    <tr>
      <td>&#129408; Rust</td>
      <td><code>reference/rust.md</code></td>
      <td>~840</td>
    </tr>
    <tr>
      <td>&#128187; C# / .NET 8</td>
      <td><code>reference/csharp.md</code></td>
      <td>~520</td>
    </tr>
    <tr>
      <td>&#128013; Python</td>
      <td><code>reference/python.md</code></td>
      <td>~1,070</td>
    </tr>
    <tr>
      <td rowspan="7"><strong>Mobile / Systems</strong></td>
      <td>&#128241; Kotlin / Android</td>
      <td><code>reference/kotlin.md</code></td>
      <td>~1,020</td>
    </tr>
    <tr>
      <td>&#127822; Swift / SwiftUI</td>
      <td><code>reference/swift.md</code></td>
      <td>~930</td>
    </tr>
    <tr>
      <td>&#127919; Dart / Flutter</td>
      <td><code>reference/dart.md</code></td>
      <td>~650</td>
    </tr>
    <tr>
      <td>&#9881;&#65039; C</td>
      <td><code>reference/c.md</code></td>
      <td>~890</td>
    </tr>
    <tr>
      <td>&#128297; C++</td>
      <td><code>reference/cpp.md</code></td>
      <td>~890</td>
    </tr>
    <tr>
      <td>&#9889; Zig</td>
      <td><code>reference/zig.md</code></td>
      <td>~440</td>
    </tr>
    <tr>
      <td>&#128421;&#65039; Qt Framework</td>
      <td><code>reference/qt.md</code></td>
      <td>~760</td>
    </tr>
    <tr>
      <td rowspan="9"><strong>Cross-Cutting</strong></td>
      <td>&#127963;&#65039; Architecture Design Review</td>
      <td><code>reference/architecture-review-guide.md</code></td>
      <td>~470</td>
    </tr>
    <tr>
      <td>&#9889; Performance Review</td>
      <td><code>reference/performance-review-guide.md</code></td>
      <td>~820</td>
    </tr>
    <tr>
      <td>&#128269; Universal Quality Anti-Patterns</td>
      <td><code>reference/code-quality-universal.md</code></td>
      <td>~490</td>
    </tr>
    <tr>
      <td>&#128737;&#65039; Security Review</td>
      <td><code>reference/security-review-guide.md</code></td>
      <td>~640</td>
    </tr>
    <tr>
      <td>&#128200; N+1 Queries</td>
      <td><code>reference/cross-cutting/n-plus-one-queries.md</code></td>
      <td>~310</td>
    </tr>
    <tr>
      <td>&#9888;&#65039; Error Handling Principles</td>
      <td><code>reference/cross-cutting/error-handling-principles.md</code></td>
      <td>~490</td>
    </tr>
    <tr>
      <td>&#9889; Async &amp; Concurrency Patterns</td>
      <td><code>reference/cross-cutting/async-concurrency-patterns.md</code></td>
      <td>~540</td>
    </tr>
    <tr>
      <td>&#128737;&#65039; SQL Injection Prevention</td>
      <td><code>reference/cross-cutting/sql-injection-prevention.md</code></td>
      <td>~310</td>
    </tr>
    <tr>
      <td>&#128737;&#65039; XSS Prevention</td>
      <td><code>reference/cross-cutting/xss-prevention.md</code></td>
      <td>~260</td>
    </tr>
  </tbody>
</table>

---

### &#128260; The Four-Phase Review Process

```
Phase 1 - Context Gathering
  Understand PR scope, linked issues, and intent
                    |
                    v
Phase 2 - High-Level Review
  Architecture - Performance impact - Test strategy
                    |
                    v
Phase 3 - Line-by-Line Analysis
  Logic - Security - Maintainability - Edge cases
                    |
                    v
Phase 4 - Summary & Decision
  Structured feedback - Approval status - Action items
```

---

### &#127991;&#65039; Severity Labels

| Label | Meaning |
|-------|---------|
| &#128308; `blocking` | Must be fixed before merge |
| &#128992; `important` | Should be fixed; may block depending on context |
| &#128993; `nit` | Minor style or preference issue |
| &#128309; `suggestion` | Optional improvement worth considering |
| &#128218; `learning` | Educational note for the author |
| &#127775; `praise` | Explicitly highlight great work |

---

### &#128193; Repository Structure

```
code-review-skill/
|
+-- SKILL.md                              # Core skill - loaded on activation (~220 lines)
+-- README.md
+-- LICENSE
+-- CONTRIBUTING.md
|
+-- reference/                            # On-demand language guides
|   +-- react.md                          # React 19 / Next.js / TanStack Query v5
|   +-- vue.md                            # Vue 3.5 Composition API
|   +-- angular.md                        # Angular 17+ / Signals / Zoneless
|   +-- svelte.md                         # Svelte 5 / SvelteKit
|   +-- rust.md                           # Rust ownership, async/await, unsafe
|   +-- typescript.md                     # TypeScript strict mode, generics, ESLint
|   +-- nestjs.md                         # NestJS DI, Guards, Interceptors, DTOs
|   +-- java.md                           # Java 17/21 & Spring Boot 3
|   +-- java8.md                          # Java 8 & Spring Boot 2 (legacy)
|   +-- php.md                            # PHP 8.x types, PDO, security, Composer
|   +-- ruby.md                           # Ruby 3.4+/4.0, Rails 8.x, Active Record/Job
|   +-- python.md                         # Python async, typing, pytest
|   +-- django.md                         # Django / DRF security, serializers, async
|   +-- fastapi.md                        # FastAPI Depends, Pydantic v2, async, test-driven verification
|   +-- go.md                             # Go goroutines, channels, context, interfaces
|   +-- kotlin.md                         # Kotlin / Android coroutines, Compose, Flow
|   +-- swift.md                          # Swift 5.9+/6, SwiftUI, concurrency, optionals
|   +-- dart.md                           # Dart / Flutter rebuilds, null safety, isolates, state
|   +-- csharp.md                         # C# 12 / .NET 8, EF Core, ASP.NET Core
|   +-- c.md                              # C memory safety, UB, error handling
|   +-- cpp.md                            # C++ RAII, move semantics, exception safety
|   +-- zig.md                            # Zig allocators, errors, comptime, C interop
|   +-- qt.md                             # Qt object model, signals/slots, GUI perf
|   +-- css-less-sass.md                  # CSS/Less/Sass variables, responsive design
|   +-- architecture-review-guide.md      # SOLID, anti-patterns, coupling/cohesion
|   +-- code-quality-universal.md        # Reuse audit, parameter sprawl, TOCTOU, no-op updates
|   +-- performance-review-guide.md       # Core Web Vitals, N+1, memory leaks
|   +-- security-review-guide.md          # Security checklist (all languages)
|   +-- common-bugs-checklist.md          # Language-specific bug patterns
|   +-- code-review-best-practices.md     # Communication & process guidelines
|
+-- reference/cross-cutting/             # Language-agnostic cross-cutting patterns
|   +-- sql-injection-prevention.md       # Parameterized queries, 6 languages
|   +-- xss-prevention.md                 # Output encoding, CSP, 5 frameworks
|   +-- n-plus-one-queries.md             # N+1 queries, eager loading, 5 languages
|   +-- error-handling-principles.md      # Error handling principles, 7 languages
|   +-- async-concurrency-patterns.md     # Concurrency patterns, 7 languages
|
+-- assets/
|   +-- review-checklist.md               # Quick reference checklist
|   +-- pr-review-template.md             # PR review comment template
|
+-- scripts/
    +-- pr-analyzer.py                    # PR complexity analyzer
```

---

### &#128640; Installation

**Install with `npx skills` (Cursor, Claude Code, Codex, OpenCode, and other agents):**

```bash
npx skills add awesome-skills/code-review-skill
```

The skills CLI finds `SKILL.md` at the repo root. Do not nest this skill under `skills/`.

**Or clone into the Claude Code skills directory:**

```bash
# macOS / Linux
git clone https://github.com/awesome-skills/code-review-skill.git \
  ~/.claude/skills/code-review-skill

# Windows (PowerShell)
git clone https://github.com/awesome-skills/code-review-skill.git `
  "$env:USERPROFILE\.claude\skills\code-review-skill"
```

**Or add to an existing plugin:**

```bash
cp -r code-review-skill ~/.claude/plugins/your-plugin/skills/code-review/
```

---

### &#128161; Usage

Once installed, activate the skill in your Claude Code session:

```
Use code-review-skill to review this PR
```

Or create a custom slash command in `.claude/commands/`:

```markdown
<!-- .claude/commands/review.md -->
Use code-review-skill to perform a thorough review of the changes in this PR.
Focus on: security, performance, and maintainability.
```

**Example prompts:**

| Prompt | What happens |
|--------|-------------|
| `Review this React component` | Loads `react.md` - checks hooks, Server Components, Suspense patterns |
| `Review this Java PR` | Loads `java.md` - checks virtual threads, JPA, Spring Boot 3 patterns |
| `Review this Java 8 / Spring Boot 2 PR` | Loads `java8.md` - Stream/Optional, java.time, RestTemplate timeouts, javax.* |
| `Review this Flutter widget` | Loads `dart.md` - rebuilds, const, null safety, isolates, Riverpod/Bloc, keys, disposal |
| `Security review of this Go service` | Loads `go.md` + `security-review-guide.md` |
| `Architecture review` | Loads `architecture-review-guide.md` - SOLID, anti-patterns, coupling |
| `Performance review` | Loads `performance-review-guide.md` - Web Vitals, N+1, complexity |

---

### &#128300; Highlights by Language

<details>
<summary><strong>&#9883;&#65039; React 19</strong></summary>

- `useActionState` - Unified form state management
- `useFormStatus` - Access parent form status without prop drilling
- `useOptimistic` - Optimistic UI updates with automatic rollback
- Server Components & Server Actions patterns (Next.js 15+)
- Suspense boundary design, Error Boundary integration, streaming SSR
- `use()` Hook for consuming Promises

</details>

<details>
<summary><strong>&#9749; Java & Spring Boot 3</strong></summary>

- **Java 17/21**: Records, Pattern Matching for Switch, Text Blocks, Sealed Classes
- **Virtual Threads** (Project Loom): High-throughput I/O patterns
- **Spring Boot 3**: Constructor injection, `@ConfigurationProperties`, `ProblemDetail`
- **JPA Performance**: Solving N+1, correct `equals`/`hashCode` on Entities

</details>

<details>
<summary><strong>&#9749; Java 8 & Spring Boot 2 (Legacy)</strong></summary>

- **Java 8**: Lambda, Stream/`Collectors`, Optional, `java.time` (no Records / virtual threads)
- **Concurrency**: bounded thread pools, `CompletableFuture` + explicit executors
- **Spring Boot 2**: `javax.*`, constructor injection, `RestTemplate` timeouts
- **JPA / Hibernate 5**: N+1, Entity `equals`/`hashCode`, UTC time mapping pitfalls

</details>

<details>
<summary><strong>&#129408; Rust</strong></summary>

- Ownership patterns and common pitfalls
- `unsafe` code review requirements (mandatory `SAFETY` comments)
- Async/await - avoiding blocking in async context, cancellation safety
- Error handling: `thiserror` for libraries, `anyhow` for applications

</details>

<details>
<summary><strong>&#128057; Go</strong></summary>

- Goroutine lifecycle management and leak prevention
- Channel patterns, select usage
- `context.Context` propagation
- Interface design (accept interfaces, return structs)
- Error wrapping with `%w`

</details>

<details>
<summary><strong>&#9881;&#65039; C / C++</strong></summary>

- **C**: Pointer/buffer safety, undefined behavior, resource cleanup, integer overflow
- **C++**: RAII ownership, Rule of 0/3/5, move semantics, exception safety, `noexcept`
- **Qt**: Object parent/child memory model, thread-safe signal/slot connections, GUI performance

</details>

---

### &#129309; Contributing

Contributions are welcome! See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

**Ideas:**
- New language guides (Elixir, Scala, Lua...)
- Framework-specific guides (Laravel, Spring WebFlux...)
- Additional checklists and templates
- Translations of core documentation

---

### &#128196; License

MIT &copy; [awesome-skills](https://github.com/awesome-skills)

---

<div align="center">
  Made with &#10084;&#65039; for developers who care about code quality
</div>
