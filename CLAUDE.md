# CLAUDE.md - Project Configuration

---

## Project Context

This file defines the standard workflow for working with Claude Code.

**Project-specific documentation:**
- `@PROJECT_SPECS.md` - User stories, requirements, and acceptance criteria
- `@CORRECTIONS.md` - Known discrepancies and pending refactorizations

---

## CRITICAL: Session Initialization Protocol (OPTIMIZED)

**MANDATORY: EVERY time a new session starts (after /clear or new conversation or compact conversation), you MUST AUTOMATICALLY:**

### Step 1: Load Core Configuration (MINIMAL - DO THIS FIRST, WITHOUT BEING ASKED)

1. **Read ONLY core files (~500 lines total):**
   ```
   .claude/SUBAGENT_GUIDE.md                  # Subagent quick reference
   .claude/doc/README_CLAUDE.md                       # Research-First Pattern
   .claude/doc/python_professional_standards.md  # NASA + Wemake standards
   ```

2. **Check for active feature documentation:**
   ```
   .claude/doc/{feature_name}/checkpoint.md   # Current progress
   .claude/doc/{feature_name}/checklist.md    # Implementation checklist
   ```

3. **Confirm to user:**
   ```
   Session initialized: Core configuration loaded
   Research-First Pattern active
   Python Professional Standards (NASA + Wemake) active
   Subagent system ready (lazy loading)
   Ready to work

   Available modes:
   - Standard Mode (critical features): Full TDD + audits
   - Fast Track Mode (UI/admin): Rapid development
   - Planning Mode: Architecture analysis

   Quick actions:
   - "do fast:" prefix - Skip subagents, direct work
   - "planning mode:" prefix - Analysis only, no code
   ```

**Context Optimization:** Only core files loaded (~500 lines). Specialists load on-demand (~1500 lines when needed). **Total savings: 55% less context vs loading all 18 files.**

---

### Step 2: Lazy Load Specialists (AUTOMATIC - DO NOT ASK USER)

**When user describes a task, YOU automatically:**

1. **Analyze task type and load relevant specialists:**

   | Task Type | Load These Specialists (~1500 lines) | Also Read |
   |-----------|--------------------------------------|-----------|
   | Critical feature (OAuth, multi-tenant, reservations, file uploads) | django-architect, tdd-test-first, django-implementer, security-auditor | solid_ddd_patterns.md |
   | Payment integration | django-architect, django-implementer, security-auditor | mercadopago_integration_guide.md, solid_ddd_patterns.md |
   | OAuth/Authentication | django-architect, django-implementer, security-auditor | google_oauth_integration_guide.md |
   | Complex business logic | django-architect, tdd-test-first, django-implementer | solid_ddd_patterns.md |
   | UI/Admin feature (templates, CRUD, admin interfaces) | django-implementer, frontend-integrator | - |
   | Planning/Architecture question | django-architect (only if needed) | solid_ddd_patterns.md |
   | Celery/async task | celery-async-expert, django-implementer | - |
   | Analytics feature | analytics-expert, django-implementer | - |
   | Security audit | security-auditor | - |
   | Commit/Git operation | git-workflow-manager | - |
   | Checkpoint creation | checkpoint-manager | - |
   | "do fast:" prefix | NONE (skip all subagents) | - |
   | "planning mode:" prefix | django-architect (if needed) | solid_ddd_patterns.md |

2. **Check for existing research documentation:**
   ```bash
   # Look for (in priority order):
   .claude/doc/{feature_name}/checkpoint.md   # Progress tracking (MANDATORY)
   .claude/doc/{feature_name}/checklist.md    # Implementation tasks (MANDATORY)
   .claude/doc/{feature_name}/backend.md      # Django models, managers, services, forms, Celery
   .claude/doc/{feature_name}/frontend.md     # UI components, DaisyUI, HTMX, JavaScript
   .claude/doc/{feature_name}/security.md     # OWASP, XSS, CSRF, multi-tenant isolation

   # Core architecture guides (ALWAYS consult for complex features):
   .claude/doc/solid_ddd_patterns.md              # DDD/SOLID patterns - Value Objects, DTOs, Builder, Specification
   ```

3. **If documentation EXISTS:**
   - Read all relevant files
   - Follow the documented approach
   - Reference the documentation in your work
   - **CRITICAL**: backend.md contains the architecture blueprint - follow it exactly

4. **If documentation DOES NOT EXIST:**
   - **Create checkpoint.md and checklist.md first:**
     ```
     Creating documentation for {feature_name}:

     MANDATORY (always create):
     - checkpoint.md - Progress tracking, blockers, learnings
     - checklist.md - Implementation phases and tasks

     OPTIONAL (create if needed):
     - backend.md - Architecture blueprint
     - security.md - Security considerations
     ```
   - Then proceed with implementation following the checklist

