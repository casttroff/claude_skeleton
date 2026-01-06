# JavaScript Module Patterns for Django Projects (2025)

---

## Executive Summary

**RECOMMENDATION: Hybrid Approach with ES6 Modules + Inline Scripts for Django Integration**

After extensive research of 2025 best practices, browser compatibility, Django ecosystem trends, and HTMX integration patterns, the recommended approach for Django + HTMX projects is:

**Use ES6 modules for reusable JavaScript logic + inline `<script type="module">` in templates for Django integration.**

**Why this approach:**
- ES6 modules are now the standard (88% browser support including all modern browsers)
- Import maps are production-ready across all major browsers (Safari 16.4+, Firefox 109+, Chrome 89+)
- Django template tags work in inline scripts, but NOT in external `.js` files
- HTMX 2.0+ has native ES6 module support
- No build step required for MVP (faster development)
- Clear migration path to bundler when needed (Vite/django-vite)

**Browser Support Cutoff:**
- Chrome 89+ (March 2021)
- Firefox 109+ (January 2023)
- Safari 16.4+ (March 2023)
- Edge 89+ (March 2021)

**Coverage: 88% of global browsers** (as of 2025)

---

## Browser Compatibility Analysis (2025 Data)

### ES6 Modules Native Support

| Feature | Chrome | Firefox | Safari | Edge | Coverage |
|---------|--------|---------|--------|------|----------|
| `<script type="module">` | 61+ | 60+ | 11+ | 79+ | 97% |
| Dynamic `import()` | 63+ | 67+ | 11.1+ | 79+ | 95% |
| Import Maps | 89+ | 109+ | 16.4+ | 89+ | 88% |
| Top-level `await` | 89+ | 89+ | 15+ | 89+ | 93% |

**Source:** caniuse.com (January 2025)

### Import Maps Browser Support

**Cross-browser support achieved:** March 2023

**Current support:**
- Chrome: 89-136 (fully supported)
- Edge: 89-133 (fully supported)
- Firefox: 109-138 (fully supported)
- Safari: 16.5-18.4 (fully supported)
- Safari iOS: 16.4-18.4 (fully supported)

**Polyfill available:** ES Module Shims supports ~94% of browsers with baseline ES module support.

**Production-ready:** YES (as of 2025)

**Feature detection:**
```javascript
if (HTMLScriptElement.supports && HTMLScriptElement.supports('importmap')) {
    console.log('Import maps supported');
}
```

### Key Findings

1. **ES6 modules are the present standard** (not the future)
2. **Import maps eliminate need for bundlers** for basic dependency management
3. **All major browsers support ES modules natively** since 2021
4. **No IE11 support needed** for modern Django projects (dropped in HTMX 2.0)

---

## Performance Comparison: IIFE vs ES6 Modules

### HTTP/2 Considerations

**ES6 Modules WIN with HTTP/2:**
- Modules are NOT meant to be concatenated (actively harmful to performance)
- HTTP/2 multiplexing benefits from separate module files
- Browser can cache individual modules (better cache invalidation)
- Parallel loading of dependencies

**IIFE with Concatenation:**
- Single large bundle = one cache invalidation = re-download everything
- No tree-shaking (dead code elimination)
- Harder to debug (minified code)

### Loading Performance

**ES6 Modules:**
- First load: Multiple HTTP requests (but parallel with HTTP/2)
- Subsequent loads: Cached modules (only changed modules re-downloaded)
- Dynamic imports enable code splitting (load on demand)

**IIFE:**
- First load: Single HTTP request (larger file)
- Subsequent loads: Re-download entire bundle if anything changes
- No code splitting (all-or-nothing)

### Caching Strategy

**ES6 Modules (Better):**
```
module-a.js (v1) → cached
module-b.js (v1) → cached
module-c.js (v2) → re-download only this
```

**IIFE Bundle (Worse):**
```
bundle.js (v1) → cached
bundle.js (v2) → re-download entire bundle (even if only one function changed)
```

### Verdict

**ES6 modules are faster** in production with HTTP/2 when:
- Code changes frequently
- Application grows beyond ~50KB JavaScript
- Multiple developers work on codebase
- Fine-grained cache control needed

**IIFE may be faster** for:
- Tiny apps (<10KB total JavaScript)
- HTTP/1.1 servers (not recommended in 2025)
- Legacy browser support (IE11)

---

## Django Ecosystem Survey (2025)

### What Modern Django Projects Use

**Surveyed Projects:**
1. **spookylukey/django-htmx-patterns** → Minimal vanilla JS (no modules)
2. **talkpython/htmx-django-course** → Inline scripts + vanilla JS
3. **adamchainz/django-htmx** → No JavaScript (pure HTMX)
4. **django-unicorn** → Single bundled file (~8KB gzipped)
5. **django-components** → Component-based (inline scripts)

**Key Trend:** Most Django + HTMX projects use **minimal JavaScript** with inline scripts.

**Why?**
- HTMX eliminates most JavaScript needs
- Alpine.js handles lightweight interactivity (14% adoption in Django ecosystem)
- Server-side rendering preferred over SPAs

### JavaScript Framework Usage in Django (2025 Survey)

| Framework | 2021 | 2025 | Growth |
|-----------|------|------|--------|
| HTMX | 5% | 24% | +380% |
| Alpine.js | 3% | 14% | +367% |
| React | 32% | 28% | -13% |
| Vue.js | 18% | 15% | -17% |

**Source:** JetBrains State of Django 2025

**Interpretation:** Django developers are moving AWAY from heavy JavaScript frameworks toward HTMX + minimal JS.

### Build Tools in Django Projects

**No build step (majority):**
- Faster development
- Simpler deployment
- No Node.js dependency
- Django collectstatic works out-of-box

**With build step (django-vite, webpack):**
- Large teams (5+ developers)
- Complex SPAs
- Tree-shaking critical
- TypeScript required

**Ribera Calma Project:** No build step for MVP (follows majority pattern).

---

## Django-Specific Considerations

### Problem 1: Django Template Tags in JavaScript Files

**THE CORE ISSUE:**
Django template tags ({% %}) are NOT processed in external `.js` files, only in `.html` templates.

**Example (DOES NOT WORK):**
```javascript
// static/js/gallery.js (external file)
const apiUrl = "{% url 'api:endpoint' %}";  // NOT PROCESSED BY DJANGO
```

**Solutions:**

#### Solution A: Pass Django Variables to Inline Script (RECOMMENDED)

```html
<!-- accommodation_form.html -->
<script type="module">
    import GalleryManager from '{% static "js/gallery-manager.js" %}';

    // Pass Django context to JavaScript
    const config = {
        apiUrl: "{% url 'accommodations:admin-add-image' site_slug=site.slug pk=accommodation.pk %}",
        csrfToken: "{{ csrf_token }}",
        siteSlug: "{{ site.slug }}",
        accommodationId: {{ accommodation.pk }},
    };

    const manager = new GalleryManager(config);
    manager.init('gallery-grid');
</script>
```

```javascript
// static/js/gallery-manager.js (ES6 module)
export default class GalleryManager {
    constructor(config) {
        this.apiUrl = config.apiUrl;
        this.csrfToken = config.csrfToken;
        this.siteSlug = config.siteSlug;
        this.accommodationId = config.accommodationId;
    }

    init(elementId) {
        // Use this.apiUrl, this.csrfToken, etc.
    }
}
```

