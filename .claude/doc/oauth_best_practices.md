# OAuth Best Practices - Django + django-allauth

## Overview

This document covers OAuth 2.0 best practices for Django projects, focusing on Google OAuth integration with django-allauth in multi-tenant systems.

---

## What You're Doing Right ✅

### 1. Email Uniqueness Validation (Bidirectional)
**Status:** ✅ Implemented correctly

You're preventing email conflicts between OAuth and Django auth:

```python
# apps/users/signals.py
@receiver(pre_social_login)
def check_social_email_uniqueness(sender, request, sociallogin, **kwargs):
    """Prevent OAuth signup if email is already used by Django auth."""
    email = sociallogin.account.extra_data.get("email")
    if email:
        existing_user = User.objects.get(email=email)
        if existing_user.auth_provider == "django":
            raise ImmediateHttpResponse(redirect(conflict_url))
```

**Why this is important:**
- Prevents duplicate accounts
- Maintains data integrity
- Provides clear error messages to users

---

### 2. Profile Auto-Creation
**Status:** ✅ Implemented correctly

You're automatically creating Profile when user authenticates via OAuth:

```python
@receiver(social_account_added)
def populate_profile_from_google(sender, request, sociallogin, **kwargs):
    """Create/update Profile with Google OAuth data."""
    profile, created = Profile.objects.get_or_create(user=user)
    # Update profile fields from Google data
```

**Why this is important:**
- Ensures every user has a Profile
- Captures OAuth data (picture, name, phone)
- No manual Profile creation needed

---

### 3. Auth Provider Tracking
**Status:** ✅ Implemented correctly

You're tracking authentication method in User model:

```python
auth_provider = models.CharField(
    max_length=50,
    default="django",
    choices=[("google", "Google OAuth"), ("django", "Django Auth")]
)
```

**Why this is important:**
- Allows different logic for OAuth vs Django users
- Helps with debugging and analytics
- Supports multiple OAuth providers in future

---

## What You Should Add 🔧

### 1. Account Linking (Missing)
**Status:** ⚠️ Recommended

Currently, if a user has a Django account and tries to login with Google using the same email, they get an error. You should allow **account linking** instead.

**Implementation:**

```python
# apps/users/signals.py
@receiver(pre_social_login)
def check_social_email_uniqueness(sender, request, sociallogin, **kwargs):
    """
    Check email uniqueness and offer account linking.

    If email exists with Django auth:
    - Option A: Redirect to login page with message to link accounts
    - Option B: Auto-link if user is authenticated (safest)
    """
    if sociallogin.is_existing:
        return

    email = sociallogin.account.extra_data.get("email")
    if not email:
        return

    try:
        existing_user = User.objects.get(email=email)

        # Option A: User is already authenticated - allow auto-linking
        if request.user.is_authenticated and request.user == existing_user:
            # User is logged in and trying to connect OAuth - allow it
            sociallogin.connect(request, existing_user)
            return

        # Option B: User not authenticated - require manual linking
        if existing_user.auth_provider == "django":
            # Store sociallogin in session for after login
            request.session['pending_social_connect'] = {
                'provider': sociallogin.account.provider,
                'email': email
            }

            # Redirect to login with message
            from django.contrib import messages
            messages.warning(
                request,
                f"An account with {email} already exists. "
                "Please login first to link your Google account."
            )
            raise ImmediateHttpResponse(redirect('user:login'))

        # User registered with OAuth - auto-connect
        else:
            sociallogin.connect(request, existing_user)

    except User.DoesNotExist:
        pass  # Allow signup
```

**Benefits:**
- Better user experience (no duplicate accounts)
- Users can use both login methods
- Follows OAuth 2.0 best practices

---

### 2. Token Refresh Strategy (Missing)
**Status:** ⚠️ Recommended for long sessions

OAuth tokens expire. If your users stay logged in for extended periods, you need a refresh strategy.

**Implementation:**

