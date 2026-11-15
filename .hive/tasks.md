# Tasks

## Completed Reviews
- Task 1: Zhang Zeduan reviews core files - done
- Task 2: Tang Yin reviews reference/ + scripts/ - done

## Wave 1 (Quick Wins) ✅
- Fix-1: SKILL.md description keywords + README line counts + add swift.md to CONTRIBUTING.md - done
- Fix-2: Add fastapi/common-bugs/best-practices to HTML + line count fixes - done
- Fix-3: HTML 190→220 line count sync - done
- Review-1: Gu Yanwu review - done (no blocking)

## Wave 2 (Structural Improvements) ✅
- Fix-A: pr-analyzer.py logic fixes + tests expanded to 40 cases - done
- Fix-B: SKILL.md Cross-Cutting table consolidation + template enhancements + checklist alignment - done
- Fix-4: Fix the 3 important issues found in Review-2 - done
- Review-2: Gu Yanwu review - done (no blocking)

## Wave 3 (Guide Expansion) - In Progress

### Fix-C1: C + C++ + Qt expansion
- **Assignee:** Zhang Zeduan
- **Status:** done
- **Result:** c.md 890 lines, cpp.md 893 lines, qt.md 757 lines (all ≥500)
- **Items:**
  - C (285→500+ lines): add testing section, CERT C secure coding, UB examples, cross-platform portability
  - C++ (385→500+ lines): add C++20/23 features (concepts/modules/ranges), testing, constexpr
  - Qt (185→500+ lines): add testing, QML/Qt Quick, Qt6 migration, Model/View architecture

### Fix-C2: Angular + TypeScript + security expansion
- **Assignee:** Tang Yin
- **Status:** done
- **Result:** angular 788 lines, typescript 1015 lines, security 636 lines (all meet target)
- **Items:**
  - Angular (419→500+ lines): add testing (Jasmine/Karma), route guards, DI patterns, HttpInterceptor
  - TypeScript (553 lines): add testing (Vitest/Jest), module resolution (ESM vs CJS), TS 4.9+/5.x features
  - security-review-guide.md (266→500+ lines): cross-language code examples for SQLi/XSS/CSRF/SSRF/IDOR/command injection

### Review-3: Review of Wave 3
- **Assignee:** Gu Yanwu
- **Status:** done
- **Result:** 3 blocking (broken code fences + corrupted table + untracked files) + 4 important (HTML line counts + angular interceptor + guard + satisfies)

### Fix-5: Fix Review-3 blocking/important issues
- **Assignee:** Zhang Zeduan (README/HTML) + Tang Yin (angular/typescript)
- **Status:** done
- **Result:** all fixed, 40/40 tests passed

## Wave 4 (Shared Module Extraction)

### Fix-D: Extract cross-language duplicate content
- **Status:** done
- **Result:** 5 cross-cutting modules (1914 lines), reference links added to 15+ guides
- **Items:**
  - #14 Extract N+1 queries, SQL injection, XSS, error handling principles, and async patterns into shared modules
  - Language guides now reference the shared modules

### Review-4: Final review
- **Assignee:** Gu Yanwu
- **Status:** dispatched