**Pros:**
- Clean separation (template variables in template, logic in module)
- No Django coupling in JavaScript files
- Easy to test modules independently
- Works with all bundlers

**Cons:**
- Extra boilerplate in template

#### Solution B: Data Attributes (Alternative)

```html
<!-- accommodation_form.html -->
<div id="gallery-grid"
     data-api-url="{% url 'accommodations:admin-add-image' site_slug=site.slug pk=accommodation.pk %}"
     data-csrf-token="{{ csrf_token }}"
     data-site-slug="{{ site.slug }}"
     data-accommodation-id="{{ accommodation.pk }}">
</div>

<script type="module">
    import GalleryManager from '{% static "js/gallery-manager.js" %}';
    const manager = new GalleryManager();
    manager.init('gallery-grid');  // Reads data from element attributes
</script>
```

```javascript
// static/js/gallery-manager.js
export default class GalleryManager {
    init(elementId) {
        const element = document.getElementById(elementId);
        this.apiUrl = element.dataset.apiUrl;
        this.csrfToken = element.dataset.csrfToken;
        this.siteSlug = element.dataset.siteSlug;
        this.accommodationId = parseInt(element.dataset.accommodationId);
    }
}
```

**Pros:**
- Configuration lives in HTML (closer to data source)
- Less template code
- Follows web components pattern

**Cons:**
- Tight coupling between HTML structure and JavaScript
- Harder to debug (attributes scattered in DOM)

#### Solution C: JSON Script Tag (For Complex Configs)

```html
<!-- accommodation_form.html -->
<script id="gallery-config" type="application/json">
{
    "apiUrl": "{% url 'accommodations:admin-add-image' site_slug=site.slug pk=accommodation.pk %}",
    "csrfToken": "{{ csrf_token }}",
    "siteSlug": "{{ site.slug }}",
    "accommodationId": {{ accommodation.pk }},
    "maxImages": 10,
    "allowedTypes": ["image/jpeg", "image/png", "image/webp"]
}
</script>

<script type="module">
    import GalleryManager from '{% static "js/gallery-manager.js" %}';
    const config = JSON.parse(document.getElementById('gallery-config').textContent);
    const manager = new GalleryManager(config);
    manager.init('gallery-grid');
</script>
```

**Pros:**
- Clean JSON structure
- Easy to add complex configuration
- Type-safe (can validate schema)

**Cons:**
- Extra DOM element
- Slightly more verbose

### Problem 2: Django Static Files with ES6 Modules

**CRITICAL:** Django serves `.js` files with `Content-Type: text/plain` by default, breaking ES6 modules.

**Error:**
```
Failed to load module script: The server responded with a non-JavaScript MIME type of "text/plain".
Strict MIME type checking is enforced for module scripts per HTML spec.
```

**Solution (Add to settings.py):**

```python
# settings.py
import mimetypes

# CRITICAL for ES6 modules
mimetypes.add_type("application/javascript", ".js", True)
mimetypes.add_type("application/javascript", ".mjs", True)
```

**Why:** Django's staticfiles app uses Python's mimetypes library, which may not have correct JavaScript MIME types.

### Problem 3: CSRF Token in HTMX Requests

**HTMX + ES6 modules pattern:**

```javascript
// static/js/htmx-csrf.js
export function setupCSRFToken(token) {
    document.body.addEventListener('htmx:configRequest', (event) => {
        event.detail.headers['X-CSRFToken'] = token;
    });
}
```

```html
<!-- base.html -->
<script type="module">
    import { setupCSRFToken } from '{% static "js/htmx-csrf.js" %}';
    setupCSRFToken('{{ csrf_token }}');
</script>
```

**Alternative (global htmx configuration):**

```html
<!-- base.html -->
<meta name="csrf-token" content="{{ csrf_token }}">

<script type="module">
    // Set CSRF token globally for all HTMX requests
    const csrfToken = document.querySelector('meta[name="csrf-token"]').content;

    document.body.addEventListener('htmx:configRequest', (event) => {
        event.detail.headers['X-CSRFToken'] = csrfToken;
    });
</script>
```

---

## HTMX Integration Patterns

### HTMX 2.0 ES6 Module Support

**Good news:** HTMX 2.0+ has native ES6 module support.

**Loading HTMX as ES6 module:**

```html
<!-- base.html -->
<script type="importmap">
{
    "imports": {
        "htmx.org": "https://unpkg.com/[email protected]/dist/htmx.esm.js",
        "htmx.org/ext/debug": "https://unpkg.com/[email protected]/dist/ext/debug.js"
    }
}
</script>

<script type="module">
    import htmx from 'htmx.org';
    window.htmx = htmx;  // Make available globally for inline attributes
</script>
```

**HOWEVER:** For Django + HTMX projects, **loading HTMX via CDN in `<script>` tag is simpler** and recommended:

```html
<!-- base.html (RECOMMENDED for Django) -->
<script src="https://unpkg.com/[email protected]"></script>
```

**Why?**
- HTMX works best as a global object
- Inline `hx-*` attributes expect `htmx` on `window`
- No import needed in every module
- Simpler debugging

### HTMX Event Handling in ES6 Modules

**Pattern: Initialize after HTMX content swaps**

```javascript
// static/js/gallery-manager.js
export default class GalleryManager {
    constructor() {
        this.instances = new Map();
    }

    init(elementId) {
        const element = document.getElementById(elementId);
        if (!element) return;

        // Initialize Sortable.js
        const sortable = new Sortable(element, {
            animation: 150,
            onEnd: (evt) => this.handleReorder(evt)
        });

        this.instances.set(elementId, sortable);
    }

    destroy(elementId) {
        const sortable = this.instances.get(elementId);
        if (sortable) {
            sortable.destroy();
            this.instances.delete(elementId);
        }
    }

    handleReorder(evt) {
        // Handle drag-drop reordering
    }
}

// Re-initialize after HTMX swaps
export function setupHTMXListeners(manager) {
    document.body.addEventListener('htmx:afterSwap', (event) => {
        if (event.detail.target.id === 'gallery-grid') {
            manager.destroy('gallery-grid');  // Clean up old instance
            manager.init('gallery-grid');     // Re-initialize
        }
    });
}
```

```html
<!-- accommodation_form.html -->
<script type="module">
    import GalleryManager, { setupHTMXListeners } from '{% static "js/gallery-manager.js" %}';

    const manager = new GalleryManager();
    manager.init('gallery-grid');
    setupHTMXListeners(manager);  // Listen for HTMX swaps
</script>
```

**Critical:** Always clean up instances before re-initializing to prevent memory leaks.

### Common HTMX + ES6 Modules Pitfall

**Problem:** Event listeners accumulate after HTMX swaps.

**BAD (Listener accumulates):**
```javascript
document.body.addEventListener('htmx:afterSwap', (event) => {
    // This listener is added EVERY time this code runs
    // After 5 HTMX swaps, you have 5 listeners!
});
```