```python
# settings.py
SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'SCOPE': [
            'profile',
            'email',
        ],
        'AUTH_PARAMS': {
            'access_type': 'offline',  # Get refresh token
            'prompt': 'consent',       # Force consent screen
        },
        'FETCH_USERINFO': True,
    }
}
```

**Middleware to refresh tokens:**

```python
# apps/users/middleware.py
from allauth.socialaccount.models import SocialToken
from datetime import timedelta
from django.utils import timezone

class TokenRefreshMiddleware:
    """Refresh OAuth tokens before they expire."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.user.is_authenticated:
            # Check if user has Google OAuth token
            try:
                token = SocialToken.objects.get(
                    account__user=request.user,
                    account__provider='google'
                )

                # Refresh if expiring within 1 hour
                if token.expires_at:
                    expires_soon = token.expires_at - timedelta(hours=1)
                    if timezone.now() > expires_soon:
                        # Trigger token refresh
                        token.refresh()

            except SocialToken.DoesNotExist:
                pass

        return self.get_response(request)
```

**Benefits:**
- Prevents "token expired" errors
- Maintains uninterrupted user sessions
- Required for accessing Google APIs long-term

---

### 3. Scope Management (Review)
**Status:** ⚠️ Review current scopes

You're currently requesting:
```python
'SCOPE': ['profile', 'email']
```

**Recommended additional scopes if needed:**

```python
'SCOPE': [
    'profile',                    # Name, picture
    'email',                      # Email address
    # 'openid',                   # OIDC (if needed)
    # 'https://www.googleapis.com/auth/userinfo.phone',  # Phone number
]
```

**Note:** Google OAuth typically doesn't provide `phone_number` in basic scopes. You need to request specific permission:

```python
# settings.py
SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'SCOPE': [
            'profile',
            'email',
            'https://www.googleapis.com/auth/user.phonenumbers.read',  # For phone
        ],
    }
}
```

**Why this matters:**
- Minimizes permissions (privacy best practice)
- Only request what you actually use
- Some scopes require Google verification

---

### 4. Error Handling (Enhance)
**Status:** ⚠️ Add user-friendly error pages

Currently, OAuth errors may show generic error pages. Add custom error handling:

**Create error view:**

```python
# apps/users/views.py
from django.shortcuts import render

def oauth_error(request):
    """Handle OAuth errors with user-friendly message."""
    error = request.GET.get('error', 'unknown')
    error_description = request.GET.get('error_description', '')

    context = {
        'error': error,
        'error_description': error_description,
        'support_email': settings.DEFAULT_FROM_EMAIL
    }
    return render(request, 'users/oauth_error.html', context)
```

**Template:**

```html
<!-- templates/users/oauth_error.html -->
{% extends "web/base.html" %}

{% block content %}
<div class="container mx-auto px-4 py-12 max-w-md">
    <div class="alert alert-error">
        <svg xmlns="http://www.w3.org/2000/svg" class="stroke-current shrink-0 h-6 w-6" fill="none" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z" />
        </svg>
        <div>
            <h3 class="font-bold">OAuth Authentication Failed</h3>
            <p class="text-sm">{{ error_description }}</p>
        </div>
    </div>

    <div class="mt-6 text-center">
        <a href="{% url 'user:login' %}" class="btn btn-primary">Try Again</a>
        <a href="mailto:{{ support_email }}" class="btn btn-ghost">Contact Support</a>
    </div>
</div>
{% endblock %}
```

**Benefits:**
- Better user experience during failures
- Clear guidance on next steps
- Reduces support requests

---

### 5. Security Headers (Add)
**Status:** ⚠️ Protect OAuth redirect URIs

Add security headers to prevent OAuth hijacking:

```python
# settings.py (production)
if not DEBUG:
    # Prevent clickjacking on OAuth pages
    X_FRAME_OPTIONS = 'DENY'

    # Content Security Policy
    CSP_DEFAULT_SRC = ("'self'", 'accounts.google.com')
    CSP_SCRIPT_SRC = ("'self'", 'accounts.google.com')
    CSP_STYLE_SRC = ("'self'", "'unsafe-inline'", 'accounts.google.com')

    # HTTPS enforcement
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True

    # HSTS
    SECURE_HSTS_SECONDS = 31536000
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_HSTS_PRELOAD = True
```

