# OAuth Cookie Domain Configuration for Multi-tenant Applications

**Date Created:** 2025-11-08
**Status:** Production-Ready Solution
**Severity:** CRITICAL

---

## Problem Statement

Multi-tenant Django applications using subdomain-based access (e.g., `riberacalma.localhost`, `alpendorf.localhost`) require session cookies to be shared across all subdomains to maintain user authentication. OAuth Google login was returning `AnonymousUser` after successful authentication when navigating between the main domain and subdomains.

---

## Root Cause Analysis

### Issue Discovery Timeline

1. **Initial symptom**: OAuth Google login completing successfully but user showing as `AnonymousUser` on subdomain
2. **Signal verification**: `pre_social_login` and `social_account_updated` signals firing correctly
3. **Database verification**: User created in database with correct email
4. **Django configuration check**: `SESSION_COOKIE_DOMAIN = '.localhost'` configured correctly
5. **Browser inspection**: Cookie domain showing `"localhost"` (without dot) and `HostOnly: true`
6. **Critical finding**: Modern browsers reject cookies with domain `.localhost`

### Technical Root Cause

Modern browsers (Chrome, Firefox, Safari, Edge) have special handling for the `localhost` top-level domain (TLD):

- **Security policy**: Browsers reject wildcard cookies on special TLDs like `localhost`
- **Result**: Django sets `SESSION_COOKIE_DOMAIN = '.localhost'` but browsers store it as `Domain: "localhost"` (without dot)
- **Effect**: Cookie becomes host-only and is NOT shared with subdomains
- **Impact**: User authenticated on `localhost:8011` appears as `AnonymousUser` on `riberacalma.localhost:8011`

---

## Solution: Use lvh.me Domain

### What is lvh.me?

- **Public DNS domain**: `lvh.me` and `*.lvh.me` resolve to 127.0.0.1 (localhost)
- **Browser compatibility**: Treated as regular domain (not special TLD)
- **Wildcard cookie support**: Browsers accept `Domain: .lvh.me` cookies
- **Industry standard**: Used by Stripe, Shopify, and other multi-tenant SaaS platforms
- **Zero configuration**: No hosts file changes required
- **Maintained by**: Community-maintained public service (similar to nip.io, xip.io)

### Implementation

#### 1. Django Settings (settings.py)

```python
import os

# Multi-tenant Configuration
MULTI_TENANT_SHARED_DOMAIN = os.environ.get('MULTI_TENANT_SHARED_DOMAIN', 'lvh.me')
MULTI_TENANT_DEV_HOSTS = ['localhost', '127.0.0.1', 'testserver', 'lvh.me']

# Cookie domain configuration for multi-tenant subdomain support
# CRITICAL: The leading dot (.) makes cookies available to all subdomains
# NOTE: Browsers reject .localhost cookies, so we use .lvh.me (points to 127.0.0.1)
if DEBUG:
    # Development: Use .lvh.me (browsers accept this, unlike .localhost)
    SESSION_COOKIE_DOMAIN = '.lvh.me'
    CSRF_COOKIE_DOMAIN = '.lvh.me'
else:
    # Production: Share cookies across your shared domain and all subdomains
    SESSION_COOKIE_DOMAIN = f'.{MULTI_TENANT_SHARED_DOMAIN}'
    CSRF_COOKIE_DOMAIN = f'.{MULTI_TENANT_SHARED_DOMAIN}'

# Cookie security settings
SESSION_COOKIE_SAMESITE = 'Lax'  # Allow cross-subdomain requests while maintaining CSRF protection
CSRF_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_HTTPONLY = True  # Prevent JavaScript access to session cookie
CSRF_COOKIE_HTTPONLY = False  # CSRF token needs to be readable by JavaScript for HTMX
SESSION_COOKIE_SECURE = False if DEBUG else True  # HTTPS only in production

# ALLOWED_HOSTS Configuration
if DEBUG:
    # Development: Allow localhost and lvh.me (with all subdomains)
    ALLOWED_HOSTS = ['localhost', '127.0.0.1', 'testserver', '.localhost', '.lvh.me', 'lvh.me']
else:
    # Production: Explicit allow list
    ALLOWED_HOSTS = [
        # Custom domains (add each site domain)
        'riberacalma.com',
        'www.riberacalma.com',

        # Wildcard for subdomains
        f'.{MULTI_TENANT_SHARED_DOMAIN}',  # Matches *.calmasites.com
    ]

# OAuth Google settings (django-allauth)
SOCIALACCOUNT_EMAIL_VERIFICATION = "none"  # Trust OAuth providers (Google verifies emails)
SOCIALACCOUNT_AUTO_SIGNUP = True
ACCOUNT_EMAIL_VERIFICATION = 'optional'  # Don't block OAuth with email verification
```

#### 2. Middleware Support (apps/core/middleware.py)