**GOOD (Use { once: true } or conditional):**
```javascript
// Option 1: Use { once: true } for one-time events
document.body.addEventListener('siteCreated', function() {
    document.getElementById('modal').close();
}, { once: true });

// Option 2: Use named function + check if already added
const afterSwapHandler = (event) => {
    // Handler logic
};

if (!window._htmxHandlerAdded) {
    document.body.addEventListener('htmx:afterSwap', afterSwapHandler);
    window._htmxHandlerAdded = true;
}

// Option 3: Use delegation (listen once, filter by target)
document.body.addEventListener('htmx:afterSwap', (event) => {
    if (event.detail.target.id === 'gallery-grid') {
        // Re-initialize only if target matches
    }
});
```

---

## Approach Analysis: IIFE vs ES6 Modules vs Build Tools

### Approach A: IIFE Pattern (Classic)

**Example:**
```javascript
// static/js/gallery-manager.js
window.GalleryManager = (function() {
    'use strict';

    const instances = new Map();

    function init(elementId) {
        const element = document.getElementById(elementId);
        // ... initialization logic
    }

    function destroy(elementId) {
        const sortable = instances.get(elementId);
        if (sortable) {
            sortable.destroy();
            instances.delete(elementId);
        }
    }

    return {
        init: init,
        destroy: destroy
    };
})();
```

```html
<script src="{% static 'js/gallery-manager.js' %}"></script>
<script>
    window.GalleryManager.init('gallery-grid');
</script>
```

**PROS:**
- Works in ALL browsers (even IE11)
- Simple mental model (global namespace)
- onclick handlers work (`onclick="GalleryManager.destroy('id')"`)
- No build step required
- No MIME type issues
- Familiar to all JavaScript developers

**CONS:**
- Global namespace pollution (risk of name collisions)
- No tree-shaking (all code loaded, even unused functions)
- No dependency management (must manually order `<script>` tags)
- Harder to test (globals are harder to mock)
- No async module loading (all-or-nothing)
- Outdated pattern (not the future of JavaScript)

**WHEN TO USE:**
- Legacy browser support required (IE11)
- Very small projects (<10KB JavaScript)
- Team unfamiliar with ES6 modules
- onclick handlers critical (forms with inline handlers)

**VERDICT FOR RIBERA CALMA:** ❌ Not recommended (no IE11 requirement, modern stack)

---

### Approach B: ES6 Modules (Modern, No Build)

**Example:**
```javascript
// static/js/gallery-manager.js
export default class GalleryManager {
    constructor() {
        this.instances = new Map();
    }

    init(elementId) {
        const element = document.getElementById(elementId);
        // ... initialization logic
    }

    destroy(elementId) {
        const sortable = this.instances.get(elementId);
        if (sortable) {
            sortable.destroy();
            this.instances.delete(elementId);
        }
    }
}
```

```html
<script type="module">
    import GalleryManager from '{% static "js/gallery-manager.js" %}';

    const config = {
        apiUrl: "{% url 'api:endpoint' %}",
        csrfToken: "{{ csrf_token }}"
    };

    const manager = new GalleryManager();
    manager.init('gallery-grid');

    // Expose to window if onclick handlers needed
    window.GalleryManager = manager;
</script>
```

**PROS:**
- Modern JavaScript standard (future-proof)
- Scoped variables (no global pollution)
- Explicit dependencies (`import`/`export`)
- Browser native (no build step)
- Tree-shaking ready (when you add bundler later)
- Better for testing (import in test files)
- Async module loading (`import()`)
- HTTP/2 optimization (cache individual modules)

**CONS:**
- Requires modern browsers (Chrome 61+, Firefox 60+, Safari 11+)
- MIME type configuration needed in Django (one-line fix)
- Django template tags don't work in `.js` files (use inline scripts)
- onclick handlers need `window.` assignment
- Slightly more verbose setup

**WHEN TO USE:**
- Modern browser support OK (2021+ browsers)
- Project expected to grow beyond MVP
- Multiple developers on team
- Code reuse across multiple pages
- No build step desired (fast iteration)

**VERDICT FOR RIBERA CALMA:** ✅ **RECOMMENDED** (matches project requirements)

---

### Approach C: ES6 Modules + Import Maps

**Example:**
```html
<!-- base.html -->
<script type="importmap">
{
    "imports": {
        "gallery-manager": "{% static 'js/gallery-manager.js' %}",
        "sortablejs": "https://cdn.jsdelivr.net/npm/sortablejs@1.15.0/+esm",
        "htmx-utils": "{% static 'js/htmx-utils.js' %}"
    }
}
</script>

<script type="module">
    import GalleryManager from 'gallery-manager';
    import Sortable from 'sortablejs';

    const manager = new GalleryManager();
    manager.init('gallery-grid');
</script>
```

**PROS:**
- All benefits of ES6 modules
- Clean imports (no long paths)
- CDN libraries work seamlessly
- Feels like Node.js (`import from 'package-name'`)
- No build step required
- Easy to swap implementations (change import map)

**CONS:**
- Requires Safari 16.4+ (March 2023) - newer than basic ES6 modules
- Browser support: 88% (vs 97% for basic modules)
- Import map must be in `<head>` or before first module
- Django template tags needed for static URLs in import map
- Slightly more complex setup

**WHEN TO USE:**
- Many CDN dependencies (lodash, dayjs, etc.)
- Want Node.js-like import experience
- Modern browser support OK (2023+ browsers)
- Team comfortable with import maps

**VERDICT FOR RIBERA CALMA:** ⚠️ Optional Enhancement (start with basic ES6, add import maps later if needed)

---

### Approach D: Build Tool (Vite + django-vite)

**Example:**
```javascript
// frontend/src/gallery-manager.js
import Sortable from 'sortablejs';  // NPM package
import { showToast } from './toast-utils';  // Local module

export default class GalleryManager {
    // ... implementation
}
```

```python
# settings.py
INSTALLED_APPS += ['django_vite']

DJANGO_VITE = {
    'default': {
        'dev_mode': DEBUG,
        'dev_server_host': 'localhost',
        'dev_server_port': 5173,
        'manifest_path': BASE_DIR / 'static' / 'dist' / 'manifest.json',
    }
}
```

```html
<!-- base.html -->
{% load django_vite %}
<!DOCTYPE html>
<html>
<head>
    {% vite_hmr_client %}
    {% vite_asset 'js/main.js' %}
</head>
```

**PROS:**
- Hot Module Replacement (HMR) in development (instant updates)
- Tree-shaking (dead code elimination)
- Code splitting (load modules on demand)
- NPM packages work seamlessly
- TypeScript support (if needed)
- Minification + optimization
- Source maps for debugging
- Best performance in production

**CONS:**
- Node.js dependency (requires npm/yarn)
- Build step required (slower first-time setup)
- More complex deployment
- Steeper learning curve
- Overkill for small projects
- Extra tooling to maintain

**WHEN TO USE:**
- Large projects (100+ KB JavaScript)
- TypeScript required
- Many NPM dependencies
- Team >5 developers
- Production performance critical
- Code splitting needed

**VERDICT FOR RIBERA CALMA:** ❌ Not recommended for MVP (overkill, adds complexity)

**Migration Path:** Start with ES6 modules (Approach B), migrate to Vite later if needed.

---

## Recommendation Matrix