**Benefits:**
- Prevents OAuth token interception
- Mitigates CSRF attacks
- Required for production

---

### 6. Logging and Monitoring (Add)
**Status:** ⚠️ Track OAuth events

Add comprehensive logging for OAuth events:

```python
# apps/users/signals.py
import logging

logger = logging.getLogger('oauth')  # Separate logger for OAuth

@receiver(social_account_added)
def populate_profile_from_google(sender, request, sociallogin, **kwargs):
    """Create/update Profile with Google OAuth data."""
    user = sociallogin.user

    # Log OAuth success
    logger.info(
        f"OAuth login successful - Provider: {sociallogin.account.provider}, "
        f"User: {user.email}, UID: {sociallogin.account.uid}"
    )

    # ... rest of implementation
```

**Log OAuth failures:**

```python
@receiver(pre_social_login)
def check_social_email_uniqueness(sender, request, sociallogin, **kwargs):
    """Check email uniqueness and log failures."""
    try:
        # ... email uniqueness check

        if existing_user.auth_provider == "django":
            logger.warning(
                f"OAuth login blocked - Email {email} already exists with Django auth"
            )
            # ... redirect logic

    except Exception as e:
        logger.error(f"OAuth pre_social_login error: {str(e)}", exc_info=True)
        raise
```

**Configure separate logger:**

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'oauth_file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': 'logs/oauth.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 5,
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'oauth': {
            'handlers': ['oauth_file'],
            'level': 'INFO',
            'propagate': False,
        },
    },
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
}
```

**Benefits:**
- Debug OAuth issues quickly
- Monitor for security incidents
- Track user authentication patterns

---

### 7. Rate Limiting (Add)
**Status:** ⚠️ Prevent OAuth abuse

Add rate limiting to OAuth endpoints:

```python
# apps/users/views.py (if you have custom OAuth views)
from django.core.cache import cache
from django.http import HttpResponseForbidden

def oauth_rate_limit(request):
    """Rate limit OAuth attempts per IP."""
    ip = request.META.get('REMOTE_ADDR')
    cache_key = f'oauth_attempts_{ip}'

    attempts = cache.get(cache_key, 0)

    if attempts >= 10:  # Max 10 attempts per hour
        return HttpResponseForbidden("Too many OAuth attempts. Try again later.")

    cache.set(cache_key, attempts + 1, 3600)  # 1 hour TTL
    return None
