# Research Documentation

This directory contains research documentation created by specialized agents **BEFORE** implementation.

---

## Documentation Classification

### Reusable Guides (Generic - Copy to Other Projects)

These guides are framework/pattern documentation applicable to ANY Django project:

| File | Description | Use Case |
|------|-------------|----------|
| `solid_ddd_patterns.md` | DDD + SOLID patterns implementation | Complex business logic |
| `python_professional_standards.md` | NASA + Wemake Python standards | Code quality standards |
| `django_cbv_patterns.md` | Django CBV patterns library | Multi-tenant CRUD views |
| `mercadopago_integration_guide.md` | MercadoPago payment integration | Payment processing (Argentina) |
| `google_oauth_integration_guide.md` | Google OAuth with django-allauth | Social authentication |
| `flatpickr_implementation_guide.md` | Flatpickr date picker best practices | Date range selection |
| `htmx_inline_script_globals.md` | HTMX + inline scripts patterns | Dynamic HTML rendering |
| `javascript_module_patterns_2025.md` | ES6 modules for Django (2025) | Modern JavaScript |
| `oauth_best_practices.md` | OAuth 2.0 best practices | Authentication security |
| `oauth_cookie_multitenant.md` | Cookie config for multi-tenant | Subdomain sessions |
| `oauth_signal_debugging.md` | django-allauth signal debugging | OAuth troubleshooting |

### Project-Specific Documentation (Feature Research)

These are research documents for specific features - NOT meant for reuse:

| Directory/File | Description |
|----------------|-------------|
| `accommodation_pricing/` | Pricing logic for accommodations |
| `accommodation_units_validation.md` | Unit availability validation |
| `payment_integration/` | Payment flow implementation |
| `ribera-018/` | Specific JIRA ticket research |

### Template Structure for New Features

When creating documentation for new features, use subdirectories:

```
.claude/doc/{feature_name}/
    backend.md      # Django models, managers, services, forms
    frontend.md     # UI components, JavaScript, HTMX
    security.md     # OWASP threats, mitigations
    analytics.md    # Tracking, metrics (if needed)
```

---

## Purpose

Agents follow a **research-first pattern**:

1. **Research**: Agent researches best practices, patterns, libraries
2. **Document**: Agent creates markdown file in `.claude/doc/{feature}/`
3. **Review**: User reviews and approves plan
4. **Implement**: Only after approval, implementer agent writes code

---

## Directory Structure

```
.claude/doc/
├── README.md (this file)
├── component_system/
│   ├── backend.md         # Django models, managers, services, forms
│   ├── frontend.md        # DaisyUI patterns, SortableJS config
│   ├── security.md        # XSS prevention, validation
│   └── analytics.md       # PageView tracking for components
├── reservation_system/
│   ├── backend.md         # Reservation model, availability logic, Celery tasks
│   ├── frontend.md        # Calendar UI, Flatpickr integration
│   ├── security.md        # Multi-tenant isolation, input validation
│   └── analytics.md       # Booking metrics, conversion tracking
├── page_builder_canvas/
│   ├── backend.md         # Component storage, rendering logic
│   ├── frontend.md        # Drag-and-drop implementation
│   ├── security.md        # CSRF in HTMX, component validation
│   └── analytics.md       # Track canvas interactions
└── {other_features}/
    ├── backend.md         # ALWAYS for features with Django logic
    ├── frontend.md        # If feature has UI
    ├── security.md        # ALWAYS (security-first)
    └── analytics.md       # If feature needs tracking
```

**Note**: `backend.md` should be created for ANY feature that involves Django models, forms, managers, services, or Celery tasks. This ensures professional backend architecture from the start.

---

## Documentation Templates

### `analytics.md` Template

Created by: `analytics-expert` agent

```markdown
# Analytics Research: {Feature Name}

## Overview
[What analytics are needed for this feature]

## Models Required

### Model: {ModelName}
**Purpose**: [What it tracks]

**Fields**:
- field_name (type): description
- field_name (type): description

**Indexes**:
- (field1, field2): reason

## Query Patterns

### Query 1: {Name}
**Purpose**: [What this query calculates]

**Implementation**:
```python
# Example query
```

**Performance**: [Expected query time, caching strategy]

## Chart.js Integration

### Chart 1: {Name}
**Type**: line/bar/pie
**Data source**: [Query that feeds this chart]
**Update frequency**: real-time/hourly/daily

**Configuration**:
```javascript
// Chart.js config
```

## Privacy Considerations

- [IP anonymization]
- [Data retention policy]
- [GDPR compliance]

## Recommendations

1. [Best practice 1]
2. [Best practice 2]
```

