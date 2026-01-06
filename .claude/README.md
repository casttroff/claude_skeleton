# Claude Code Configuration

This directory contains configuration and workflow files for Claude Code development.

---

## Directory Structure

```
.claude/
├── agents/              # Specialized sub-agents for different concerns
├── commands/            # Slash commands for common workflows
├── sessions/            # Context session files for features
├── doc/                 # Research documentation from agents
├── hooks/               # Shell scripts triggered by workflow events
├── settings.json        # Permissions, MCP servers, hooks config
└── README.md            # This file
```

---

## Agents

Specialized sub-agents for different aspects of development:

| Agent | Purpose | Output |
|-------|---------|--------|
| **checkpoint-manager** | Create/update daily checkpoints | `checkpoints/checkpoint_day_X.md` |
| **django-architect** | Design Django structure and models | Documentation + recommendations |
| **django-mentor** | Answer Django/Python questions | Guidance + context7 docs |
| **security-auditor** | Audit for OWASP Top 10, XSS, CSRF | Security report + fixes |
| **analytics-expert** | Design analytics features | `.claude/doc/{feature}/analytics.md` |
| **frontend-integrator** | Validate DaisyUI + HTMX + SortableJS | `.claude/doc/{feature}/frontend.md` |
| **tdd-test-first** | Create comprehensive tests (RED) | Test files |
| **django-implementer** | Implement code to pass tests (GREEN) | Implementation |
| **accessibility-auditor** | Check WCAG 2.1 Level A | A11y report + fixes |
| **git-workflow-manager** | Manage commits (no PRs in this project) | Git operations |

### Agent Workflow

```
Planning
    ↓
django-architect (design structure)
    ↓
analytics-expert (if analytics feature) → Research → .claude/doc/{feature}/analytics.md
    ↓
frontend-integrator (if UI feature) → Research → .claude/doc/{feature}/frontend.md
    ↓
tdd-test-first (create tests - RED)
    ↓
django-implementer (implement - GREEN)
    ↓
security-auditor (audit) → If FAIL: STOP and fix
    ↓
accessibility-auditor (if UI) → If FAIL: STOP and fix
    ↓
git-workflow-manager (commit)
    ↓
checkpoint-manager (update checkpoint)
```

---

## Commands

Slash commands for common workflows:

### `/checkpoint`
Create or update checkpoint for current day.

**Usage**:
```bash
> /checkpoint
```

**Output**: `checkpoints/checkpoint_day_X.md`

**When**: End of each work session, before long break, after milestone

---

### `/component-new`
Scaffold a new DaisyUI component.

**Usage**:
```bash
> /component-new
```

**Workflow**:
1. Ask for component name and fields
2. Research DaisyUI patterns
3. Create Pydantic schema
4. Create DRF serializer
5. Create template
6. Create tests (RED)
7. Register in COMPONENT_REGISTRY

**Output**:
- Schema, serializer, template, tests
- Tests FAILING (ready for implementation)

---

### `/security-check`
Run comprehensive security audit.

**Usage**:
```bash
> /security-check
```

**Checks**:
- Bandit (Python security linter)
- Safety (dependency vulnerabilities)
- XSS protection
- File upload validation
- CSRF in HTMX
- Multi-tenant isolation
- SQL injection

**Output**: `security_report.md` with PASS/FAIL verdict

**When**: After implementing feature, before commit, weekly

---

## Sessions

Context session files preserve feature context across work sessions.

### Structure

```
.claude/sessions/
├── TEMPLATE.md
├── context_session_component_system.md
├── context_session_page_builder_canvas.md
└── context_session_analytics_dashboard.md
```

### When to Create

- At start of each ETAPA
- Before long planning sessions
- After each checkpoint
- When returning to feature after pause

### Template

See `sessions/TEMPLATE.md` for complete structure.

**Key sections**:
- Feature Overview
- Status and Timeline
- Plan (phases)
- User Decisions
- Technical Challenges
- Next Steps

---

## Documentation

Research documentation from agents.

### Structure

```
.claude/doc/
├── component_system/
│   ├── analytics.md       # From analytics-expert
│   ├── frontend.md        # From frontend-integrator
│   └── security.md        # From security-auditor
├── page_builder_canvas/
│   ├── analytics.md
│   ├── frontend.md
│   └── security.md
└── README.md
```

### Workflow

**Research-First Pattern**:
1. Agent researches (e.g., DaisyUI patterns)
2. Agent creates `.claude/doc/{feature}/frontend.md`
3. User reviews plan
4. User approves
5. Implementer codes it

**Benefits**:
- Deliberate design decisions
- Reusable documentation
- No "code first, think later"

---

## MCP Servers

Model Context Protocol servers for extended capabilities.

### Configured Servers