| Scenario | IIFE | ES6 Modules | Import Maps | Build Tool | Recommendation |
|----------|------|-------------|-------------|------------|----------------|
| **MVP/Prototype** | ✅ | ✅ | ⚠️ | ❌ | **ES6 Modules** (simple, modern) |
| **Small Projects (<50KB JS)** | ✅ | ✅ | ✅ | ❌ | **ES6 Modules** (no overhead) |
| **Medium Projects (50-200KB JS)** | ⚠️ | ✅ | ✅ | ⚠️ | **ES6 Modules + Import Maps** |
| **Large Projects (>200KB JS)** | ❌ | ⚠️ | ⚠️ | ✅ | **Build Tool (Vite)** |
| **Teams >5 developers** | ❌ | ✅ | ✅ | ✅ | **Build Tool** (consistency) |
| **Legacy browser support (IE11)** | ✅ | ❌ | ❌ | ⚠️ | **IIFE** (no choice) |
| **No Node.js allowed** | ✅ | ✅ | ✅ | ❌ | **ES6 Modules** |
| **TypeScript required** | ❌ | ❌ | ❌ | ✅ | **Build Tool** (only option) |
| **Many NPM packages** | ❌ | ⚠️ | ✅ | ✅ | **Import Maps or Build Tool** |
| **Fast iteration (no builds)** | ✅ | ✅ | ✅ | ❌ | **ES6 Modules** |
| **Production perf critical** | ⚠️ | ✅ | ✅ | ✅ | **Build Tool** (tree-shaking) |
| **HTMX + Django** | ✅ | ✅ | ✅ | ⚠️ | **ES6 Modules** (HTMX is lightweight) |

**Legend:**
- ✅ Recommended
- ⚠️ Possible but not ideal
- ❌ Not recommended

---

## Ribera Calma Project Recommendation

### Current State Analysis

**Project:**
- Django 5.1+ backend
- HTMX 2.0+ for dynamic interactions
- DaisyUI 4.12+ for styling
- Sortable.js 1.15.0 (via CDN)
- Vanilla JavaScript only
- NO build step
- MVP phase (Week 1-4 of 30-day roadmap)

**JavaScript Needs:**
- Gallery drag-drop (Sortable.js initialization)
- HTMX event handling (afterSwap, custom events)
- Toast notifications
- Modal management
- Form validation helpers
- Analytics tracking

**Estimated Total JS:** ~20-30KB (small project)

### Recommended Architecture

**Use ES6 Modules + Inline Scripts**

**Structure:**
```
static/
├── js/
│   ├── gallery-manager.js       (ES6 module)
│   ├── toast-notifications.js   (ES6 module)
│   ├── modal-manager.js         (ES6 module)
│   ├── form-validator.js        (ES6 module)
│   └── htmx-utils.js            (ES6 module, HTMX helpers)
├── css/
│   └── custom.css
└── img/
```

**Template Pattern:**
```html
<!-- templates/accommodations/admin/form.html -->
{% extends "base.html" %}
{% load static %}

{% block extra_js %}
<script type="module">
    import GalleryManager from '{% static "js/gallery-manager.js" %}';
    import { showToast } from '{% static "js/toast-notifications.js" %}';

    // Pass Django context to JavaScript
    const config = {
        apiUrl: "{% url 'accommodations:admin-add-image' site_slug=site.slug pk=accommodation.pk %}",
        csrfToken: "{{ csrf_token }}",
        siteSlug: "{{ site.slug }}",
        accommodationId: {{ accommodation.pk }},
        maxImages: 10
    };

    // Initialize gallery
    const manager = new GalleryManager(config);
    manager.init('gallery-grid');

    // Listen for HTMX events
    document.body.addEventListener('htmx:afterSwap', (event) => {
        if (event.detail.target.id === 'gallery-grid') {
            manager.destroy('gallery-grid');
            manager.init('gallery-grid');
        }
    });

    // Listen for success events
    document.body.addEventListener('imageUploaded', () => {
        showToast('Image uploaded successfully', 'success');
    }, { once: true });
</script>
{% endblock %}
```

**Base Template:**
```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="es-ar" data-theme="{{ site.theme }}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="csrf-token" content="{{ csrf_token }}">
    <title>{% block title %}{{ site.name }}{% endblock %}</title>

    <!-- DaisyUI + Tailwind -->
    <link href="https://cdn.jsdelivr.net/npm/daisyui@4.12.10/dist/full.min.css" rel="stylesheet">
    <script src="https://cdn.tailwindcss.com"></script>

    <!-- HTMX (NOT as module, simpler as global) -->
    <script src="https://unpkg.com/[email protected]"></script>

    <!-- Sortable.js (NOT as module, simpler as global) -->
    <script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.0/Sortable.min.js"></script>
</head>
<body>
    {% block content %}{% endblock %}

    <!-- Global HTMX CSRF setup -->
    <script type="module">
        const csrfToken = document.querySelector('meta[name="csrf-token"]').content;

        document.body.addEventListener('htmx:configRequest', (event) => {
            event.detail.headers['X-CSRFToken'] = csrfToken;
        });
    </script>

    {% block extra_js %}{% endblock %}
</body>
</html>
```

**Why This Architecture:**
1. **ES6 modules for reusable logic** (gallery-manager, toast, etc.)
2. **Inline scripts for Django integration** (pass template variables)
3. **Global HTMX and Sortable** (simpler than importing in every module)
4. **No build step** (fast iteration during MVP)
5. **Clear migration path** to Vite later (ES6 modules are bundler-ready)

---

## Migration Guide: Current State → ES6 Modules

### Step 1: Configure Django MIME Types

**Add to settings.py:**
```python
# settings.py
import mimetypes

# CRITICAL for ES6 modules
mimetypes.add_type("application/javascript", ".js", True)
mimetypes.add_type("application/javascript", ".mjs", True)
```

### Step 2: Convert Existing Inline Scripts to Modules

**BEFORE (Current Ribera Calma):**
```html
<!-- accommodation_form.html -->
<script>
function initializeSortable() {
    const galleryGrid = document.getElementById('gallery-grid');
    if (!galleryGrid) return;

    new Sortable(galleryGrid, {
        animation: 150,
        onEnd: function(evt) {
            // Handle reorder
        }
    });
}

document.addEventListener('DOMContentLoaded', initializeSortable);

document.body.addEventListener('htmx:afterSwap', function(event) {
    if (event.detail.target.id === 'gallery-grid') {
        initializeSortable();
    }
});
</script>
```

**AFTER (ES6 Module):**