---

### `frontend.md` Template

Created by: `frontend-integrator` agent

```markdown
# Frontend Research: {Feature Name}

## Overview
[What UI components are needed]

## DaisyUI Components

### Component 1: {Name}
**DaisyUI Class**: [btn, card, hero, etc.]
**Purpose**: [What it does]

**HTML Structure**:
```html
<!-- DaisyUI markup -->
```

**Variants**:
- Primary: [description]
- Secondary: [description]

## JavaScript Integration

### SortableJS Configuration
**Purpose**: [Drag-and-drop behavior]

**Implementation**:
```javascript
new Sortable(element, {
    // config
});
```

### HTMX Integration
**Purpose**: [Dynamic updates]

**Implementation**:
```html
<div hx-get="..." hx-swap="...">
```

**CSRF Handling**:
```html
hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'
```

## Responsive Design

- Mobile (< 640px): [behavior]
- Tablet (640-1024px): [behavior]
- Desktop (> 1024px): [behavior]

## Accessibility

- Keyboard navigation: [how it works]
- Screen reader support: [ARIA labels]
- Focus management: [focus trapping, autofocus]

## Recommendations

1. [Best practice 1]
2. [Best practice 2]
```

---

### `backend.md` Template

Created by: `django-architect` agent

```markdown
# Backend Research: {Feature Name}

## Overview
[What backend functionality is needed]

## Django Models

### Model 1: {ModelName}
**Purpose**: [What this model represents]

**Fields**:
```python
class ModelName(models.Model):
    field_name = models.CharField(max_length=200)  # Description
    field_name = models.ForeignKey(...)  # Multi-tenant isolation
    # ... more fields

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['field1', 'field2']),
        ]
```

**Relationships**:
- 1:N with {OtherModel}: [reason]
- M:N with {OtherModel}: [reason]

**Validation**:
- Field-level: [constraints]
- Model-level: [clean() method logic]

## Managers and QuerySets

### Manager: {ModelName}Manager
**Purpose**: [Business logic encapsulation]

**Location**: `apps/{app}/managers.py`

**Methods**:
```python
class ModelNameManager(models.Manager):
    def method_name(self, param):
        """[What this method does]"""
        return self.get_queryset().filter(...)
```

### QuerySet: {ModelName}QuerySet
**Purpose**: [Filtering and aggregation logic]

**Location**: `apps/{app}/querysets.py`

**Methods**:
```python
class ModelNameQuerySet(models.QuerySet):
    def filter_method(self, param):
        """[What this filters]"""
        return self.filter(...)
```

## Services Layer

### Service: {ServiceName}
**Purpose**: [Complex business logic that spans multiple models]

**Location**: `src/services/{service_name}.py`

**Key Methods**:
```python
def process_feature(param1, param2):
    """
    [What this service does]

    Args:
        param1: [description]
        param2: [description]

    Returns:
        [what it returns]

    Raises:
        ValidationError: [when]
    """
    # Implementation
```

**Dependencies**:
- Uses: [which models/services]
- Called by: [which views/tasks]

## Forms/Serializers

### Form: {FormName}
**Purpose**: [User input validation]

**Location**: `apps/{app}/forms.py`

**Fields**:
```python
class FormName(forms.ModelForm):
    honeypot = forms.CharField(required=False)  # Bot detection

    class Meta:
        model = ModelName
        fields = ['field1', 'field2']

    def clean_field(self):
        # Field-level validation + XSS sanitization

    def clean(self):
        # Cross-field validation
```

**Validation Strategy**:
- Sanitization: [which fields, how]
- Business rules: [what rules]
- Multi-tenant: [site context required?]

## Celery Tasks

### Task: {task_name}
**Purpose**: [Async operation]

**Location**: `apps/{app}/tasks.py`

**Implementation**:
```python
@shared_task
def task_name(param_id):
    """
    [What this task does]

    Args:
        param_id: [description]
    """
    # Async logic
