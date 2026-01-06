# Workflow Guide

---

## Development Modes

Use **Hybrid Mode**: Standard Mode for critical features + Fast Track Mode for secondary features.

---

## Decision Matrix: Standard vs Fast Track

Use this matrix BEFORE starting each feature:

| Question | If YES | Mode |
|----------|--------|------|
| Handles sensitive data? | YES | **Standard** |
| Handles user input without sanitization? | YES | **Standard** |
| Is security-critical logic? | YES | **Standard** |
| Can cause data breach if it fails? | YES | **Standard** |
| Hard to debug after deployment? | YES | **Standard** |
| Affects multi-tenant isolation? | YES | **Standard** |
| Is complex business logic? | YES | **Standard** |
| Is simple UI or admin? | YES | **Fast Track** |
| Is static template with editable content? | YES | **Fast Track** |
| Immediate visual feedback only? | YES | **Fast Track** |

### Examples by Mode

**Standard Mode Features:**
- Authentication systems (OAuth, login, sessions)
- Multi-tenant data isolation
- API integrations with external systems
- Payment processing
- File uploads
- Business logic with validation

**Fast Track Mode Features:**
- Admin interfaces
- UI templates and styling
- Simple CRUD operations
- Configuration pages
- Static content
- Dashboard displays

---

## STANDARD MODE - Full TDD Workflow

### When to Use

- Security-critical features (OAuth, multi-tenant isolation, file uploads)
- Complex business logic (API integrations, data processing)
- User data handling
- Multi-tenant data isolation
- Features that are hard to debug after deployment

### Workflow (10 Steps)

#### 1. Planning (Optional)

```bash
> Planning mode: Design feature architecture
```

#### 2. Create Documentation

Create in `.claude/doc/{feature_name}/`:

```bash
# Create checkpoint.md and checklist.md
.claude/doc/{feature_name}/checkpoint.md
.claude/doc/{feature_name}/checklist.md
```

**Expected structure:**
```
.claude/doc/{feature_name}/
    checkpoint.md    # Progress tracking
    checklist.md     # Implementation tasks
```

#### 3. Architecture Design

```bash
> Use django-architect to design feature architecture
```

#### 4. Tests (RED)

```bash
> Use tdd-test-first to create comprehensive tests
```

**Expected:**
- Tests created and FAILING (RED)
- Update checklist.md with test tasks

#### 5. Commit Tests

```bash
> Use git-workflow-manager to commit tests
```

```bash
# Pre-commit checks
grep -r "print(" apps/ src/  # Must be empty
black .
flake8
mypy .

# Commit
git add tests/
git commit -m "TICKET-XXX: add feature tests (RED)"
git push
```

#### 6. Implementation (GREEN)

```bash
> Use django-implementer to implement code to pass all tests
```

**Expected:**
- Code implemented
- Tests PASSING (GREEN)
- Update checklist.md with implementation tasks

#### 7. Security Audit

```bash
> Use security-auditor to check implementation
```

**If issues found:**
```
[STOP]
1. Fix issues
2. Update tests if needed
3. Re-run security-auditor
4. Update checkpoint.md with blocker and solution
```

#### 8. Accessibility Audit (UI features only)

```bash
> Use accessibility-auditor to check UI
```

**Skip if no visible UI.**

#### 9. Commit Implementation

```bash
> Use git-workflow-manager to commit implementation
```

```bash
# Pre-commit checks
grep -r "print(" apps/ src/
black .
flake8
mypy .
python manage.py test

# Commit
git add .
git commit -m "TICKET-XXX: implement feature (GREEN)"
git push
```

#### 10. Update Documentation

Update in `.claude/doc/{feature_name}/`:
- `checkpoint.md` - Add checkpoint entry with learnings
- `checklist.md` - Mark all tasks complete

**Total:** 2 commits (RED + GREEN), ~4-6 hours per critical feature

---

## FAST TRACK MODE - Rapid Development

### When to Use

- UI templates and styling
- Simple CRUD operations
- Admin interfaces
- Configuration pages
- Static content templates
- Email templates

### Workflow (6 Steps)

#### 1. Create Documentation

Create in `.claude/doc/{feature_name}/`:

```bash
.claude/doc/{feature_name}/checkpoint.md
.claude/doc/{feature_name}/checklist.md
```

#### 2. Architecture (Quick Review)

```bash
> Use django-architect to review implementation approach
```

#### 3. Direct Implementation (no tests first)

```bash
> Use django-implementer to implement feature
```

**Implement directly, test manually in browser.**

#### 4. Frontend Validation (if UI)

```bash
> Use frontend-integrator to validate UI integration
```

#### 5. Commit

```bash
> Use git-workflow-manager to commit
```

```bash
# Pre-commit checks
grep -r "print(" apps/ src/
black .
python manage.py test  # Ensure no breaks

# Commit
git add .
git commit -m "TICKET-XXX: add feature"
git push
```

#### 6. Update Documentation

Update in `.claude/doc/{feature_name}/`:
- `checkpoint.md` - Add entry with learnings
- `checklist.md` - Mark tasks complete

**Total:** 1 commit, ~1-2 hours per simple feature

---

## HYBRID MODE - Recommended

### Strategy

Combine Standard Mode for critical features with Fast Track Mode for secondary features.

**Critical Features (Standard Mode):**
- Authentication and authorization
- Multi-tenant data isolation
- API integrations
- Payment processing
- File uploads
- Complex business logic

**Secondary Features (Fast Track Mode):**
- Admin interfaces
- UI templates
- Dashboard displays
- Configuration pages
- Static content

### Quality Checkpoints

