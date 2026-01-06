# Security Check Command

Run comprehensive security audit on current changes.

---

## Workflow

You will perform a multi-layered security audit focusing on pagebuilder-specific risks.

### Step 1: Automated Scanning

#### Bandit (Python Security Linter)
```bash
bandit -r apps/ src/ -ll -f txt
```

Expected focus:
- SQL injection patterns
- Hardcoded secrets
- Unsafe file operations
- Eval/exec usage
- Weak crypto

#### Safety (Dependency Vulnerability Scanner)
```bash
safety check --json
```

Expected focus:
- Known CVEs in dependencies
- Outdated packages with security issues

### Step 2: Manual Code Review (Use security-auditor Agent)

```bash
> Use security-auditor to audit current changes with focus on:
- XSS in component rendering
- File upload validation
- CSRF protection in HTMX
- Multi-tenant isolation
```

### Step 3: Pagebuilder-Specific Checks

#### 3.1 XSS in Component Rendering
**Risk**: User-provided content rendered without escaping

**Check**:
```bash
# Search for potentially dangerous template tags
grep -r "safe\|mark_safe\|autoescape off" templates/components/
```

**Expected**: NONE or minimal with clear justification

**Agent validation**:
- All user content uses `{{ variable }}` (auto-escaped)
- NO use of `{{ variable|safe }}` unless HTML is sanitized with bleach
- NO `{% autoescape off %}`

#### 3.2 File Upload Security
**Risk**: Malicious file uploads (XSS via SVG, RCE via scripts)

**Check**:
```python
# Verify FileUploadValidator is used
grep -r "FileUploadValidator" apps/pages/
```

**Expected**: All file fields use CustomFileValidator

**Agent validation**:
- File extension whitelist enforced
- MIME type validation with python-magic
- File content scanning for malicious patterns
- Max file size enforced
- Files stored outside webroot

#### 3.3 CSRF Protection in HTMX
**Risk**: CSRF attacks via HTMX POST requests

**Check**:
```bash
# Verify CSRF token in HTMX requests
grep -r "hx-post\|hx-put\|hx-delete" templates/
```

**Expected**: All non-GET HTMX requests include CSRF token

**Agent validation**:
```html
<!-- GOOD -->
<form hx-post="{% url 'pages:save' %}" hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>

<!-- BAD -->
<form hx-post="{% url 'pages:save' %}">  <!-- Missing CSRF! -->
```

#### 3.4 Multi-Tenant Isolation
**Risk**: Users accessing other tenants' data

**Check**:
```bash
# Verify all queries filter by site
grep -r "objects.filter\|objects.get" apps/pages/views.py
```

**Expected**: ALL queries include `site=request.site`

**Agent validation**:
- Middleware sets `request.site`
- All model queries filter by site
- No direct ID access without site check
- Admin properly filters by site

#### 3.5 SQL Injection
**Risk**: Raw SQL queries with unsanitized input

**Check**:
```bash
# Search for raw SQL
grep -r "raw\|execute\|cursor" apps/ src/
```

**Expected**: NONE or parameterized queries only

**Agent validation**:
```python
# GOOD (parameterized)
Page.objects.raw('SELECT * FROM pages WHERE site_id = %s', [site_id])

# BAD (vulnerable)
Page.objects.raw(f'SELECT * FROM pages WHERE site_id = {site_id}')
```

### Step 4: OWASP Top 10 Checklist

Run through critical OWASP categories:

#### A01: Broken Access Control
- [ ] All views check `request.site`
- [ ] Admin properly scoped
- [ ] No direct object references without validation

#### A03: Injection
- [ ] No raw SQL with f-strings
- [ ] All user input sanitized in templates
- [ ] Component JSON validated with Pydantic

#### A05: Security Misconfiguration
- [ ] DEBUG = False in production
- [ ] SECRET_KEY not in source code
- [ ] ALLOWED_HOSTS configured
- [ ] SECURE_SSL_REDIRECT = True

#### A07: Identification and Authentication Failures
- [ ] Strong password validators
- [ ] Session timeout configured
- [ ] Multi-tenant session isolation

#### A08: Software and Data Integrity Failures
- [ ] All file uploads validated
- [ ] Component data validated with Pydantic
- [ ] Integrity checks on published pages

### Step 5: Generate Report

Create structured report at `security_report.md`:

```markdown
# Security Audit Report

## Date
YYYY-MM-DD HH:MM

## Scope
Files changed since last checkpoint:
- [List of files]

## Automated Scan Results

### Bandit
- Critical: X
- High: Y
- Medium: Z

### Safety
- Vulnerabilities: X

## Manual Review Results

### XSS Protection
**Status**: PASS/FAIL
**Issues**: [List if any]

### File Upload Security
**Status**: PASS/FAIL
**Issues**: [List if any]

### CSRF Protection
**Status**: PASS/FAIL
**Issues**: [List if any]

### Multi-Tenant Isolation
**Status**: PASS/FAIL
**Issues**: [List if any]

### SQL Injection
**Status**: PASS/FAIL
**Issues**: [List if any]

## OWASP Top 10 Compliance

- A01: PASS/FAIL
- A03: PASS/FAIL
- A05: PASS/FAIL
- A07: PASS/FAIL
- A08: PASS/FAIL

## Critical Issues (Must Fix Immediately)
1. [Issue description]
   - **Risk**: [Impact]
   - **Location**: [File:line]
   - **Fix**: [Recommended solution]

## Medium Issues (Fix Before Commit)
1. [Issue description]

## Low Issues (Technical Debt)
1. [Issue description]

## Recommendations
1. [Best practice suggestion]
2. [Improvement opportunity]

## Final Verdict
**PASS** ✅ - Safe to commit
**FAIL** ❌ - Must fix critical issues before committing
```

### Step 6: Action Based on Result

#### If PASS ✅
```bash
echo "✅ Security audit passed"
git add security_report.md
git commit -m "security: audit passed for [feature]"
```

#### If FAIL ❌
```bash
echo "❌ Security issues found. STOP and fix before committing."
echo "See security_report.md for details"

# DO NOT commit until issues resolved
# Fix issues, then re-run /security-check
```

---

## Usage Examples

### Example 1: After Implementing Component
```bash
# Just implemented hero component
> /security-check

# Claude runs audit
> Running security audit on hero component...

# Output
✅ XSS Protection: PASS (all output escaped)
✅ File Upload: N/A
✅ CSRF: PASS (no forms)
✅ Multi-Tenant: PASS (no DB queries)
✅ SQL Injection: PASS (no raw SQL)

Final Verdict: PASS ✅
```

### Example 2: After Implementing File Upload
```bash
# Just implemented page thumbnail upload
> /security-check

# Claude runs audit
> Running security audit on file upload...

# Output
❌ File Upload: FAIL
  Issue: Missing MIME type validation
  Location: apps/pages/forms.py:45
  Fix: Add CustomFileValidator with allowed_mimetypes=['image/jpeg', 'image/png']

Final Verdict: FAIL ❌
Must fix before committing.
```

---

## When to Use

- **After implementing any feature** (before commit)
- **After modifying existing code** that handles:
  - User input
  - File uploads
  - Database queries
  - Template rendering
- **Before creating PR** (final check)
- **Weekly** (comprehensive audit of all changes)

---

## Integration with Workflow

### Standard Mode
```
1. tdd-test-first (create tests)
2. django-implementer (implement)
3. /security-check ← Run here
4. If PASS → commit GREEN
5. If FAIL → fix → re-run /security-check
```

### Fast Track Mode
```
1. Implement feature
2. /security-check ← Run here
3. If PASS → commit
4. If FAIL → fix → re-run /security-check
```

---

## Common Issues and Fixes

### Issue: XSS via |safe filter
```python
# BAD
{{ component.html_content|safe }}

# GOOD
import bleach
clean_html = bleach.clean(
    component.html_content,
    tags=['p', 'h2', 'strong', 'em'],
    strip=True
)
```

### Issue: Missing CSRF in HTMX
```html
<!-- BAD -->
<form hx-post="{% url 'save' %}">

<!-- GOOD -->
<form hx-post="{% url 'save' %}" hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>
```

### Issue: SQL Injection
```python
# BAD
Page.objects.raw(f"SELECT * FROM pages WHERE id = {page_id}")

# GOOD
Page.objects.raw("SELECT * FROM pages WHERE id = %s", [page_id])
```

### Issue: Broken Multi-Tenant
```python
# BAD
page = Page.objects.get(id=page_id)

# GOOD
page = Page.objects.get(id=page_id, site=request.site)
```

---

## False Positives

Some warnings may be acceptable:

### Safe |safe Usage
```python
# OK if content is from trusted source (not user input)
{{ admin_generated_html|safe }}

# Document with comment:
{# safe: Content generated by admin, not user input #}
{{ admin_generated_html|safe }}
```

### Raw SQL in Tests
```python
# OK in tests for setup
def setUp(self):
    cursor.execute("TRUNCATE TABLE pages")
```

---

## Benefits

- **Catch vulnerabilities early** (before production)
- **Standardized security checks**
- **Automated + manual review**
- **Clear pass/fail criteria**
- **Actionable fix recommendations**
- **Compliance tracking** (OWASP Top 10)

---

## Security Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Django Security](https://docs.djangoproject.com/en/5.1/topics/security/)
- [Bandit Docs](https://bandit.readthedocs.io/)
- [Bleach Docs](https://bleach.readthedocs.io/)