**File: static/js/gallery-manager.js**
```javascript
export default class GalleryManager {
    constructor(config = {}) {
        this.apiUrl = config.apiUrl || '';
        this.csrfToken = config.csrfToken || '';
        this.instances = new Map();
    }

    init(elementId) {
        const element = document.getElementById(elementId);
        if (!element) return;

        // Clean up existing instance
        this.destroy(elementId);

        // Create new Sortable instance
        const sortable = new Sortable(element, {
            animation: 150,
            handle: '.drag-handle',
            onEnd: (evt) => this.handleReorder(evt, elementId)
        });

        this.instances.set(elementId, sortable);
    }

    destroy(elementId) {
        const sortable = this.instances.get(elementId);
        if (sortable) {
            sortable.destroy();
            this.instances.delete(elementId);
        }
    }

    handleReorder(evt, elementId) {
        const element = document.getElementById(elementId);
        const items = Array.from(element.children);
        const order = items.map((item, index) => ({
            id: item.dataset.imageId,
            order: index
        }));

        // Send order to backend
        fetch(this.apiUrl, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRFToken': this.csrfToken
            },
            body: JSON.stringify({ order })
        })
        .then(response => response.json())
        .then(data => {
            if (data.success) {
                this.triggerEvent('orderUpdated', { order });
            }
        })
        .catch(error => {
            console.error('Reorder failed:', error);
        });
    }

    triggerEvent(eventName, detail = {}) {
        const event = new CustomEvent(eventName, { detail });
        document.body.dispatchEvent(event);
    }
}

// Helper: Setup HTMX listeners
export function setupHTMXListeners(manager, elementId) {
    document.body.addEventListener('htmx:afterSwap', (event) => {
        if (event.detail.target.id === elementId) {
            manager.destroy(elementId);
            manager.init(elementId);
        }
    });
}
```

**File: templates/accommodations/admin/form.html**
```html
{% extends "base.html" %}
{% load static %}

{% block extra_js %}
<script type="module">
    import GalleryManager, { setupHTMXListeners } from '{% static "js/gallery-manager.js" %}';
    import { showToast } from '{% static "js/toast-notifications.js" %}';

    const config = {
        apiUrl: "{% url 'accommodations:admin-reorder-images' site_slug=site.slug pk=accommodation.pk %}",
        csrfToken: "{{ csrf_token }}"
    };

    const manager = new GalleryManager(config);
    manager.init('gallery-grid');
    setupHTMXListeners(manager, 'gallery-grid');

    // Listen for order updates
    document.body.addEventListener('orderUpdated', () => {
        showToast('Gallery order updated', 'success');
    });
</script>
{% endblock %}
```

### Step 3: Create Reusable Modules

**File: static/js/toast-notifications.js**
```javascript
export function showToast(message, type = 'info', duration = 3000) {
    const toast = document.createElement('div');
    toast.className = `alert alert-${type} shadow-lg`;
    toast.innerHTML = `
        <div>
            <span>${message}</span>
        </div>
    `;

    const container = document.getElementById('toast-container') || createToastContainer();
    container.appendChild(toast);

    setTimeout(() => {
        toast.remove();
    }, duration);
}

function createToastContainer() {
    const container = document.createElement('div');
    container.id = 'toast-container';
    container.className = 'toast toast-top toast-end';
    document.body.appendChild(container);
    return container;
}

export function showError(message) {
    showToast(message, 'error', 5000);
}

export function showSuccess(message) {
    showToast(message, 'success', 3000);
}
```

**File: static/js/modal-manager.js**
```javascript
export default class ModalManager {
    constructor(modalId) {
        this.modalId = modalId;
        this.modal = document.getElementById(modalId);
    }

    open() {
        if (this.modal) {
            this.modal.showModal();
        }
    }

    close() {
        if (this.modal) {
            this.modal.close();
        }
    }

    onClose(callback) {
        if (this.modal) {
            this.modal.addEventListener('close', callback);
        }
    }
}

// Helper: Create modal from HTMX response
export function openModalFromHTMX(targetId) {
    document.body.addEventListener('htmx:afterSwap', (event) => {
        if (event.detail.target.id === targetId) {
            const modal = event.detail.target.querySelector('dialog');
            if (modal) {
                modal.showModal();
            }
        }
    }, { once: true });
}
```

**File: static/js/htmx-utils.js**
```javascript
// Global HTMX helpers

export function setupCSRF(csrfToken) {
    document.body.addEventListener('htmx:configRequest', (event) => {
        event.detail.headers['X-CSRFToken'] = csrfToken;
    });
}

export function setupLoadingIndicators() {
    document.body.addEventListener('htmx:beforeRequest', (event) => {
        const indicator = event.detail.elt.querySelector('.htmx-indicator');
        if (indicator) {
            indicator.classList.remove('htmx-indicator');
        }
    });

    document.body.addEventListener('htmx:afterRequest', (event) => {
        const indicator = event.detail.elt.querySelector('.loading');
        if (indicator) {
            indicator.classList.add('htmx-indicator');
        }
    });
}

export function setupErrorHandling() {
    document.body.addEventListener('htmx:responseError', (event) => {
        console.error('HTMX Error:', event.detail);

        const statusCode = event.detail.xhr.status;
        let message = 'Error en la solicitud';

        if (statusCode === 400) {
            message = 'Datos inválidos';
        } else if (statusCode === 403) {
            message = 'No tiene permisos';
        } else if (statusCode === 404) {
            message = 'Recurso no encontrado';
        } else if (statusCode === 500) {
            message = 'Error del servidor';
        }

        // Trigger custom event for error handling
        const errorEvent = new CustomEvent('htmxError', {
            detail: { message, statusCode }
        });
        document.body.dispatchEvent(errorEvent);
    });

    document.body.addEventListener('htmx:sendError', (event) => {
        console.error('Network Error:', event.detail);
        const errorEvent = new CustomEvent('htmxError', {
            detail: {
                message: 'Error de conexión. Verifique su internet.',
                statusCode: 0
            }
        });
        document.body.dispatchEvent(errorEvent);
    });
}
```

**Updated base.html:**
```html
<!-- templates/base.html -->
<script type="module">
    import { setupCSRF, setupLoadingIndicators, setupErrorHandling } from '{% static "js/htmx-utils.js" %}';
    import { showError } from '{% static "js/toast-notifications.js" %}';

    // Setup HTMX globally
    const csrfToken = document.querySelector('meta[name="csrf-token"]').content;
    setupCSRF(csrfToken);
    setupLoadingIndicators();
    setupErrorHandling();

    // Listen for HTMX errors and show toast
    document.body.addEventListener('htmxError', (event) => {
        showError(event.detail.message);
    });
</script>
```

### Step 4: Test Migration

**Checklist:**
- [ ] MIME type configured in settings.py
- [ ] All modules load without MIME errors
- [ ] Django template tags work in inline scripts
- [ ] HTMX requests include CSRF token
- [ ] Sortable.js initializes correctly
- [ ] HTMX swaps re-initialize modules
- [ ] Toast notifications work
- [ ] No console errors
- [ ] onclick handlers work (if any)

**Testing Script:**
```javascript
// Run in browser console
console.log('HTMX loaded:', typeof htmx !== 'undefined');
console.log('Sortable loaded:', typeof Sortable !== 'undefined');
console.log('CSRF token:', document.querySelector('meta[name="csrf-token"]')?.content);
console.log('Gallery instances:', window.galleryManager?.instances.size || 0);
```

---

## Handling onclick Handlers with ES6 Modules

### Problem

**ES6 modules create their own scope**, so functions are NOT in global namespace by default.

**This DOES NOT work:**
```html
<button onclick="deleteImage(123)">Delete</button>

<script type="module">
    import GalleryManager from './gallery-manager.js';

    function deleteImage(id) {
        // This function is NOT in global scope
        // onclick handler cannot access it
    }
</script>
```

### Solutions

#### Solution 1: Expose to window (Quick Fix)