**After each major feature:**
```bash
# Check coverage
coverage run --source='.' manage.py test
coverage report
# Target: 50%+ overall

# Security scan
bandit -r apps/ src/ -ll

# Manual testing
# - Core workflows work
# - Multi-tenant isolation verified
# - No data leakage
```

---

## Checkpoint System

### Location

All checkpoints go in `.claude/doc/{feature_name}/`:

```
.claude/doc/{feature_name}/
    checkpoint.md    # Progress tracking, blockers, learnings
    checklist.md     # Implementation phases and tasks
```

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
- Decision: Rationale

### Blockers Encountered
- **Problem:** Description
- **Solution:** How resolved
- **Time lost:** X min

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

---

## Fase 1: {Phase Name}

- [x] Task 1
- [x] Task 2
- [ ] Task 3

---

## Files Created/Modified

| File | Action | Description |
|------|--------|-------------|
| path/to/file.py | CREATED | Description |
```

### When to Update

| Event | Action |
|-------|--------|
| Feature starts | Create checkpoint.md and checklist.md |
| Task completed | Update checklist.md |
| Blocker found | Add to checkpoint.md immediately |
| Session ends | Add checkpoint entry |
| Context summary | Update checkpoint.md first |
| Feature done | Mark all complete, document learnings |

### Existing Examples

See implementation examples in:
- `.claude/doc/galicia_integration/checkpoint.md`
- `.claude/doc/galicia_integration/checklist.md`

---

## Git Strategy

### Branch Strategy

**Option 1: Direct to main (solo developer)**
```bash
git checkout main
# ... work ...
git commit -m "TICKET-XXX: implement feature"
git push origin main
```

**Option 2: Feature branches (team or experiments)**
```bash
git checkout -b feature/{feature_name}
# ... work ...
git commit -m "TICKET-XXX: implement feature (GREEN)"
git checkout main
git merge feature/{feature_name} --no-ff
git push origin main
```

### Commit Messages

**Standard Mode (2 commits):**
```bash
# Tests
git commit -m "TICKET-XXX: add feature tests (RED)"

# Implementation
git commit -m "TICKET-XXX: implement feature (GREEN)"
```

**Fast Track Mode (1 commit):**
```bash
git commit -m "TICKET-XXX: add feature"
```

**Hotfix:**
```bash
git commit -m "TICKET-XXX: fix critical issue"
```

---

## Testing Strategy

### Coverage Requirements

**Overall target:** 50%+ (mix of Standard 70% + Fast Track 40%)

**Priority coverage:**
1. Business logic (70%+)
2. Multi-tenant isolation (80%+)
3. API integrations (70%+)
4. File upload validation (75%+)
5. Authentication flows (75%+)

### Test Organization

```
tests/
    {app}/
        test_models.py         # Model validation
        test_views.py          # View logic
        test_services.py       # Service layer
    security/
        test_multi_tenant.py   # Data isolation
        test_input_validation.py # XSS, injection
    integration/
        test_workflows.py      # End-to-end flows
```

### Running Tests

```bash
# All tests
python manage.py test

# Specific app
python manage.py test tests.{app}

# Specific file
python manage.py test tests.security.test_multi_tenant

# With coverage
coverage run --source='.' manage.py test
coverage report
coverage html  # Generate HTML report
```

---

## Troubleshooting

### Planning Mode No Activa

**Solución:** Activar manualmente:
```bash
> Planning mode: [describe decision needed]
```

### Tests Failing Unexpectedly

**Check:**
1. Database state (migrations applied?)
2. Fixtures loaded correctly?
3. CSRF token in test client?
4. Multi-tenant filters applied?
5. Timezone settings correct?

### HTMX Not Updating

**Common issues:**
1. Wrong `hx-target` selector
2. Missing CSRF token in headers
3. View not returning HTML fragment
4. JavaScript error blocking

**Debug:**
```javascript
// Add to base.html
htmx.on('htmx:responseError', function(evt) {
    console.error('HTMX error:', evt.detail);
});
```

### DaisyUI Not Styling

**Check:**
1. CDN link in base.html?
2. `data-theme` attribute set?
3. Class names correct?
4. Tailwind config loaded?

### OAuth Google Not Working

**Check:**
1. GOOGLE_CLIENT_ID and GOOGLE_CLIENT_SECRET in .env?
2. Redirect URI configured in Google Console?
3. django-allauth installed and configured?
4. Social app created in admin?

### Multi-tenant Data Leakage

**Check:**
1. All queries filter by request.site?
2. Middleware setting request.site correctly?
3. Admin queryset overridden with site filter?
4. Form validation checking site ownership?

---

## Daily Routine

### Start of Session

1. Read `.claude/doc/{feature}/checkpoint.md` for current state
2. Review pending tasks in `checklist.md`
3. Check for blockers
4. Plan work items

### During Development

1. Work incrementally
2. Test frequently
3. Commit often
4. Document blockers immediately in checkpoint.md
5. Update checklist.md as tasks complete

### End of Session

1. Update checkpoint.md with:
   - Completed items
   - Blockers resolved
   - Learnings
   - Next session plan
2. Update checklist.md (mark complete tasks)
3. Push all commits
4. Quick coverage check

---

## Philosophy

> "Standard Mode for what hurts, Fast Track for what's visible."

**Standard Mode = 70% coverage, 2 commits** - Security, business logic, core features

**Fast Track Mode = 40% coverage, 1 commit** - UI, styling, admin interfaces

**Hybrid Mode = 50% coverage, flexible commits** - Intelligent mix based on criticality

**Document First** - Every feature leaves a trail in `.claude/doc/`

**Result:** Professional, tested code with complete documentation trail.
