# Subagent Quick Reference Guide

**WHEN TO USE THIS:** Quickly determine which subagent to use for any task.

---

## Decision Tree (30 seconds)

```
User Request Received
    ↓
Does message start with "do fast:"?
    ↓ YES → Skip subagents, provide quick solution directly
    ↓ NO
Does message start with "planning mode:"?
    ↓ YES → Enter planning mode, NO implementation
    ↓ NO
Is this a question about Django/architecture?
    ↓ YES → Use django-mentor
    ↓ NO
Is this a new feature or major refactor?
    ↓ YES → Use django-architect (design first)
    ↓ NO
Is this a critical feature (reservation, auth, payments)?
    ↓ YES → Use tdd-test-first → django-implementer → security-auditor
    ↓ NO
Is this a simple feature (UI, templates, CRUD)?
    ↓ YES → Use django-implementer → frontend-integrator
    ↓ NO
Is this a git operation?
    ↓ YES → Use git-workflow-manager
    ↓ NO
Is this a checkpoint task?
    ↓ YES → Use checkpoint-manager
```

---

## Quick Lookup Table

| Task Type | Use Subagent | When |
|-----------|--------------|------|
| **Questions** | django-mentor | Architecture questions, Django patterns, best practices |
| **Design** | django-architect | New features, major refactors, architecture decisions |
| **Testing (RED)** | tdd-test-first | Critical features (Standard Mode), before implementation |
| **Implementation** | django-implementer | After tests fail (Standard) or directly (Fast Track) |
| **Security Audit** | security-auditor | Before final commit (OWASP, XSS, CSRF, multi-tenant) |
| **Accessibility** | accessibility-auditor | After UI complete (WCAG 2.1 Level A) |
| **Analytics** | analytics-expert | PageView tracking, queries, Chart.js visualization |
| **Frontend** | frontend-integrator | HTMX, DaisyUI, Flatpickr, SortableJS validation |
| **Git Operations** | git-workflow-manager | Commits, branches, no PRs (solo dev) |
| **Progress Tracking** | checkpoint-manager | Create/update checkpoints, track blockers |

---

## Standard Mode Workflow (Critical Features)

**Use for:** OAuth, multi-tenant, reservations, payments, file uploads

```
1. django-architect (design)
2. tdd-test-first (RED - tests fail)
3. git-workflow-manager (commit tests)
4. django-implementer (GREEN - tests pass)
5. security-auditor (OWASP audit)
6. accessibility-auditor (if UI)
7. git-workflow-manager (commit implementation)
8. checkpoint-manager (update progress)
```

**Coverage:** 70%+ | **Commits:** 2 (RED + GREEN)

---

## Fast Track Mode Workflow (Simple Features)

**Use for:** Templates, styling, admin CRUD, configuration

```
1. django-architect (quick review)
2. django-implementer (implement directly)
3. frontend-integrator (validate UI/UX)
4. git-workflow-manager (commit)
5. checkpoint-manager (update progress)
```

**Coverage:** 40%+ | **Commits:** 1

---

## Exception Cases

### Exception 1: "do fast:" Prefix

**User says:** `"do fast: fix this bug"`

**Action:** Skip ALL subagents, provide quick solution directly.

**Use for:** Emergency fixes, prototypes, debugging, simple questions.

---

### Exception 2: "planning mode:" Prefix

**User says:** `"planning mode: design reservation system"`

**Action:** Analyze options, NO implementation.

**Output:** Design document with trade-offs.

---

## Subagent Descriptions

### django-mentor
**Purpose:** Answer Django/GeoPandas questions, provide guidance
**Has:** context7 MCP tool for up-to-date library docs
**Example:** "How do I use Q objects for multi-tenant filtering?"

### django-architect
**Purpose:** Design Django structure, models, architecture decisions
**Output:** Model schemas, URL patterns, multi-tenant design
**Example:** "Design reservation availability system"

### tdd-test-first
**Purpose:** Create comprehensive tests BEFORE implementation (RED phase)
**Output:** Test files in `tests/` directory, all failing
**Coverage:** 70%+ for Standard Mode features

### django-implementer
**Purpose:** Implement code to pass tests (GREEN phase) or directly (Fast Track)
**Output:** Working code in `apps/` and `src/`
**Coverage:** Tests pass (Standard) or 40%+ (Fast Track)