```html
<button onclick="window.galleryManager.deleteImage(123)">Delete</button>

<script type="module">
    import GalleryManager from '{% static "js/gallery-manager.js" %}';

    const manager = new GalleryManager();
    window.galleryManager = manager;  // Expose globally
</script>
```

**Pros:** onclick works
**Cons:** Pollutes global namespace (defeats purpose of modules)

#### Solution 2: Use addEventListener (RECOMMENDED)

```html
<button data-image-id="123" class="delete-image-btn">Delete</button>

<script type="module">
    import GalleryManager from '{% static "js/gallery-manager.js" %}';

    const manager = new GalleryManager();

    // Event delegation (works even after HTMX swaps)
    document.body.addEventListener('click', (event) => {
        if (event.target.classList.contains('delete-image-btn')) {
            const imageId = event.target.dataset.imageId;
            manager.deleteImage(imageId);
        }
    });
</script>
```

**Pros:**
- No global pollution
- Works with dynamically added elements (HTMX swaps)
- Event delegation (single listener for all buttons)
- Easier to test

**Cons:**
- Slightly more code

#### Solution 3: Module exports deleteImage helper

```javascript
// static/js/gallery-manager.js
export default class GalleryManager {
    deleteImage(id) {
        // Implementation
    }
}

// Helper function for onclick (if absolutely necessary)
export function deleteImage(id) {
    if (window._galleryManager) {
        window._galleryManager.deleteImage(id);
    }
}
```

```html
<script type="module">
    import GalleryManager, { deleteImage } from '{% static "js/gallery-manager.js" %}';

    const manager = new GalleryManager();
    window._galleryManager = manager;
    window.deleteImage = deleteImage;  // Expose helper globally
</script>

<button onclick="deleteImage(123)">Delete</button>
```

**Verdict:** Prefer **Solution 2 (addEventListener)** for modern Django + HTMX projects.

---

## Code Examples for Ribera Calma

### Example 1: Complete Gallery Manager Module

**File: static/js/gallery-manager.js**
```javascript
/**
 * Gallery Manager - Handles drag-drop image reordering with Sortable.js
 * Usage:
 *   import GalleryManager from './gallery-manager.js';
 *   const manager = new GalleryManager(config);
 *   manager.init('gallery-grid');
 */

export default class GalleryManager {
    constructor(config = {}) {
        this.apiUrl = config.apiUrl || '';
        this.csrfToken = config.csrfToken || '';
        this.maxImages = config.maxImages || 10;
        this.instances = new Map();
    }

    /**
     * Initialize Sortable on a gallery element
     * @param {string} elementId - DOM element ID
     */
    init(elementId) {
        const element = document.getElementById(elementId);
        if (!element) {
            console.warn(`Gallery element not found: ${elementId}`);
            return;
        }

        // Clean up existing instance to prevent duplicates
        this.destroy(elementId);

        // Create Sortable instance
        const sortable = new Sortable(element, {
            animation: 150,
            handle: '.drag-handle',
            ghostClass: 'sortable-ghost',
            dragClass: 'sortable-drag',
            onEnd: (evt) => this.handleReorder(evt, elementId)
        });

        this.instances.set(elementId, sortable);
        console.log(`Gallery initialized: ${elementId}`);
    }

    /**
     * Destroy Sortable instance
     * @param {string} elementId - DOM element ID
     */
    destroy(elementId) {
        const sortable = this.instances.get(elementId);
        if (sortable) {
            sortable.destroy();
            this.instances.delete(elementId);
            console.log(`Gallery destroyed: ${elementId}`);
        }
    }

    /**
     * Handle drag-drop reorder
     * @param {Event} evt - Sortable.js event
     * @param {string} elementId - DOM element ID
     */
    handleReorder(evt, elementId) {
        const element = document.getElementById(elementId);
        const items = Array.from(element.children);

        // Build order array
        const order = items.map((item, index) => ({
            id: parseInt(item.dataset.imageId),
            order: index
        }));

        // Send to backend
        this.saveOrder(order);
    }

    /**
     * Save order to backend via HTMX or fetch
     * @param {Array} order - Array of {id, order} objects
     */
    async saveOrder(order) {
        try {
            const response = await fetch(this.apiUrl, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'X-CSRFToken': this.csrfToken
                },
                body: JSON.stringify({ order })
            });

            if (!response.ok) {
                throw new Error(`HTTP error ${response.status}`);
            }

            const data = await response.json();

            if (data.success) {
                this.triggerEvent('orderUpdated', { order });
            } else {
                this.triggerEvent('orderError', { message: data.error });
            }
        } catch (error) {
            console.error('Failed to save order:', error);
            this.triggerEvent('orderError', { message: error.message });
        }
    }

    /**
     * Delete image via HTMX trigger
     * @param {number} imageId - Image ID
     */
    deleteImage(imageId) {
        const confirmed = confirm('Are you sure you want to delete this image?');
        if (!confirmed) return;

        this.triggerEvent('imageDeleted', { imageId });
    }

    /**
     * Trigger custom event on document.body
     * @param {string} eventName - Event name
     * @param {Object} detail - Event detail data
     */
    triggerEvent(eventName, detail = {}) {
        const event = new CustomEvent(eventName, { detail });
        document.body.dispatchEvent(event);
    }
}

/**
 * Setup HTMX listeners to re-initialize gallery after swaps
 * @param {GalleryManager} manager - Gallery manager instance
 * @param {string} targetId - HTMX swap target ID
 */
export function setupHTMXListeners(manager, targetId) {
    document.body.addEventListener('htmx:afterSwap', (event) => {
        if (event.detail.target.id === targetId) {
            console.log('HTMX swap detected, re-initializing gallery');
            manager.destroy(targetId);
            manager.init(targetId);
        }
    });
}
```

### Example 2: Toast Notifications Module

**File: static/js/toast-notifications.js**
```javascript
/**
 * Toast Notifications - DaisyUI-compatible toast system
 * Usage:
 *   import { showToast, showError, showSuccess } from './toast-notifications.js';
 *   showToast('Message', 'info');
 */

let toastContainer = null;

/**
 * Create toast container if it doesn't exist
 */
function getToastContainer() {
    if (!toastContainer) {
        toastContainer = document.createElement('div');
        toastContainer.id = 'toast-container';
        toastContainer.className = 'toast toast-top toast-end z-50';
        document.body.appendChild(toastContainer);
    }
    return toastContainer;
}

/**
 * Show toast notification
 * @param {string} message - Message to display
 * @param {string} type - Toast type (info, success, warning, error)
 * @param {number} duration - Duration in milliseconds
 */
export function showToast(message, type = 'info', duration = 3000) {
    const container = getToastContainer();

    const toast = document.createElement('div');
    toast.className = `alert alert-${type} shadow-lg`;
    toast.innerHTML = `
        <div>
            <span>${message}</span>
        </div>
    `;

    container.appendChild(toast);

    // Auto-remove after duration
    setTimeout(() => {
        toast.classList.add('opacity-0', 'transition-opacity');
        setTimeout(() => toast.remove(), 300);
    }, duration);
}

/**
 * Show error toast (5 second duration)
 * @param {string} message - Error message
 */
export function showError(message) {
    showToast(message, 'error', 5000);
}

/**
 * Show success toast (3 second duration)
 * @param {string} message - Success message
 */
export function showSuccess(message) {
    showToast(message, 'success', 3000);
}

/**
 * Show warning toast (4 second duration)
 * @param {string} message - Warning message
 */
export function showWarning(message) {
    showToast(message, 'warning', 4000);
}
```