| Server | Purpose | Usage |
|--------|---------|-------|
| **context7** | Real-time docs for Django, DaisyUI, SortableJS | `django-mentor`, `frontend-integrator` |
| **playwright** | Browser automation for E2E tests | Future: E2E testing |

### Usage Example

```bash
# django-mentor queries Django 5.1 docs
> Use django-mentor: How to implement multi-tenant with django.contrib.sites?

# context7 fetches latest Django docs
# django-mentor provides answer with examples
```

---

## Settings

`.claude/settings.json` configures:

### Permissions

**Allowed**:
- File operations (mkdir, Write, Edit, Read)
- Django commands (test, migrate, makemigrations)
- Quality tools (black, flake8, mypy, bandit)
- Git operations (add, commit, push, status, diff)
- MCP tools (context7, playwright)

**Denied**:
- Destructive operations (rm -rf, DROP TABLE)
- Force operations (git push --force, git reset --hard)

### Hooks

**Stop** (session complete):
- Reminder to run `/checkpoint`
- Show checkpoint list
- Show context sessions

**SubagentStop** (subagent finished):
- Reminder to review documentation

---

## Hooks

Shell scripts triggered by workflow events.

### Future Hooks

**`on-checkpoint.sh`**:
- Triggered after checkpoint creation
- Generates coverage report
- Saves to `checkpoints/day_X_coverage.md`

**`on-test-pass.sh`**:
- Triggered after all tests pass
- Runs bandit security check
- Generates `security_report.txt`

---

## Workflow Examples

### Example 1: Create New Component (Standard Mode)

```bash
# 1. Create context session
> Create context session for hero component

# 2. Research phase
> Use frontend-integrator to research DaisyUI hero patterns

# 3. Scaffold component
> /component-new
> Component name: hero
> Fields: title, subtitle, button_text, button_url

# 4. Review scaffold (tests are RED)
python manage.py test tests.test_component_hero

# 5. Implement
> Use django-implementer to implement hero component

# 6. Tests pass (GREEN)
python manage.py test tests.test_component_hero

# 7. Security check
> /security-check

# 8. If PASS, commit
git add .
git commit -m "feat: add hero component (GREEN)"

# 9. Update checkpoint
> /checkpoint
```

### Example 2: Implement Analytics Feature (Fast Track Mode)

```bash
# 1. Create context session
> Create context session for analytics dashboard

# 2. Research phase
> Use analytics-expert to design PageView tracking

# 3. Implement directly (Fast Track)
> Use django-implementer to implement PageView model and queries

# 4. Quick tests (basic coverage)
python manage.py test tests.test_analytics

# 5. Security check
> /security-check

# 6. If PASS, commit
git add .
git commit -m "feat: add analytics dashboard"

# 7. Update checkpoint
> /checkpoint
```

---

## Best Practices

### Context Sessions

- **Create at start** of each ETAPA
- **Update after** each checkpoint
- **Reference in** commits

### Commands

- **Use `/checkpoint`** at end of every session
- **Use `/security-check`** before every commit
- **Use `/component-new`** for all new components

### Documentation

- **Review `.claude/doc/`** before implementing
- **Update after** user decisions
- **Reference in** context sessions

### Agents

- **Let agents research first** (don't skip documentation phase)
- **Review agent plans** before approving implementation
- **Trust agent expertise** in their domain

---

## Troubleshooting

### Command Not Found

```bash
# If /checkpoint doesn't work
> Read .claude/commands/checkpoint.md and follow workflow
```

### Context Lost

```bash
# Restore context from session file
> Read .claude/sessions/context_session_{feature}.md
```

### Agent Documentation Missing

```bash
# Check if doc was created
ls -la .claude/doc/{feature}/

# If missing, re-run agent in research mode
> Use {agent-name} to research {topic}
```

---

## Quick Reference

### Daily Workflow

```bash
# Morning: Review yesterday's checkpoint
cat checkpoints/checkpoint_day_$(date +%d).md

# During: Work on features with agents
# ...

# Evening: Create checkpoint
> /checkpoint
```

### Before Every Commit

```bash
# 1. Tests pass
python manage.py test

# 2. Security check
> /security-check

# 3. If PASS, commit
git add .
git commit -m "..."
```

### Weekly Review

```bash
# Review all checkpoints
ls -la checkpoints/

# Review all context sessions
ls -la .claude/sessions/

# Review all documentation
ls -la .claude/doc/
```

---

## Resources

- **PROJECT_SPECS.md**: Feature requirements
- **TECHNICAL_SPECS.md**: Technical architecture
- **WORKFLOW_GUIDE.md**: Standard vs Fast Track modes
- **CLAUDE.md**: Complete project configuration
- **WORKFLOW_IMPROVEMENTS.md**: Patterns from claude-code-demo