### security-auditor
**Purpose:** Security audits (OWASP Top 10, XSS, CSRF, SQL injection, multi-tenant)
**Critical:** Runs BEFORE final commit
**Output:** Security report with severity levels

### accessibility-auditor
**Purpose:** WCAG 2.1 Level A compliance audits
**Focus:** Semantic HTML, ARIA labels, keyboard navigation
**Skip:** If no UI visible

### analytics-expert
**Purpose:** Analytics tracking, queries, Chart.js visualization
**Use:** For PageView tracking, dashboard metrics
**Example:** "Track reservation conversions"

### frontend-integrator
**Purpose:** Validate HTMX, DaisyUI, Flatpickr, SortableJS integration
**Critical Patterns:** Custom HX-Trigger events, { once: true }, ES6 modules
**Example:** "Validate HTMX modal form submission"

### git-workflow-manager
**Purpose:** Git operations (commits, branches, NO PRs - solo dev)
**Pre-commit:** Remove debug statements, format code, run tests
**Commit format:** `RIBERA-XXX: description [(RED|GREEN)]`

### checkpoint-manager
**Purpose:** Create/update checkpoints, track progress and blockers
**File:** `checkpoints/checkpoint_week_X_day_Y.md`
**Use:** Start/end of sessions, after completing features

---

## Common Patterns

### New Critical Feature (Standard Mode)
```bash
> Use checkpoint-manager to create entry for WEEK 2 RIBERA-011
> Use django-architect to design reservation availability system
> Use tdd-test-first to create tests for reservation availability
> Use git-workflow-manager to commit tests
> Use django-implementer to implement reservation availability
> Use security-auditor to check reservation system
> Use git-workflow-manager to commit implementation
> Use checkpoint-manager to update progress for WEEK 2
```

### New Simple Feature (Fast Track Mode)
```bash
> Use checkpoint-manager to create entry for WEEK 4 RIBERA-027
> Use django-architect to review UI polish tasks
> Use django-implementer to add loading indicators
> Use frontend-integrator to validate DaisyUI components
> Use git-workflow-manager to commit UI improvements
> Use checkpoint-manager to update progress for WEEK 4
```

### Emergency Bug Fix
```bash
> do fast: fix multi-tenant isolation bug in reservation queries
# No subagents, immediate fix
```

### Architecture Question
```bash
> Use django-mentor to explain when to use managers vs querysets
```

---

## TDD Refactoring Workflow

**For DDD/SOLID pattern refactoring:**

```
1. Write failing tests (RED) - define expected behavior
2. Commit tests: `REFACTOR-XXX: add tests (RED)`
3. Implement minimal code (GREEN)
4. Commit implementation: `REFACTOR-XXX: implement (GREEN)`
```

**Pattern locations:**
- Value Objects: `src/domain/value_objects/`
- Domain Services: `src/services/`
- Specifications: `apps/reservations/specifications.py`
- Commands: `apps/reservations/commands.py`

---

## Complete Documentation

**For detailed patterns and code examples, see:**
- `CLAUDE.md` - Orchestration and workflow
- `.claude/doc/solid_ddd_patterns.md` - SOLID/DDD patterns reference
- `.claude/agents/django-architect.md` - Multi-tenant architecture, DDD structure
- `.claude/agents/django-implementer.md` - CBV patterns, Django Signals, DDD implementation
- `.claude/agents/frontend-integrator.md` - HTMX, ES6 modules, DaisyUI
- `.claude/agents/security-auditor.md` - XSS, CSRF, multi-tenant security

---

## Quick Tips

**✅ DO:**
- Use subagents for ALL tasks (except "do fast:" and "planning mode:")
- Check for research docs in `.claude/doc/{feature_name}/` before implementing
- Run security-auditor BEFORE final commit on critical features
- Track progress with checkpoint-manager

**❌ DON'T:**
- Skip tdd-test-first for critical features (Standard Mode)
- Skip security-auditor for features with user input
- Commit without removing debug statements (`print()`, `console.log()`)
- Forget to update checkpoints

---

**Version:** 1.0 | **Last Updated:** 2025-11-11