### Example 3: Form Validation Module

**File: static/js/form-validator.js**
```javascript
/**
 * Form Validation - Client-side validation helpers
 * NOTE: Always replicate validation on backend
 * Usage:
 *   import FormValidator from './form-validator.js';
 *   const validator = new FormValidator('my-form');
 *   validator.validate();
 */

export default class FormValidator {
    constructor(formId) {
        this.form = document.getElementById(formId);
        this.errors = new Map();
    }

    /**
     * Validate entire form
     * @returns {boolean} - True if valid
     */
    validate() {
        if (!this.form) return false;

        this.clearErrors();

        const inputs = this.form.querySelectorAll('input, textarea, select');
        let isValid = true;

        inputs.forEach(input => {
            if (!this.validateField(input)) {
                isValid = false;
            }
        });

        return isValid;
    }

    /**
     * Validate single field
     * @param {HTMLElement} field - Form field
     * @returns {boolean} - True if valid
     */
    validateField(field) {
        const value = field.value.trim();
        const name = field.name;

        // Required validation
        if (field.hasAttribute('required') && !value) {
            this.addError(name, 'This field is required');
            return false;
        }

        // Email validation
        if (field.type === 'email' && value) {
            const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
            if (!emailRegex.test(value)) {
                this.addError(name, 'Invalid email address');
                return false;
            }
        }

        // Number validation
        if (field.type === 'number' && value) {
            const num = parseFloat(value);
            const min = field.hasAttribute('min') ? parseFloat(field.min) : null;
            const max = field.hasAttribute('max') ? parseFloat(field.max) : null;

            if (min !== null && num < min) {
                this.addError(name, `Minimum value is ${min}`);
                return false;
            }

            if (max !== null && num > max) {
                this.addError(name, `Maximum value is ${max}`);
                return false;
            }
        }

        // Custom validation patterns
        if (field.hasAttribute('pattern') && value) {
            const pattern = new RegExp(field.pattern);
            if (!pattern.test(value)) {
                this.addError(name, field.title || 'Invalid format');
                return false;
            }
        }

        return true;
    }

    /**
     * Add validation error
     * @param {string} fieldName - Field name
     * @param {string} message - Error message
     */
    addError(fieldName, message) {
        this.errors.set(fieldName, message);

        const field = this.form.querySelector(`[name="${fieldName}"]`);
        if (field) {
            field.classList.add('input-error');

            // Add error message
            const errorDiv = document.createElement('div');
            errorDiv.className = 'text-error text-sm mt-1';
            errorDiv.textContent = message;
            field.parentNode.appendChild(errorDiv);
        }
    }

    /**
     * Clear all validation errors
     */
    clearErrors() {
        this.errors.clear();

        if (this.form) {
            // Remove error classes
            this.form.querySelectorAll('.input-error').forEach(el => {
                el.classList.remove('input-error');
            });

            // Remove error messages
            this.form.querySelectorAll('.text-error').forEach(el => {
                el.remove();
            });
        }
    }

    /**
     * Setup real-time validation
     */
    setupRealtimeValidation() {
        if (!this.form) return;

        const inputs = this.form.querySelectorAll('input, textarea, select');
        inputs.forEach(input => {
            input.addEventListener('blur', () => {
                this.validateField(input);
            });

            input.addEventListener('input', () => {
                // Clear error on input
                const name = input.name;
                if (this.errors.has(name)) {
                    this.errors.delete(name);
                    input.classList.remove('input-error');

                    const errorMsg = input.parentNode.querySelector('.text-error');
                    if (errorMsg) errorMsg.remove();
                }
            });
        });
    }
}

/**
 * Validate date range (check-in < check-out)
 * @param {Date} checkIn - Check-in date
 * @param {Date} checkOut - Check-out date
 * @returns {boolean} - True if valid
 */
export function validateDateRange(checkIn, checkOut) {
    return checkIn < checkOut;
}

/**
 * Sanitize input (basic XSS prevention)
 * NOTE: Always sanitize on backend with bleach
 * @param {string} input - User input
 * @returns {string} - Sanitized input
 */
export function sanitizeInput(input) {
    const div = document.createElement('div');
    div.textContent = input;
    return div.innerHTML;
}
```

### Example 4: Template Usage

**File: templates/accommodations/admin/form.html**
```html
{% extends "base.html" %}
{% load static %}

{% block content %}
<div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">
        {% if accommodation.pk %}Edit{% else %}Create{% endif %} Accommodation
    </h1>

    <form id="accommodation-form" method="post" enctype="multipart/form-data">
        {% csrf_token %}

        <!-- Form fields here -->
        {{ form.as_p }}

        <button type="submit" class="btn btn-primary">Save</button>
    </form>

    {% if accommodation.pk %}
    <div class="mt-8">
        <h2 class="text-xl font-bold mb-4">Gallery</h2>

        <!-- Gallery Grid (HTMX target) -->
        <div id="gallery-grid" class="grid grid-cols-3 gap-4">
            {% for image in accommodation.images.all %}
            <div class="card bg-base-100 shadow-xl"
                 data-image-id="{{ image.pk }}">
                <figure>
                    <img src="{{ image.image.url }}" alt="{{ image.alt_text }}">
                </figure>
                <div class="card-body p-2">
                    <div class="flex justify-between items-center">
                        <span class="drag-handle cursor-move">☰</span>
                        <button type="button"
                                class="btn btn-sm btn-error delete-image-btn"
                                data-image-id="{{ image.pk }}">
                            Delete
                        </button>
                    </div>
                </div>
            </div>
            {% endfor %}
        </div>

        <!-- Upload Button -->
        <button type="button"
                class="btn btn-primary mt-4"
                onclick="document.getElementById('upload-modal').showModal()">
            Add Image
        </button>
    </div>
    {% endif %}
</div>

<!-- Upload Modal -->
<dialog id="upload-modal" class="modal">
    <div class="modal-box">
        <h3 class="font-bold text-lg">Upload Image</h3>

        <form hx-post="{% url 'accommodations:admin-add-image' site_slug=site.slug pk=accommodation.pk %}"
              hx-encoding="multipart/form-data"
              hx-target="#gallery-grid"
              hx-swap="beforeend">
            {% csrf_token %}
            <input type="file" name="image" accept="image/*" class="file-input w-full mt-4">
            <input type="text" name="alt_text" placeholder="Alt text" class="input input-bordered w-full mt-2">

            <div class="modal-action">
                <button type="button" class="btn" onclick="document.getElementById('upload-modal').close()">
                    Cancel
                </button>
                <button type="submit" class="btn btn-primary">Upload</button>
            </div>
        </form>
    </div>
</dialog>
{% endblock %}

{% block extra_js %}
<script type="module">
    import GalleryManager, { setupHTMXListeners } from '{% static "js/gallery-manager.js" %}';
    import FormValidator from '{% static "js/form-validator.js" %}';
    import { showToast, showError, showSuccess } from '{% static "js/toast-notifications.js" %}';

    // Configuration from Django
    const config = {
        apiUrl: "{% url 'accommodations:admin-reorder-images' site_slug=site.slug pk=accommodation.pk %}",
        csrfToken: "{{ csrf_token }}",
        siteSlug: "{{ site.slug }}",
        accommodationId: {{ accommodation.pk|default:"null" }},
        maxImages: 10
    };

    // Initialize gallery if accommodation exists
    {% if accommodation.pk %}
    const galleryManager = new GalleryManager(config);
    galleryManager.init('gallery-grid');
    setupHTMXListeners(galleryManager, 'gallery-grid');

    // Listen for order updates
    document.body.addEventListener('orderUpdated', () => {
        showSuccess('Gallery order saved');
    });

    document.body.addEventListener('orderError', (event) => {
        showError('Failed to save order: ' + event.detail.message);
    });

    // Handle image deletion
    document.body.addEventListener('click', (event) => {
        if (event.target.classList.contains('delete-image-btn')) {
            const imageId = event.target.dataset.imageId;
            galleryManager.deleteImage(imageId);
        }
    });

    // Close modal after successful upload
    document.body.addEventListener('htmx:afterSwap', (event) => {
        if (event.detail.target.id === 'gallery-grid') {
            document.getElementById('upload-modal').close();
            showSuccess('Image uploaded successfully');
            galleryManager.destroy('gallery-grid');
            galleryManager.init('gallery-grid');
        }
    });
    {% endif %}

    // Form validation
    const formValidator = new FormValidator('accommodation-form');
    formValidator.setupRealtimeValidation();

    document.getElementById('accommodation-form').addEventListener('submit', (event) => {
        if (!formValidator.validate()) {
            event.preventDefault();
            showError('Please fix form errors');
        }
    });
</script>
{% endblock %}
```