```

**Benefits:**
- Prevents brute force attacks
- Protects against OAuth enumeration
- Reduces server load

---

## Testing Checklist ✅

### Manual Testing
- [ ] New user can signup with Google OAuth
- [ ] Existing user can login with Google OAuth
- [ ] Profile is created/updated with Google data
- [ ] Email uniqueness is enforced (Django vs OAuth)
- [ ] Phone number is saved if provided by Google
- [ ] Avatar displays correctly in navbar
- [ ] User can logout and login again
- [ ] Error messages are user-friendly

### Security Testing
- [ ] OAuth redirect URI is HTTPS (production)
- [ ] CSRF tokens are present on OAuth pages
- [ ] State parameter is validated (django-allauth handles this)
- [ ] Tokens are stored securely (encrypted in database)
- [ ] Token expiration is handled gracefully

### Edge Cases
- [ ] User clicks "Cancel" on Google consent screen
- [ ] Google account has no profile picture (generic avatar shown)
- [ ] Google account has no phone number (field remains empty)
- [ ] Network error during OAuth flow (retry mechanism)
- [ ] Session expires during OAuth flow (redirect to login)

---

## Production Deployment Checklist 🚀

### Before Going Live
- [ ] OAuth credentials are in environment variables (not settings.py)
- [ ] HTTPS is enforced (SECURE_SSL_REDIRECT=True)
- [ ] Redirect URIs are whitelisted in Google Console
- [ ] Error logging is enabled for OAuth events
- [ ] Rate limiting is configured
- [ ] Security headers are enabled (CSP, HSTS, X-Frame-Options)
- [ ] Token refresh strategy is implemented (if long sessions)
- [ ] Account linking is enabled (if desired)

### Google OAuth Console Settings
1. **Authorized redirect URIs:**
   ```
   https://yourdomain.com/accounts/google/login/callback/
   https://yourdomain.com/accounts/google/login/
   ```

2. **OAuth consent screen:**
   - Application name: "Ribera Calma Resort Management"
   - Support email: your-email@domain.com
   - Scopes: email, profile (minimum)
   - Authorized domains: yourdomain.com

3. **Credentials:**
   - Client ID: Store in `.env` as `GOOGLE_CLIENT_ID`
   - Client Secret: Store in `.env` as `GOOGLE_CLIENT_SECRET`

---

## Common Pitfalls to Avoid ⚠️

### 1. Hardcoding OAuth Credentials
**Bad:**
```python
SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = 'abc123.apps.googleusercontent.com'
SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = 'secret123'
```

**Good:**
```python
SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = os.getenv('GOOGLE_CLIENT_ID')
SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = os.getenv('GOOGLE_CLIENT_SECRET')
```

### 2. Not Handling Missing Profile Data
**Bad:**
```python
profile.picture = google_data["picture"]  # KeyError if missing
```

**Good:**
```python
if "picture" in google_data:
    profile.picture = google_data["picture"]
```

### 3. Assuming Google Always Provides Phone
**Reality:** Google OAuth **rarely** provides `phone_number` unless you request specific scopes and the user grants permission.

**Solution:** Make phone optional and allow users to add it manually in profile settings.

### 4. Not Validating Redirect URIs
**Security Risk:** OAuth hijacking if redirect URIs are not strictly validated.

**Solution:** Django-allauth handles this automatically, but ensure ALLOWED_HOSTS is configured correctly:

```python
# settings.py (production)
ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com']
```

### 5. Forgetting to Update OAuth Credentials in Production
**Symptom:** OAuth works in development but fails in production.

**Solution:** Use separate Google OAuth credentials for dev and prod:
- Dev: http://localhost:8000/accounts/google/login/callback/
- Prod: https://yourdomain.com/accounts/google/login/callback/

---

## Future Enhancements 🚀

### 1. Multiple OAuth Providers
Add Facebook, GitHub, etc.:

```python
# settings.py
INSTALLED_APPS += [
    'allauth.socialaccount.providers.facebook',
    'allauth.socialaccount.providers.github',
]

SOCIALACCOUNT_PROVIDERS = {
    'google': {...},
    'facebook': {...},
    'github': {...},
}
```

### 2. Two-Factor Authentication (2FA)
Add 2FA for enhanced security:
- django-otp
- TOTP support
- Backup codes

### 3. Social Profile Sync
Periodically sync profile data from Google:
- Celery task to refresh profile pictures
- Update name if changed on Google
- Detect account deactivation

### 4. OAuth Audit Trail
Track all OAuth events:
- Login timestamps
- Failed attempts
- Account linking events
- Token refresh events

---

## Summary

Your current OAuth implementation is **solid** for MVP. The main areas to enhance before production:

1. **Account Linking** - Allow users to connect Google to existing accounts
2. **Token Refresh** - Handle long sessions gracefully
3. **Error Handling** - User-friendly error pages
4. **Security Headers** - Protect OAuth flow (HTTPS, CSP, HSTS)
5. **Logging** - Monitor OAuth events for debugging and security

Everything else is **optional** but recommended for a production-ready system.

---

## Resources

- [django-allauth Documentation](https://django-allauth.readthedocs.io/)
- [Google OAuth 2.0 Guide](https://developers.google.com/identity/protocols/oauth2)
- [OAuth 2.0 Security Best Practices](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [OWASP OAuth 2.0 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)