```

**Trigger**: [When/how this task is triggered]
**Retry Strategy**: [max_retries, backoff]
**Error Handling**: [what happens on failure]

## API Endpoints (if applicable)

### Endpoint: GET /api/{resource}/
**Purpose**: [What it returns]

**Authentication**: Required/Optional
**Permissions**: [who can access]

**Query Parameters**:
- param1 (type): [description]
- param2 (type): [description]

**Response**:
```python
{
    "field": "value",
    "nested": {
        "field": "value"
    }
}
```

**Filtering**: [How multi-tenant filtering is applied]

## Database Queries

### Query Pattern 1: {Name}
**Purpose**: [What it retrieves]

**Implementation**:
```python
# Optimized query
Model.objects.select_related('fk_field').prefetch_related('m2m_field').filter(
    site=site,  # Multi-tenant
    field=value
)
```

**Performance**:
- Expected time: [ms]
- Indexes used: [which indexes]
- N+1 prevention: [select_related/prefetch_related used]

## Multi-Tenant Isolation

**Critical Points**:
- All queries MUST filter by `site`
- Forms MUST receive `site` context
- Managers MUST enforce site filtering
- Admin MUST show only user's sites

**Pattern**:
```python
# In views
def view(request, site_slug):
    site = get_object_or_404(Site, slug=site_slug)
    # All queries: Model.objects.filter(site=site)

# In forms
form = FormName(request.POST, site=site)

# In managers
def get_queryset(self):
    return super().get_queryset().filter(site=self.site)
```

## Type Hints & Docstrings

**Google-style docstrings required**:
```python
def function_name(param: Type) -> ReturnType:
    """
    Brief description.

    Detailed explanation if needed.

    Args:
        param: Parameter description

    Returns:
        Return value description

    Raises:
        ExceptionType: When and why
    """
```

**Type hints everywhere**:
- Function parameters
- Return types
- Class attributes
- mypy strict mode must pass

## Testing Strategy

**Coverage target**: 70% (Standard Mode) / 40% (Fast Track)

**Test types needed**:
- Model tests: Field validation, methods, relationships
- Manager tests: Business logic, filtering
- QuerySet tests: Complex queries, multi-tenant isolation
- Form tests: Validation, sanitization, cross-field rules
- Service tests: Business logic, error handling
- Integration tests: End-to-end workflows

**Multi-tenant test pattern**:
```python
def test_multi_tenant_isolation(self):
    """User A cannot see User B's data"""
    site_a = Site.objects.create(...)
    site_b = Site.objects.create(...)

    data_a = Model.objects.filter(site=site_a)
    # Assert isolation
```

## Performance Considerations

**Database**:
- Indexes on: [which fields and why]
- Query optimization: [select_related, prefetch_related patterns]
- Caching: [what to cache, TTL]

**Celery**:
- Which operations must be async: [list]
- Task priorities: [high/normal/low for each]

## Security Checklist

- [ ] All queries filter by site (multi-tenant isolation)
- [ ] Forms sanitize input with bleach
- [ ] Type hints on all functions
- [ ] Docstrings on all public methods
- [ ] No raw SQL (use ORM parameterized queries)
- [ ] No print() statements (use logging)
- [ ] Proper error handling (try/except with logging)

## Recommendations

1. [Django/Python best practice 1]
2. [Architecture decision 1]
3. [Performance optimization 1]

## Open Questions

- [ ] Question 1?
- [ ] Question 2?
```

---

### `security.md` Template

Created by: `security-auditor` agent

```markdown
# Security Research: {Feature Name}

## Overview
[Security risks specific to this feature]

## Threats

### Threat 1: {Name}
**Risk Level**: Critical/High/Medium/Low
**Attack Vector**: [How attacker exploits]
**Impact**: [What happens if exploited]

## Mitigations

### Mitigation 1: {Name}
**Addresses**: [Which threat]
**Implementation**:
```python
# Code example
```

**Verification**:
```python
# Test example
```

## OWASP Mapping

- A01 (Broken Access Control): [how addressed]
- A03 (Injection): [how addressed]
- A05 (Security Misconfiguration): [how addressed]

## Security Checklist

- [ ] All user input validated
- [ ] All output escaped (XSS prevention)
- [ ] CSRF protection enabled
- [ ] Multi-tenant isolation enforced
- [ ] File uploads validated (if applicable)
- [ ] SQL injection prevented (parameterized queries)

## Recommendations

1. [Best practice 1]
2. [Best practice 2]
```