---

## Cons of Each Approach (Honest Assessment)

### IIFE Pattern - Cons

1. **Global namespace pollution** - Risk of name collisions as app grows
2. **No dependency management** - Must manually order `<script>` tags
3. **No tree-shaking** - All code loaded, even if unused
4. **No async loading** - Can't lazy-load modules on demand
5. **Testing harder** - Globals are harder to mock and isolate
6. **Not the future** - Industry moving away from this pattern
7. **No code splitting** - Can't split code by route/feature
8. **Harder to refactor** - Globals make code coupling harder to track

### ES6 Modules (No Build) - Cons

1. **Browser support** - Requires 2021+ browsers (Chrome 61+, Firefox 60+, Safari 11+)
2. **Django template tags don't work in .js files** - Must use inline scripts to pass Django variables
3. **onclick handlers need window exposure** - Defeats module scope encapsulation
4. **MIME type configuration** - Requires one-line Django settings change
5. **No tree-shaking without bundler** - Unused exports still downloaded
6. **Multiple HTTP requests** - More requests than bundled IIFE (mitigated by HTTP/2)
7. **Debugging can be harder** - Browser dev tools vary in module support
8. **CORS issues with CDN modules** - Some CDNs don't set correct CORS headers

### Import Maps - Cons

1. **Browser support newer** - Requires 2023+ browsers (Safari 16.4+, Firefox 109+)
2. **88% browser coverage** - 12% of users on older browsers can't use it
3. **Must be in `<head>` or before modules** - Template structure matters
4. **Django template tags needed in import map** - Couples HTML to static file paths
5. **Limited tooling** - IDE autocomplete doesn't understand import maps well
6. **No fallback for old browsers** - Requires polyfill (ES Module Shims)
7. **Debugging confusing** - Bare specifiers make stack traces harder to read
8. **Not all CDNs support ESM** - Some libraries not available as ES modules

### Build Tool (Vite) - Cons

1. **Node.js dependency** - Requires npm/yarn installed on dev machine and CI/CD
2. **Build step required** - Slower development loop (save → build → refresh)
3. **More complex deployment** - Must run `npm run build` before collectstatic
4. **Learning curve** - Team needs to learn Vite configuration
5. **Extra tooling to maintain** - package.json, vite.config.js, Node version
6. **Overkill for small projects** - 20KB of JS doesn't need Vite
7. **HTMX doesn't benefit much** - HTMX eliminates most JS, so bundling less useful
8. **django-vite has quirks** - Some Django features (like dev server auto-reload) conflict with Vite HMR

---

## Final Recommendation for Ribera Calma

### Use ES6 Modules + Inline Scripts

**Why:**
1. **Modern standard** - ES6 modules are the present and future of JavaScript
2. **No build step** - Fast iteration during MVP (matches project phase)
3. **Browser support OK** - 97% coverage for basic modules (Chrome 61+, Firefox 60+, Safari 11+)
4. **HTMX compatible** - HTMX 2.0 has native ES6 module support
5. **Clear migration path** - Can add Vite later if project grows
6. **Small JavaScript footprint** - ~20-30KB total JS doesn't need bundler
7. **Clean separation** - Django variables in templates, logic in modules
8. **Reusable code** - Modules can be imported across templates

**Architecture:**
```
static/js/
├── gallery-manager.js        (ES6 module)
├── toast-notifications.js    (ES6 module)
├── modal-manager.js          (ES6 module)
├── form-validator.js         (ES6 module)
└── htmx-utils.js             (ES6 module)
```

**Template Pattern:**
```html
<script type="module">
    import GalleryManager from '{% static "js/gallery-manager.js" %}';

    const config = {
        apiUrl: "{% url 'api:endpoint' %}",
        csrfToken: "{{ csrf_token }}"
    };

    const manager = new GalleryManager(config);
    manager.init('gallery-grid');
</script>
```

**When to migrate to Vite:**
- JavaScript grows beyond 100KB
- Team adds 3+ developers
- TypeScript becomes necessary
- Code splitting needed (route-based lazy loading)
- NPM dependencies exceed 5 packages

---

## Conclusion

**ES6 modules are production-ready in 2025** and recommended for modern Django + HTMX projects like Ribera Calma.

**Key Takeaways:**
1. ES6 modules are the standard (not IIFE)
2. Import maps are production-ready (88% browser support)
3. Django template tags work in inline scripts, NOT external .js files
4. HTMX 2.0 has native ES6 module support
5. No build step needed for MVP (can add Vite later)
6. Use addEventListener instead of onclick for module functions
7. Always sanitize user input on backend (client-side validation is UX only)

**Next Steps:**
1. Add `mimetypes.add_type("application/javascript", ".js", True)` to settings.py
2. Create `static/js/` directory with ES6 modules
3. Convert inline scripts to modules incrementally
4. Test across browsers (Chrome, Firefox, Safari)
5. Monitor bundle size (add Vite if JS exceeds 100KB)

**Further Reading:**
- MDN: JavaScript Modules - https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules
- Import Maps Spec - https://github.com/WICG/import-maps
- HTMX + ES6 Modules - https://htmx.org/essays/no-build-step/
- django-vite Documentation - https://github.com/MrBin99/django-vite

---

**Document Version:** 1.0
**Last Updated:** 2025-11-08
**Author:** Research compiled from web.dev, MDN, Django community, HTMX documentation