```python
def _resolve_by_subdomain(self, host: str) -> Optional['Site']:
    """
    Resolve site by subdomain.

    Supports both development and production:
    - Development: ribera.localhost or ribera.lvh.me
    - Production: ribera.calmasites.com

    Args:
        host: Request host (e.g., 'ribera.lvh.me' or 'ribera.localhost')

    Returns:
        Site instance or None
    """
    # Extract subdomain
    if host.endswith('.localhost'):
        subdomain = host.replace('.localhost', '')
    elif host.endswith('.lvh.me'):
        subdomain = host.replace('.lvh.me', '')
    elif self.SHARED_DOMAIN and host.endswith(f'.{self.SHARED_DOMAIN}'):
        subdomain = host.replace(f'.{self.SHARED_DOMAIN}', '')
    else:
        return None

    # Skip 'www' subdomain (typically reserved for main site/landing page)
    if subdomain == 'www':
        return None

    cache_key = f"site:subdomain:{subdomain}"
    site = cache.get(cache_key)

    if site is not None:
        return site if site != 'NOT_FOUND' else None

    from apps.core.models import Site

    try:
        site = Site.objects.select_related("owner").get(subdomain=subdomain, is_active=True)
        cache.set(cache_key, site, self.CACHE_TIMEOUT)
        logger.debug(f"Site resolved by subdomain: {subdomain} → {site.name}")
        return site

    except Site.DoesNotExist:
        cache.set(cache_key, 'NOT_FOUND', 60)  # Cache negative result briefly
        return None
```

#### 3. OAuth Google Configuration