---

## Agent Workflows

### analytics-expert

**When to use**: Any feature that needs data tracking or visualization

**Workflow**:
1. Analyze feature requirements
2. Design models for tracking
3. Design query patterns
4. Design Chart.js visualizations
5. Create `.claude/doc/{feature}/analytics.md`
6. Present to user for approval

**Example**:
```bash
> Use analytics-expert to design PageView tracking for component system

# Agent researches and creates:
# .claude/doc/component_system/analytics.md

# User reviews and approves

# Then implementation happens
```

---

### frontend-integrator

**When to use**: Any feature with UI components

**Workflow**:
1. Analyze UI requirements
2. Research DaisyUI components
3. Design responsive layouts
4. Design JavaScript interactions (SortableJS, HTMX)
5. Create `.claude/doc/{feature}/frontend.md`
6. Present to user for approval

**Example**:
```bash
> Use frontend-integrator to research DaisyUI patterns for hero component

# Agent researches and creates:
# .claude/doc/component_system/frontend.md

# User reviews and approves

# Then implementation happens
```

---

### security-auditor

**When to use**: Before implementing any feature (research mode)

**Workflow**:
1. Analyze feature for security risks
2. Research threat vectors
3. Design mitigations
4. Create security checklist
5. Create `.claude/doc/{feature}/security.md`
6. Present to user for approval

**Example**:
```bash
> Use security-auditor to research security risks for file upload feature

# Agent researches and creates:
# .claude/doc/file_uploads/security.md

# User reviews and approves

# Then implementation happens with security built-in
```

---

### django-architect

**When to use**: Any feature that needs backend Django implementation (models, managers, services, forms, Celery)

**Workflow**:
1. Analyze backend requirements
2. Design Django models with proper relationships
3. Design managers/querysets for business logic
4. Design services layer for complex operations
5. Design forms with validation and sanitization
6. Design Celery tasks if async needed
7. Plan database indexes and query optimization
8. Ensure multi-tenant isolation patterns
9. Create `.claude/doc/{feature}/backend.md`
10. Present to user for approval

**Example**:
```bash
> Use django-architect to research backend architecture for reservation system

# Agent researches and creates:
# .claude/doc/reservation_system/backend.md

# Document includes:
# - Reservation model with proper fields
# - ReservationManager with check_availability method
# - ReservationQuerySet with filtering logic
# - reservation_services.py for complex business logic
# - ReservationForm with validation
# - send_reservation_confirmation Celery task
# - Multi-tenant isolation patterns
# - Database indexes needed
# - Type hints and docstrings required

# User reviews and approves

# Then django-implementer uses this as blueprint
```

**What backend.md should cover**:
- Models (fields, relationships, validation, indexes)
- Managers (business logic methods)
- QuerySets (filtering, aggregation)
- Services (complex operations spanning multiple models)
- Forms (validation, sanitization, multi-tenant context)
- Celery Tasks (async operations, retry strategy)
- Database queries (optimization, N+1 prevention)
- Multi-tenant isolation (all critical points)
- Type hints & docstrings (Google style)
- Testing strategy (coverage target, test types)
- Performance considerations (indexes, caching)
- Security checklist (input validation, query filtering)

---

## Benefits of Research-First

### Before (Code-First)
```
1. Implement feature
2. Tests fail or security issues found
3. Refactor to fix
4. Waste time rewriting
```

### After (Research-First)
```
1. Research and document approach
2. User reviews and approves
3. Implement correctly the first time
4. Tests pass, security built-in
```

### Time Saved
- **Planning**: +30 minutes (upfront)
- **Implementation**: -2 hours (less rework)
- **Debugging**: -1 hour (fewer bugs)
- **Net savings**: ~2.5 hours per feature

---

## Example: Component System