5. **Proceed with workflow** (user doesn't see the lazy loading process)

**CRITICAL RULES:**
- **DO NOT ask user** "should I load X config?" - do it automatically based on task type
- **DO NOT mention** to user that you're loading specialist configs - just do it
- **DO NOT load** specialists for "do fast:" tasks - work directly
- **DO load** specialists for Standard/Fast Track workflows
- These files contain workflow specifications, subagent behaviors, and project standards
- `.claude/doc/README_CLAUDE.md` defines the **Research-First Pattern** - agents MUST research before implementing
- Without loading them, you won't follow the correct workflow patterns

**REMEMBER: This is NOT optional. This is the CORE workflow.**

---

## CRITICAL: Documentation BEFORE Code (BLOCKING RULE)

**NEVER start ANY implementation without documentation. This is a BLOCKING requirement.**

### The Rule

```
BEFORE writing ANY code for a new feature:
1. Check if .claude/doc/{feature_name}/ exists
2. If NO -> CREATE checkpoint.md + checklist.md FIRST
3. If YES -> READ and follow documented approach
4. ONLY THEN proceed with implementation
```

### Why This Matters

- **Without documentation**: Work is lost on context switches, progress is invisible, decisions are forgotten
- **With documentation**: Work persists across sessions, progress is trackable, decisions are recorded

### Enforcement

**If you find yourself about to call django-implementer or write code:**

1. STOP
2. Ask: "Did I create/verify `.claude/doc/{feature_name}/checkpoint.md` exists?"
3. If NO -> Create it NOW before proceeding
4. If YES -> Continue

**This applies to ALL implementation work, including:**
- New features
- Bug fixes that span multiple files
- Refactoring tasks
- Integration work

**Exceptions (NO documentation needed):**
- `"do fast:"` prefix tasks (emergency fixes, quick prototypes)
- Single-line fixes
- Pure research/exploration (no code changes)

---

## CRITICAL: Subagent Usage Protocol

**MANDATORY: ALWAYS use subagents for ALL tasks, with ONLY 2 exceptions:**

### Decision Tree

```
User Request Received
    ↓
Does message start with "do fast:"?
    ↓ YES → Skip subagents, provide quick solution directly (NO docs needed)
    ↓ NO
Does message start with "planning mode:"?
    ↓ YES → Enter planning mode, analyze options, NO implementation
    ↓ NO
    ↓
╔══════════════════════════════════════════════════════════════════╗
║  BLOCKING STEP: Documentation Check (MANDATORY before any code)  ║
╠══════════════════════════════════════════════════════════════════╣
║  Check: Does .claude/doc/{feature_name}/ exist?                  ║
║      ↓ YES → Read checkpoint.md + checklist.md, follow approach  ║
║      ↓ NO  → CREATE checkpoint.md + checklist.md FIRST           ║
║              (Use checkpoint-manager subagent)                   ║
║              THEN proceed with implementation                    ║
╚══════════════════════════════════════════════════════════════════╝
    ↓
Route to appropriate subagent:
    - checkpoint-manager (ALWAYS FIRST if docs don't exist)
    - django-architect for architecture/design
    - django-implementer for code implementation
    - frontend-integrator for HTMX/DaisyUI/JavaScript
    - security-auditor for security checks
    - tdd-test-first for comprehensive tests
    - etc.
```

### Exception 1: "do fast:" Prefix

**When user message starts with `"do fast:"` → Skip all subagents and provide quick solution directly.**

**Use cases:**
- Emergency bug fixes
- Quick prototypes
- One-off scripts
- Debugging assistance
- Simple questions

**Example:**
```
User: do fast: fix this syntax error in views.py line 42
Claude: [Provides direct fix without using subagents]
```

**CRITICAL:**
- NO research documentation created
- NO subagent consultations
- NO extended workflow
- Just provide the solution directly

### Exception 2: "planning mode:" Prefix

**When user message starts with `"planning mode:"` → Analyze and plan, but NO implementation.**

**Use cases:**
- Architectural decisions
- Technology comparisons
- Approach evaluation
- Design reviews
- Requirements analysis

**Example:**
```
User: planning mode: Should we use WebSocket or Server-Sent Events for real-time notifications?
Claude: [Analyzes both options, presents pros/cons, recommends approach, but does NOT implement]
```

**CRITICAL:**
- NO code generation
- NO file modifications
- NO implementation
- Only analysis, comparison, and recommendations

### Standard Workflow (Default - NO Prefix)

**For ALL other requests → ALWAYS use subagents according to feature type:**

**Standard Mode Features (Full TDD):**
- OAuth Google integration → django-architect + tdd-test-first + django-implementer + security-auditor
- Multi-tenant isolation → django-architect + tdd-test-first + django-implementer + security-auditor
- Reservation system → django-architect + tdd-test-first + django-implementer + security-auditor
- File uploads → django-architect + tdd-test-first + django-implementer + security-auditor
- Email notifications → django-architect + tdd-test-first + django-implementer

**Fast Track Features (Rapid Development):**
- Admin interfaces → django-architect + django-implementer + frontend-integrator
- UI templates → frontend-integrator + django-implementer
- Static content → frontend-integrator
- Configuration pages → django-implementer + frontend-integrator

**See @WORKFLOW_GUIDE.md for complete workflow details.**

---

## CRITICAL: Checkpoint and Checklist System

### MANDATORY: Documentation Structure

For EVERY feature implementation, create in `.claude/doc/{feature_name}/`:

```
.claude/doc/{feature_name}/
    checkpoint.md    # Progress tracking, blockers, learnings
    checklist.md     # Implementation phases and tasks
```

### When to Create/Update

| Event | Action |
|-------|--------|
| New feature starts | Create both files immediately |
| Context summary requested | Update checkpoint.md before summary |
| Session ends | Update checkpoint with current state |
| Feature completes | Mark all items complete, document learnings |
| Blocker encountered | Document in checkpoint immediately |

### Checkpoint Format

```markdown
# {Feature Name} - Progress Checkpoint

> **Purpose**: Track implementation progress
> **Last updated**: {date}

---

## Current Status

| Phase | Status | Progress |
|-------|--------|----------|
| Phase 1: ... | COMPLETED | 100% |
| Phase 2: ... | IN_PROGRESS | 50% |
| Phase 3: ... | PENDING | 0% |

---

## Checkpoint #N - {date}

### Completed This Session
- [x] Task 1
- [x] Task 2

### Key Decisions
- Decision 1: Rationale

### Blockers Encountered
- Blocker: Description
- Solution: How resolved

### Next Steps
- [ ] Task for next session

---

## Learnings
- Learning 1
- Learning 2
```

### Checklist Format

```markdown
# {Feature Name} - Implementation Checklist

> **Purpose**: Track implementation tasks
> **Reference**: See checkpoint.md for progress details
> **Last updated**: {date}

---

## Estado General

| Fase | Estado | Progreso |
|------|--------|----------|
| Fase 1: ... | COMPLETADA | 100% |
| Fase 2: ... | EN_PROGRESO | 50% |
| Fase 3: ... | PENDIENTE | 0% |

---

## Fase 1: {Phase Name}

- [x] Task 1
- [x] Task 2
- [ ] Task 3
  - [ ] Subtask 3.1
  - [ ] Subtask 3.2

---

## Files Created/Modified

| File | Action | Description |
|------|--------|-------------|
| path/to/file.py | CREATED | Description |
| path/to/file.py | MODIFIED | Description |
```

### Existing Examples

See implementation examples in:
- `.claude/doc/galicia_integration/checkpoint.md`
- `.claude/doc/galicia_integration/checklist.md`

---

## Initial Project Analysis Strategy

**On first interaction:**
1. Map project structure (directories, key files) - lightweight scan
2. Identify Django apps and their purposes
3. Note test structure and coverage status
4. Skip heavy files (>1MB, binaries, large media)

**On feature start:**
1. Deep dive into relevant app modules only
2. Load associated tests
3. Check related checkpoints (if exist)

### Exclusion Rules - DO NOT READ

**External Libraries (use context7 MCP instead):**
- `flatpickr/`, `chart.js/` - frontend libraries
- Any `node_modules/` or `venv/` content
- Django/Pydantic/django-allauth source code

**System/Cache Files:**
- `__pycache__/`, `.pytest_cache/`, `.mypy_cache/`
- `.git/` (use git commands instead)
- `*.pyc`, `*.pyo` compiled files

**Large Data Files:**
- `media/` uploaded files
- Database dumps
- Large images/videos

**Build Artifacts:**
- `staticfiles/` collected static files
- `dist/`, `build/` directories

### What TO Analyze

**Priority 1 (Always):**
- `apps/*/` - All Django apps
- `src/domain/` - Value Objects, DTOs (DDD patterns)
- `src/services/` - Domain services, builders
- `tests/*/` - Test suites
- `PROJECT_SPECS.md` - Requirements
- `TECHNICAL_SPECS.md` - Architecture details
- `WORKFLOW_GUIDE.md` - Process documentation
- `.claude/doc/solid_ddd_patterns.md` - DDD/SOLID implementation guide

**Priority 2 (On Demand):**
- `templates/*/` - When working on views
- `static/js/` - When debugging frontend
- Configuration files - When needed

**Priority 3 (Metadata Only):**
- Migration files - Only when debugging migrations

---

## Workflow Modes

**All workflow documentation is consolidated in @WORKFLOW_GUIDE.md**

Two modes available based on feature criticality:

### Standard Mode (Critical Features)
Full TDD workflow with comprehensive testing and audits. Use when:
- Security-critical features (OAuth, file uploads, input sanitization)
- Complex business logic (reservation availability, multi-tenant isolation)
- Core functionality (reservation workflow, email notifications)
- Features that are hard to debug later

**Process:** 2 commits per feature (RED + GREEN)
**Coverage:** 70%+
**See:** @WORKFLOW_GUIDE.md section "Standard Mode"

### Fast Track Mode (Secondary Features)
Faster development for simple features. Use when:
- UI templates and styling
- Simple CRUD operations
- Admin interfaces
- Configuration pages
- Static content

**Process:** Item by item work, verify each, commit when stable
**Coverage:** 40%+ on critical paths
**See:** @WORKFLOW_GUIDE.md section "Fast Track Mode"

### Hybrid Mode (Recommended for this project)
**Standard Mode** for critical features + **Fast Track Mode** for secondary features.

Decision matrix in @WORKFLOW_GUIDE.md helps classify each feature.

---

## Tech Stack

### Backend
- **Django 5.1+** - Web framework
- **PostgreSQL** - Database
- **django-allauth** - OAuth Google integration
- **Bleach** - HTML sanitization
- **Python-magic** - File type detection
- **Pillow** - Image processing
- **Celery + Redis** - Async task queue (email sending only)

### Frontend
- **HTMX 2.0+** - Dynamic HTML interactions (partial rendering)
- **Tabler UI** - Component library (built on Bootstrap CSS)
- **Tailwind CSS** - Utility CSS framework
- **Flatpickr 4.6+** - Date picker for reservation calendar
- **Google Maps JavaScript API** - Location display
- **Chart.js 4.x** - Analytics charts

### Project Settings
- **Timezone**: `America/Argentina/Buenos_Aires`
- **Language**: `es-ar` (Spanish - Argentina)
- **Templates**: Root `/templates/` directory with app subdirectories
- **Static**: Collected to `/staticfiles/` (production)
- **Multi-tenant**: Site-based isolation at database level (domain/subdomain-based)

### Development Tools
- **unittest** - Testing framework (40% minimum for Fast Track, 70% for Standard)
- **black** - Python code formatter
- **flake8** - Python linter
- **mypy + pyright** - Static type checking
- **prettier** - JavaScript formatter
- **eslint** - JavaScript linter

---

## Architecture Principles

### Scope Rule (Adapted to Django)
- **apps/** = Django apps (features)
- **src/domain/** = Value Objects, DTOs (framework-independent, DDD patterns)
- **src/services/** = Domain services, builders (business logic used by 2+ apps)
- **framework/** = Base models, mixins shared across ALL apps
- **Code used by 1 app only** = Stays in that app

**IMPORTANT - Managers and QuerySets**:
- **ALWAYS** separate complex queries into `querysets.py`
- **ALWAYS** expose querysets through `managers.py`
- **QuerySets** = Filtering, aggregation, database logic
- **Managers** = Business logic that uses querysets

**See django-architect.md for complete structure examples and patterns.**

---

## DDD and SOLID Patterns (MANDATORY for Complex Features)

**CRITICAL**: When implementing or maintaining features with complex business logic, ALWAYS apply DDD and SOLID patterns.

### When to Apply DDD/SOLID

| Feature Type | Apply DDD/SOLID? | Patterns to Use |
|--------------|------------------|-----------------|
| Reservation system | **YES** | Value Objects, DTOs, Builder Pattern, Specifications |
| Payment processing | **YES** | Domain Services, Value Objects, DTOs |
| Availability checking | **YES** | Specification Pattern, Domain Services |
| Multi-tenant isolation | **YES** | Value Objects for site context |
| Complex forms | **YES** | DTOs, Builder Pattern |
| Simple CRUD | NO | Standard Django patterns |
| Admin interfaces | NO | Standard Django Admin |
| Static templates | NO | Standard templates |

### Core DDD Patterns Implemented

1. **Value Objects** (`src/domain/value_objects/`)
   - Immutable data containers with validation
   - Examples: `DateRange`, `Money`, `GuestInfo`, `ReservationData`
   - Use when: Data has business rules and needs validation

2. **DTOs (Data Transfer Objects)** (`src/domain/dtos/`)
   - Transfer data between layers without business logic
   - Examples: `ReservationDTO`, `PaymentDTO`
   - Use when: Passing data to templates, APIs, or services

3. **Domain Services** (`src/services/`)
   - Business logic that doesn't belong to a single entity
   - Examples: `AvailabilityService`, `PricingService`
   - Use when: Logic spans multiple models or external systems

4. **Builder Pattern** (`src/services/reservation_builder.py`)
   - Fluent interface for complex object construction
   - Use when: Creating objects with many optional parameters

5. **Specification Pattern** (`apps/*/specifications.py`)
   - Encapsulate business rules as reusable objects
   - Examples: `AvailabilitySpecification`, `OverlapSpecification`
   - Use when: Complex validation or filtering rules

6. **Command Pattern** (`apps/*/commands.py`)
   - Encapsulate operations as objects
   - Use when: Operations need undo, logging, or queuing

### Value Objects vs DTOs - Decision Guide

| Pregunta | Si = Value Object | Si = DTO |
|----------|-------------------|----------|
| Tiene reglas de negocio? | SI | NO |
| Necesita validacion al crear? | SI | NO |
| Tiene metodos/comportamiento? | SI | NO |
| Solo transporta datos entre capas? | NO | SI |
| Debe ser inmutable? | SI | Opcional |

**Flujo tipico:**
```
Form/Request -> DTO (entrada) -> Value Objects (validacion) -> Model (DB)
Model (DB) -> DTO (salida) -> Template/API
```

**Ejemplo Value Object** (con reglas de negocio):
```python
@dataclass(frozen=True)  # Inmutable
class DateRange:
    check_in: date
    check_out: date

    def __post_init__(self):
        if self.check_out <= self.check_in:
            raise ValueError("check_out must be after check_in")

    @property
    def nights(self) -> int:
        return (self.check_out - self.check_in).days

    def overlaps(self, other: "DateRange") -> bool:
        return self.check_in < other.check_out and other.check_in < self.check_out
```

**Ejemplo DTO** (solo transporte de datos):
```python
class ReservationDTO(BaseModel):  # Pydantic
    id: int
    guest_name: str
    check_in: date
    total_price: Decimal
    # Sin validacion de negocio, sin metodos
```

### SOLID Principles Application

- **S (Single Responsibility)**: Each class/function has ONE job
- **O (Open/Closed)**: Extend via new classes, don't modify existing
- **L (Liskov Substitution)**: Subclasses must be substitutable
- **I (Interface Segregation)**: Small, focused interfaces
- **D (Dependency Inversion)**: Depend on abstractions, not concretions

### Documentation Reference

**MANDATORY**: Before implementing complex features, read:
```
.claude/doc/solid_ddd_patterns.md    # Complete DDD/SOLID implementation guide with examples
```

This guide contains:
- Complete Value Object implementations
- DTO patterns with Pydantic
- Builder Pattern with fluent interface
- Specification Pattern examples
- Command Pattern implementation
- Testing strategies for DDD code
- Migration path from Django models to DDD

---

## Async Processing: Celery + WebSocket + Signals

### Core Philosophy

**CRITICAL: Different technologies for different purposes:**

| Technology | Purpose | When to Use | Persistence | Retriability |
|-----------|---------|-------------|-------------|--------------|
| **Celery** | Work that MUST complete | Emails, payments, external APIs, file processing | ✅ Persistent (Redis/DB) | ✅ Automatic retries |
| **Signals** | Consistent event triggering | Notification on model save/delete | ✅ Code-level hooks | ❌ No retries (use Celery) |
| **WebSocket** | Real-time UX enhancement | Online admin notifications | ❌ Fire-and-forget | ❌ No retries |
| **DB Notifications** | Backup for offline users | Store notifications for later viewing | ✅ Persistent | N/A (read from DB) |

### Decision Tree

**Question: Does this work MUST complete (emails, payments, external APIs)?**
- **YES** → Use **Celery task** (persistent, retriable)
- **NO** → Continue

**Question: Should this trigger consistently from ANY location (view, API, admin, management command)?**
- **YES** → Use **Django Signal** (post_save, post_delete)
- **NO** → Call directly in view/service

**Question: Is this UX enhancement for online users (real-time notifications)?**
- **YES** → Add **WebSocket notification** (fire-and-forget, via signal)
- **NO** → Skip WebSocket

**Question: Do offline users need to see this later?**
- **YES** → Create **DB Notification** record (via signal)
- **NO** → Skip DB persistence

**See django-implementer.md for complete Signal implementation patterns.**
**See @IMPLEMENT_WEBSOCKET.md for complete WebSocket implementation guide.**

---

## Multi-tenant Architecture

### Core Concept

This is a multi-tenant resort management system where each Site represents a different resort/property.

**Multi-tenant Isolation:**
- All models have FK to Site
- **Domain/Subdomain access**: Custom domain OR subdomain-based URLs only
- Middleware sets `request.site` using priority-based resolution
- All queries MUST filter by site

**Multi-tenant Approach (2 Access Methods):**

1. **Custom Domain (Priority 1)** - Production recommended
   - Example: `riberacalma.com`, `alpendorf.com.ar`
   - Best for SEO and branding
   - Requires DNS configuration

2. **Subdomain (Priority 2)** - Shared platform hosting & development
   - Production: `ribera.calmasites.com`, `alpendorf.calmasites.com`
   - Development: `ribera.lvh.me` (NOT `.localhost` - browsers reject those cookies)
   - Good for multi-tenant SaaS platforms
   - Requires wildcard DNS and SSL certificate (production only)
   - **Auto-generated from site name** (same as slug)
   - **Always available** - every site has a subdomain

**CRITICAL Cookie Issue:**
- Modern browsers **reject** cookies with domain `.localhost`
- **ALWAYS use `.lvh.me`** for local development with subdomains
- `lvh.me` is a public DNS domain that points to 127.0.0.1
- Production domains work normally (`.yourdomain.com`)

**See django-architect.md for:**
- Complete Multi-tenant Middleware implementation
- Cookie Domain Configuration (lvh.me setup)
- Site Model structure
- DNS configuration examples
- Security considerations

---

## OAuth Google Integration

### Email Uniqueness Validation

**Bidirectional email validation:**
- OAuth users cannot use emails from CreationForm users
- CreationForm users cannot use emails from OAuth users

**See django-implementer.md for complete Signal and Form validation patterns.**

### Complete Implementation Guide

**MANDATORY**: For OAuth Google implementation, read:
```
.claude/doc/google_oauth_integration_guide.md    # Complete step-by-step OAuth guide
```

This guide contains:
- django-allauth configuration (settings.py)
- User model with auth_provider field
- Social account signals for email validation
- Custom AccountAdapter for phone handling
- URL configuration and views
- Frontend templates with Google button
- Subdomain handling for multi-tenant
- Testing strategies for OAuth flows
- Troubleshooting common issues

---

## Payment Integration (MercadoPago)

### Core Features

**Split Payment System:**
- 95% to site owner (marketplace recipient)
- 5% platform fee (collector)
- Fernet encryption for credentials
- HMAC webhook signature verification

### Security Requirements

- **Encrypted credentials** stored per Site
- **Webhook signature** verification using x-signature header
- **Idempotent** webhook processing
- **Multi-tenant** payment isolation

### Complete Implementation Guide

**MANDATORY**: For MercadoPago implementation, read:
```
.claude/doc/mercadopago_integration_guide.md    # Complete step-by-step MercadoPago guide
```

This guide contains:
- Site model fields for MercadoPago credentials
- Fernet encryption service implementation
- MercadoPagoService with split payments
- Webhook endpoint with HMAC verification
- WebhookEvent model for idempotency
- Frontend templates with MercadoPago JS SDK v2
- Payment preference creation
- Testing strategies for payment flows
- Production deployment checklist

---

## HTMX + DaisyUI Workflow

### Separation of Concerns

**Django Views:**
- Handle HTTP requests
- Validate data with Django forms
- Business logic in `src/services/` (if reusable)
- Return HTML fragments for HTMX

**Templates:**
- **CRITICAL**: Templates MUST be in `/templates/` root directory, NOT in app folders
- Organize by app: `/templates/sites/`, `/templates/accommodations/`, `/templates/reservations/`
- Use DaisyUI classes exclusively (avoid custom CSS)
- HTMX in `<div>`, `<form>`, or `<a>` tags

**JavaScript:**
- **ONLY** for Flatpickr (date picker), Google Maps, and reusable components
- Client-side validation with vanilla JS (UX feedback)
- Always replicate validation in backend
- NO React/Vue/Angular patterns - vanilla JS only

**See frontend-integrator.md for:**
- Complete ES6 Module patterns
- HTMX Modal patterns with custom events
- HTMX Re-initialization patterns
- Form rendering system (ui_field.html)

---

## Coding Conventions (NASA-inspired)

### Core Principles
1. **Single Responsibility:** One function/class = one job
2. **Atomic Code:** Small, focused units
3. **Early Returns:** Avoid nested conditions
4. **No None/Null Returns:** Use Result pattern or raise exceptions
5. **Strict Typing:** mypy and pyright must pass
6. **Type hints everywhere:** Python typed, JSDoc for JavaScript

### Python Standards
- Use Google-style docstrings
- Early returns to avoid nesting
- Type hints on all functions
- Max 60 lines per function (Rule 4)

**See .claude/doc/python_professional_standards.md for NASA + Wemake standards.**

### JavaScript Standards
- Use ES6 Modules for all JavaScript (97% browser support)
- Pass Django variables via inline `<script type="module">`
- External `.js` files cannot use Django template tags
- JSDoc for function documentation

**See frontend-integrator.md for complete ES6 Module patterns and examples.**

### Naming Conventions
- **Python:** `snake_case` for everything
- **JavaScript:** `camelCase` for variables/functions, `PascalCase` for classes
- **Templates:** `snake_case.html`
- **CSS Classes:** Use DaisyUI classes (avoid custom CSS)

---

## Django View Standards

**Default Rule: Always prefer CBVs for consistency and Rule 4 compliance (max 60 lines per function).**

### When to Use CBV vs FBV

| Scenario | Use CBV | Use FBV |
|----------|---------|---------|
| CRUD operations | ✅ | ❌ |
| Forms with validation | ✅ | ❌ |
| HTMX partial rendering | ✅ | ❌ |
| Multi-tenant filtering | ✅ | ❌ |
| Complex permissions | ✅ | ❌ |
| Reusable patterns | ✅ | ❌ |
| Truly trivial views (<10 lines) | ✅ | ✅ |

**Why CBVs:**
- Methods stay under 60 lines (Rule 4 compliant)
- Each method = single responsibility
- Easy to test individual methods
- Reusable mixins for multi-tenant filtering
- Clean HTMX hooks (`form_valid` / `form_invalid`)

**See django-implementer.md for:**
- Complete CBV Patterns Library
- Multi-tenant Mixins (SiteAdminRequiredMixin, SiteFilteredMixin, SiteContextMixin)
- HTMX Integration Patterns
- CBV Testing Methods
- Migration Path from FBV to CBV

---

## Git Strategy

### Branch Naming
```
feature/week-X-description
feature/ribera-XXX-description
```

### Commit Format
```
RIBERA-XXX: <description> [(RED|GREEN)]
```

**Examples:**
```bash
# Tests phase (Standard Mode)
RIBERA-001: add User model with OAuth tests (RED)

# Implementation phase (after all audits pass)
RIBERA-001: implement User model with OAuth integration (GREEN)

# Fast Track (single commit)
RIBERA-005: add home page template with editable content

# Hotfix
RIBERA-042: fix multi-tenant isolation in reservation queries
```

### Pre-Commit Checklist

**CRITICAL**: Before EVERY commit, run these commands:

```bash
# 1. Check for debug statements (MUST return empty)
grep -r "print(" apps/ src/
grep -r "console.log" static/js/

# 2. Format code
black apps/ src/ tests/
prettier --write "static/js/**/*.js"

# 3. Lint
flake8 apps/ src/ tests/
mypy apps/ src/

# 4. Run tests
python manage.py test

# 5. System check
python manage.py check
```

**Zero tolerance policy**: NO debug statements in commits.

---

## RULES (Unbreakable)

### Development
- **NEVER** write code without concrete functionality from PROJECT_SPECS.md
- **NEVER** implement critical features without failing tests (Standard Mode)
- **NEVER** mention Claude/AI in commits
- **NEVER** return None/null (use exceptions or Result pattern)
- **NEVER** use emojis in code, commits, documentation, markdown files, or any output
- **NEVER** leave debug statements (print, console.log) in committed code
- **ALWAYS** type hint everything (mypy strict mode)
- **ALWAYS** validate in backend even if validated in frontend
- **ALWAYS** use logging for errors (ERROR) and critical actions (INFO)
- **ALWAYS** sanitize user input with bleach before storing or rendering
- **ALWAYS** apply DDD/SOLID patterns for complex business logic
- **ALWAYS** consult `.claude/doc/solid_ddd_patterns.md` before implementing complex features
- **ALWAYS** create checkpoint.md and checklist.md in `.claude/doc/{feature}/` for new features
- **ALWAYS** update checkpoint.md before session ends or context summary

### Code Organization
- **NEVER** put reusable code in app/ (goes to src/)
- **NEVER** use Django methods in src/core/ (framework-independent)
- **NEVER** put templates inside app folders (use /templates/ root)
- **ALWAYS** use managers/querysets for complex queries
- **ALWAYS** separate QuerySets (filtering) from Managers (business logic)
- **ALWAYS** use Google-style docstrings
- **ALWAYS** put Value Objects and DTOs in `src/domain/` (framework-independent)
- **ALWAYS** put domain services in `src/services/` when reusable across apps
- **ALWAYS** put specifications in `apps/*/specifications.py` when app-specific

### Security & Quality
- **ALWAYS** use parameterized queries (prevent SQL injection)
- **ALWAYS** validate/sanitize user input with bleach
- **ALWAYS** use CSRF protection with HTMX
- **ALWAYS** validate file uploads (type, size, content)
- **ALWAYS** check minimum test coverage (40% Fast Track, 70% Standard)

### Multi-tenant Isolation
- **ALWAYS** filter by Site in queries (prevent cross-site data access)
- **ALWAYS** check site ownership in views
- **NEVER** expose data from other sites
- **ALWAYS** test multi-tenant isolation
- **ALWAYS** use `request.site` for filtering

### Django Admin Multi-tenant (CRITICAL)

**MANDATORY: Every model registered in Django Admin that has a FK to Site (direct or indirect) MUST implement multi-tenant security.**

**Required Methods:**
1. `get_queryset()` - Filters by site owner/admins + `.distinct()`
2. `formfield_for_foreignkey()` - Filters site/parent choices
3. `has_change_permission()` - Checks site ownership
4. `has_delete_permission()` - Checks site ownership

**See django-architect.md for complete Django Admin 4-method security pattern with examples.**

### OAuth & Authentication
- **ALWAYS** validate email uniqueness bidirectionally (OAuth ↔ CreationForm)
- **ALWAYS** check auth_provider before operations
- **NEVER** allow duplicate emails across auth methods
- **ALWAYS** handle OAuth failures gracefully

---

## Checkpoint System

We use a checkpoint system to track progress for each feature.

**Location:** `.claude/doc/{feature_name}/`

**Required files:**
- `checkpoint.md` - Progress tracking, blockers, learnings
- `checklist.md` - Implementation phases and tasks

**See "CRITICAL: Checkpoint and Checklist System" section above for formats.**
**See @WORKFLOW_GUIDE.md for complete checkpoint workflow and examples.**

---

## Subagent Reference

**See .claude/SUBAGENT_GUIDE.md for quick reference card.**

### When to Use Each Subagent

| Subagent | When to Use | Key Responsibility |
|----------|-------------|-------------------|
| **checkpoint-manager** | Start/end of session | Create checkpoint, track progress |
| **django-architect** | New features, major refactoring | Design Django structure, multi-tenant patterns |
| **django-mentor** | Architecture questions, complex patterns | Provide guidance (uses context7) |
| **tdd-test-first** | Critical features (Standard Mode) | Write comprehensive tests (RED) |
| **django-implementer** | After tests fail or directly | Implement code (GREEN), CBV patterns, Signals |
| **security-auditor** | Before final commit | OWASP Top 10, XSS, CSRF, multi-tenant isolation |
| **accessibility-auditor** | After UI complete | WCAG 2.1 Level A compliance |
| **analytics-expert** | Analytics features | PageView tracking, queries, charts |
| **frontend-integrator** | HTMX + DaisyUI + Flatpickr | Validate integration & optimize, ES6 Modules |
| **git-workflow-manager** | After tests, after implementation | Manage commits |

### Subagent Communication Flow

**CRITICAL: checkpoint-manager is ALWAYS the FIRST subagent called (unless "do fast:" prefix).**

```
PROJECT_SPECS.md (item)
    ↓
[Planning Mode] (if needed)
    ↓
╔═══════════════════════════════════════════════════════════════════╗
║  checkpoint-manager (MANDATORY FIRST STEP)                        ║
║  - Check if .claude/doc/{feature}/ exists                         ║
║  - If NO: Create checkpoint.md + checklist.md                     ║
║  - If YES: Read and update checkpoint.md                          ║
║  - NEVER skip this step for implementation tasks                  ║
╚═══════════════════════════════════════════════════════════════════╝
    ↓
django-architect (design structure)
    ↓
tdd-test-first (RED phase) [Standard Mode only]
    ↓
git-workflow-manager (commit RED) [Standard Mode only]
    ↓
django-implementer (GREEN phase)
    ↓
security-auditor → Issues? → STOP → Fix → Re-audit
    | [OK]
accessibility-auditor → Issues? → STOP → Fix → Re-audit
    | [OK]
analytics-expert (if analytics feature)
    | [OK]
frontend-integrator → Issues? → STOP → Fix → Re-validate
    | [OK]
git-workflow-manager (commit GREEN or single commit)
    ↓
checkpoint-manager (update progress - MANDATORY END STEP)
```

**Enforcement Rule**: If you realize you started implementation WITHOUT calling checkpoint-manager first:
1. STOP implementation immediately
2. Create the missing documentation NOW
3. Document what was already done
4. Continue with remaining work

---

## Philosophy

> "I am Tony Stark, the architect.
> Claude Code is Jarvis, the orchestrator.
> Subagents are the specialized team.
>
> I decide. AI executes. I review. I iterate."

**Core principles:**
- Document first, code second
- Progress is measured by completed checklists
- Blockers and learnings are valuable documentation
- Every feature leaves a trail in `.claude/doc/`

**The result:** Professional, tested, secure code with complete documentation trail.