Update Google OAuth Console (https://console.cloud.google.com/):

**Authorized JavaScript origins:**
```
http://lvh.me:8011
http://riberacalma.lvh.me:8011
```

**Authorized redirect URIs:**
```
http://lvh.me:8011/accounts/google/login/callback/
http://riberacalma.lvh.me:8011/accounts/google/login/callback/
```

**Note**: Add all subdomains you plan to test with during development.

---

## Testing & Validation

### 1. Clear Old Sessions

When switching from `.localhost` to `.lvh.me`, old sessions with incorrect domain must be deleted:

```python
# Django shell
from django.contrib.sessions.models import Session
Session.objects.all().delete()
```

**Or via management command:**
```bash
docker compose exec web python manage.py clearsessions
```

### 2. Verify Django Configuration

Run diagnostic script:
```bash
docker compose exec web python debug_oauth.py
```

Expected output:
```
🍪 COOKIE CONFIGURATION:
  SESSION_COOKIE_DOMAIN: '.lvh.me'
  CSRF_COOKIE_DOMAIN: '.lvh.me'
  SESSION_COOKIE_SAMESITE: 'Lax'
  SESSION_COOKIE_SECURE: False
```

### 3. Browser Cookie Inspection

**Steps:**
1. Navigate to `http://lvh.me:8011/users/login/`
2. Login with OAuth Google
3. Open DevTools → Application → Cookies → `http://lvh.me:8011`
4. Inspect `sessionid` cookie

**Expected values:**
```
Name: sessionid
Value: <long hash>
Domain: .lvh.me          ✅ Leading dot
Path: /
Expires: <2 weeks from now>
HttpOnly: true           ✅ Secure
Secure: false            ✅ OK for development (HTTP)
SameSite: Lax           ✅ Cross-subdomain allowed
HostOnly: false          ✅ Shared with subdomains
```

### 4. Test Cookie Sharing

**Steps:**
1. Login at `http://lvh.me:8011/users/login/` with OAuth Google
2. Verify you see your username in navbar
3. Navigate to `http://riberacalma.lvh.me:8011/`
4. Verify you remain authenticated (username still visible)

**Expected result:** User remains logged in across all subdomains

### 5. Test CSRF Protection

**Steps:**
1. Create a site at `http://lvh.me:8011/sites/`
2. Navigate to `http://riberacalma.lvh.me:8011/alojamiento/admin/`
3. Create an accommodation type

**Expected result:** Form submissions work without CSRF errors

---

## Troubleshooting Guide

### Issue 1: Still seeing AnonymousUser after login

**Diagnostic steps:**
1. Check browser cookies:
   ```
   DevTools → Application → Cookies
   Look for Domain: ".lvh.me" (with dot)
   ```

2. Verify Django settings:
   ```bash
   docker compose exec web python manage.py shell
   >>> from django.conf import settings
   >>> settings.SESSION_COOKIE_DOMAIN
   '.lvh.me'  # Should be this
   ```

3. Check middleware logs:
   ```bash
   docker compose logs web | grep "Site resolved"
   ```

4. Clear sessions and restart:
   ```bash
   docker compose exec web python manage.py shell
   >>> from django.contrib.sessions.models import Session
   >>> Session.objects.all().delete()
   >>> exit()
   docker compose restart web
   ```

### Issue 2: OAuth redirect fails

**Diagnostic steps:**
1. Check Google OAuth Console redirect URIs include `lvh.me`:
   ```
   http://lvh.me:8011/accounts/google/login/callback/
   http://*.lvh.me:8011/accounts/google/login/callback/  # If you added subdomains
   ```

2. Check `ALLOWED_HOSTS`:
   ```python
   ALLOWED_HOSTS = ['localhost', '127.0.0.1', '.lvh.me', 'lvh.me']
   ```

3. Check django-allauth Site configuration:
   ```bash
   docker compose exec web python manage.py shell
   >>> from django.contrib.sites.models import Site
   >>> Site.objects.get(id=1)
   <Site: lvh.me:8011>  # Should match your domain
   ```

### Issue 3: CSRF verification failed

**Diagnostic steps:**
1. Check CSRF cookie domain:
   ```python
   CSRF_COOKIE_DOMAIN = '.lvh.me'  # Must match SESSION_COOKIE_DOMAIN
   ```

2. Check HTMX requests include CSRF token:
   ```html
   <form hx-post="..." hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>
   ```

3. Check `render_to_string` includes request:
   ```python
   render_to_string('template.html', {'form': form}, request=self.request)
   ```

### Issue 4: Cookie not persisting across browser restarts

**Cause:** Cookie expiration too short

**Fix:**
```python
# settings.py
SESSION_COOKIE_AGE = 1209600  # 2 weeks in seconds
```

---

## Production Migration

### Development → Production Checklist

1. **Update DNS records:**
   ```
   A     @             <your-server-ip>
   A     *             <your-server-ip>  # Wildcard for subdomains
   ```

2. **Update settings.py:**
   ```python
   MULTI_TENANT_SHARED_DOMAIN = 'calmasites.com'  # Your production domain
   SESSION_COOKIE_DOMAIN = '.calmasites.com'
   CSRF_COOKIE_DOMAIN = '.calmasites.com'
   SESSION_COOKIE_SECURE = True  # HTTPS required
   CSRF_COOKIE_SECURE = True
   ```

3. **Update ALLOWED_HOSTS:**
   ```python
   ALLOWED_HOSTS = [
       'riberacalma.com',
       '.calmasites.com',
   ]
   ```

4. **SSL/TLS Certificate:**
   ```bash
   # Wildcard certificate for subdomains
   certbot certonly --dns-cloudflare -d *.calmasites.com -d calmasites.com
   ```

5. **Update OAuth redirect URIs:**
   ```
   https://calmasites.com/accounts/google/login/callback/
   https://ribera.calmasites.com/accounts/google/login/callback/
   ```

---

## Related Issues & Learnings

### Email Verification Blocking OAuth (2025-11-08)

**Issue:** `ACCOUNT_EMAIL_VERIFICATION = "mandatory"` was blocking OAuth login even though Google already verified the email.

**Fix:** Added `SOCIALACCOUNT_EMAIL_VERIFICATION = "none"` to trust OAuth providers.

**Lesson:** Social account email verification should be separate from regular account verification.

### ENVIRONMENT Variable Mismatch (2025-11-08)

**Issue:** `ENVIRONMENT=prod` in docker-compose.yml was enabling `SESSION_COOKIE_SECURE=True` which requires HTTPS, but development uses HTTP.

**Fix:** Changed to `ENVIRONMENT=dev` in docker-compose.yml for all services.

**Lesson:** Environment variables should match deployment context.

---

## Security Considerations

### Why lvh.me is Safe

1. **Public DNS verification:**
   ```bash
   dig lvh.me
   # Answer: 127.0.0.1
   ```

2. **Community maintained:** Similar to other development DNS services (nip.io, xip.io)

3. **No data leaves your machine:** All requests to `*.lvh.me` resolve to 127.0.0.1 locally

4. **Industry usage:** Used by major platforms (Stripe, Shopify, etc.)

5. **Browser security:** Browsers apply same-origin policy correctly

### Risks & Mitigations

**Risk 1: DNS hijacking**
- Mitigation: Verify DNS resolution periodically
- Fallback: Use localhost with slug-based URLs if lvh.me fails

**Risk 2: Third-party dependency**
- Mitigation: lvh.me is community-maintained, not commercial
- Fallback: Can use nip.io or xip.io as alternatives

**Risk 3: Production misconfiguration**
- Mitigation: Explicitly check `DEBUG` flag in settings
- Mitigation: Separate production settings file

---

## References

- **Django Sessions:** https://docs.djangoproject.com/en/5.1/topics/http/sessions/
- **Cookie Domain Spec:** https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
- **lvh.me Service:** https://lvh.me
- **django-allauth:** https://docs.allauth.org/
- **Browser Cookie Policies:** https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie

---

## Changelog

- **2025-11-08**: Initial documentation after solving OAuth AnonymousUser issue with lvh.me solution
- **2025-11-08**: Added comprehensive troubleshooting guide and production migration checklist