### Step 1: Research (Day 1)
```bash
> Use frontend-integrator to research DaisyUI patterns for components

# Creates: .claude/doc/component_system/frontend.md
```

**Output** (`.claude/doc/component_system/frontend.md`):
```markdown
# Frontend Research: Component System

## Overview
Need navbar, hero, card components with DaisyUI classes.

## DaisyUI Components

### Navbar
**DaisyUI Class**: navbar
**HTML Structure**:
```html
<div class="navbar bg-base-100">
  <a class="btn btn-ghost text-xl">Logo</a>
</div>
```

## Recommendations
1. Use navbar-center for centered layout
2. Use dropdown for mobile menu
3. Test on mobile/tablet/desktop
```

### Step 2: Review (Day 1)
User reads `.claude/doc/component_system/frontend.md` and approves.

### Step 3: Implement (Day 2)
```bash
> Use django-implementer to implement navbar component following .claude/doc/component_system/frontend.md
```

Implementation uses exact patterns from research document.

### Step 4: Result
- ✅ Correct DaisyUI classes used
- ✅ Responsive design works
- ✅ No rework needed
- ✅ Time saved

---

## Documentation Maintenance

### When to Update

- **User decision changes approach**: Update relevant .md file
- **New pattern discovered**: Add to .md file
- **Security issue found**: Document in security.md
- **Performance issue found**: Document in analytics.md or frontend.md

### How to Update

```bash
# Edit the file directly
> Edit .claude/doc/{feature}/{aspect}.md

# Or ask agent to update
> Use {agent-name} to update research for {feature} based on {new information}
```

### Version Control

All documentation files are committed to git:
```bash
git add .claude/doc/
git commit -m "docs: update {feature} research with {change}"
```

---

## Integration with Context Sessions

Context session files reference documentation:

```markdown
# Context Session: Component System

## Research Documentation
- [Frontend patterns](.claude/doc/component_system/frontend.md)
- [Analytics tracking](.claude/doc/component_system/analytics.md)
- [Security measures](.claude/doc/component_system/security.md)
```

---

## Quick Reference

### Create New Feature Documentation
```bash
mkdir -p .claude/doc/{feature_name}

# Research each aspect
> Use analytics-expert to research {feature_name}
> Use frontend-integrator to research {feature_name}
> Use security-auditor to research {feature_name}
```

### Review All Documentation
```bash
# List all documentation
find .claude/doc -name "*.md" -not -name "README.md"

# Read specific feature
cat .claude/doc/{feature_name}/*.md
```

### Reference in Implementation
```bash
> Use django-implementer to implement {feature_name} following:
- .claude/doc/{feature_name}/frontend.md
- .claude/doc/{feature_name}/security.md
```

---

## Best Practices

1. **Always research before implementing** (especially for new features)
2. **Keep documentation concise** (1-2 pages max per file)
3. **Include code examples** (easier to understand than prose)
4. **Document why, not just what** (explain rationale)
5. **Update when decisions change** (keep docs current)
6. **Reference in context sessions** (link to research)
7. **Commit to git** (version control for decisions)

---

## Troubleshooting

### Agent Skips Research Phase

**Problem**: Agent implements directly without creating documentation.

**Solution**:
```bash
> STOP. Use {agent-name} to RESEARCH {feature} first.
> Create .claude/doc/{feature}/{aspect}.md before implementing.
```

### Documentation Too Generic

**Problem**: Agent creates generic documentation that doesn't help implementation.

**Solution**:
```bash
> Make research more specific. Include:
- Code examples
- Specific DaisyUI classes to use
- Exact query patterns
- Security mitigations with code
```

### Multiple Agents, Conflicting Docs

**Problem**: frontend-integrator and security-auditor recommend different approaches.

**Solution**:
1. Read both documents
2. Identify conflict
3. Ask user to decide
4. Update both documents to reflect decision

---

## Resources

- **DaisyUI Docs**: https://daisyui.com/
- **HTMX Docs**: https://htmx.org/
- **SortableJS Docs**: https://github.com/SortableJS/Sortable
- **Chart.js Docs**: https://www.chartjs.org/
- **OWASP Top 10**: https://owasp.org/www-project-top-ten/
