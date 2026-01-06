# HTMX Inline Script Global Variables - Best Practices

## Research Date
2025-01-08

## Context
Managing global variables in Django template partials that include inline `<script>` tags, where templates can be loaded multiple times via HTMX partial rendering.

---

## TL;DR Recommendation

**Use Option 4 (External JS with Namespaced Module Pattern) for production code.**

For rapid prototyping or truly one-off inline scripts, use **Option 2 (window object with guard pattern)**.

---

## Detailed Analysis

### Option 1: `var` Declaration

```javascript
<script>
var sortableInstance = null;

function initializeSortable() {
    if (sortableInstance) {
        sortableInstance.destroy();
    }
    sortableInstance = new Sortable(...);
}
</script>
```

**Pros:**
- Simple syntax
- `var` allows redeclaration without errors
- Function-scoped (hoisted to top)

**Cons:**
- `var` is outdated (pre-ES6)
- Pollutes global scope
- No protection against accidental overwriting
- Confusing scoping rules (function vs block)
- Not best practice in modern JavaScript

**Verdict:** ❌ **Avoid** - Only acceptable for legacy code

---

### Option 2: `window` Object with Guard Pattern

```javascript
<script>
// Guard against multiple executions
window.sortableInstance = window.sortableInstance || null;

function initializeSortable() {
    if (window.sortableInstance) {
        window.sortableInstance.destroy();
    }

    const galleryGrid = document.getElementById('gallery-grid');
    if (!galleryGrid) return;

    window.sortableInstance = new Sortable(galleryGrid, {
        animation: 150,
        handle: '.drag-handle',
        onEnd: function(evt) {
            // Update order via HTMX
        }
    });
}

// Execute on load
document.addEventListener('DOMContentLoaded', initializeSortable);

// Re-execute after HTMX swaps
document.body.addEventListener('htmx:afterSwap', function(event) {
    if (event.detail.target.id === 'gallery-grid') {
        initializeSortable();
    }
});
</script>
```

**Pros:**
- Explicit global scope (clear intent)
- Guard pattern (`||`) prevents null overwrites
- Works with HTMX re-execution
- Simple to understand
- Good for rapid prototyping

**Cons:**
- Still pollutes global scope
- No encapsulation
- Hard to track dependencies
- Difficult to test in isolation

**Verdict:** ✅ **Acceptable for inline scripts** - Good for rapid development

---

### Option 3: IIFE (Immediately Invoked Function Expression)

```javascript
<script>
(function() {
    // Private scope
    let sortableInstance = null;

    function initializeSortable() {
        if (sortableInstance) {
            sortableInstance.destroy();
        }

        const galleryGrid = document.getElementById('gallery-grid');
        if (!galleryGrid) return;

        sortableInstance = new Sortable(galleryGrid, {
            animation: 150,
            handle: '.drag-handle',
            onEnd: function(evt) {
                updateOrder();
            }
        });
    }

    function updateOrder() {
        // HTMX request to update order
    }

    // Expose only what's needed globally
    window.GalleryManager = window.GalleryManager || {};
    window.GalleryManager.initializeSortable = initializeSortable;

    // Auto-init
    document.addEventListener('DOMContentLoaded', initializeSortable);
    document.body.addEventListener('htmx:afterSwap', function(event) {
        if (event.detail.target.id === 'gallery-grid') {
            initializeSortable();
        }
    });
})();
</script>
```

**Pros:**
- Private scope for internal variables
- Only exposes necessary functions
- Prevents variable pollution
- Better encapsulation than Option 2
- Can be used inline

**Cons:**
- More complex syntax
- Still creates global namespace (`window.GalleryManager`)
- Harder to read for beginners
- If script runs multiple times, IIFE re-executes (can create issues)

**Verdict:** ⚠️ **Good but overkill for inline** - Better moved to external file

---

### Option 4: External JS File with Module Pattern ⭐

**Template (image_gallery_inline.html):**
```html
<!-- Template only contains markup, no inline scripts -->
<div id="gallery-grid" class="grid grid-cols-3 gap-4">
    {% for image in accommodation.images.all %}
        {% include 'accommodations/admin/partials/image_card.html' with image=image %}
    {% endfor %}
</div>

<!-- Initialize via data attribute -->
<script>
    // Minimal inline script - just trigger initialization
    if (window.GalleryManager) {
        window.GalleryManager.init('gallery-grid');
    }
</script>
```

**External JS (static/js/gallery-manager.js):**
```javascript
/**
 * Gallery Manager - Sortable.js integration for image galleries
 * Handles drag-drop reordering with HTMX updates
 */
window.GalleryManager = (function() {
    'use strict';

    // Private state
    const instances = new Map(); // Store multiple sortable instances by ID

    // Private methods
    function destroyInstance(elementId) {
        const instance = instances.get(elementId);
        if (instance) {
            instance.destroy();
            instances.delete(elementId);
        }
    }

    function createSortable(element, accommodationId, siteSlug) {
        const sortable = new Sortable(element, {
            animation: 150,
            handle: '.drag-handle',
            ghostClass: 'opacity-50',
            chosenClass: 'ring-2 ring-primary',
            dragClass: 'rotate-3',

            onEnd: function(evt) {
                if (evt.oldIndex === evt.newIndex) return;

                // Update order via HTMX
                updateImageOrder(element, accommodationId, siteSlug);
            }
        });

        return sortable;
    }

    function updateImageOrder(element, accommodationId, siteSlug) {
        const imageIds = Array.from(element.querySelectorAll('[data-image-id]'))
            .map(card => card.dataset.imageId);

        // HTMX request to update order
        htmx.ajax('POST', `/${siteSlug}/alojamiento/admin/${accommodationId}/reorder-images/`, {
            target: '#gallery-grid',
            swap: 'outerHTML',
            values: {
                image_ids: JSON.stringify(imageIds)
            }
        });
    }

    // Public API
    return {
        /**
         * Initialize sortable for a gallery grid
         * @param {string} elementId - ID of the gallery container
         * @param {string} accommodationId - Accommodation ID
         * @param {string} siteSlug - Site slug for URL
         */
        init: function(elementId, accommodationId, siteSlug) {
            const element = document.getElementById(elementId);
            if (!element) {
                console.warn(`GalleryManager: Element #${elementId} not found`);
                return;
            }

            // Destroy existing instance if present
            destroyInstance(elementId);

            // Create new instance
            const sortable = createSortable(element, accommodationId, siteSlug);
            instances.set(elementId, sortable);

            console.log(`GalleryManager: Initialized for #${elementId}`);
        },

        /**
         * Destroy sortable instance
         * @param {string} elementId - ID of the gallery container
         */
        destroy: function(elementId) {
            destroyInstance(elementId);
            console.log(`GalleryManager: Destroyed for #${elementId}`);
        },

        /**
         * Destroy all instances
         */
        destroyAll: function() {
            instances.forEach((instance, id) => {
                instance.destroy();
            });
            instances.clear();
            console.log('GalleryManager: All instances destroyed');
        },

        /**
         * Check if element has active sortable
         * @param {string} elementId
         * @returns {boolean}
         */
        isActive: function(elementId) {
            return instances.has(elementId);
        }
    };
})();

// Auto-initialize on DOMContentLoaded
document.addEventListener('DOMContentLoaded', function() {
    // Find all gallery grids and initialize
    document.querySelectorAll('[data-gallery-sortable]').forEach(function(grid) {
        const accommodationId = grid.dataset.accommodationId;
        const siteSlug = grid.dataset.siteSlug;

        if (grid.id && accommodationId && siteSlug) {
            window.GalleryManager.init(grid.id, accommodationId, siteSlug);
        }
    });
});

// Re-initialize after HTMX swaps
document.body.addEventListener('htmx:afterSwap', function(event) {
    const target = event.detail.target;

    // Re-init if target is a gallery grid
    if (target.hasAttribute('data-gallery-sortable')) {
        const accommodationId = target.dataset.accommodationId;
        const siteSlug = target.dataset.siteSlug;

        window.GalleryManager.init(target.id, accommodationId, siteSlug);
    }

    // Re-init if target contains gallery grids
    target.querySelectorAll('[data-gallery-sortable]').forEach(function(grid) {
        const accommodationId = grid.dataset.accommodationId;
        const siteSlug = grid.dataset.siteSlug;

        window.GalleryManager.init(grid.id, accommodationId, siteSlug);
    });
});
```

**Updated Template:**
```html
<div
    id="gallery-grid"
    class="grid grid-cols-3 gap-4"
    data-gallery-sortable
    data-accommodation-id="{{ accommodation.pk }}"
    data-site-slug="{{ site.slug }}">

    {% for image in accommodation.images.all %}
        {% include 'accommodations/admin/partials/image_card.html' with image=image %}
    {% endfor %}
</div>

<!-- No inline script needed - auto-initialized by gallery-manager.js -->
```

**Load in base.html:**
```html
<!-- base.html -->
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.0/Sortable.min.js"></script>
<script src="{% static 'js/gallery-manager.js' %}"></script>
```

**Pros:**
- ✅ Complete separation of concerns (markup vs logic)
- ✅ Reusable across multiple templates
- ✅ Testable in isolation
- ✅ No global pollution (single namespace)
- ✅ Handles multiple instances cleanly
- ✅ Works perfectly with HTMX
- ✅ Easy to debug
- ✅ Professional code organization
- ✅ Can be minified/cached

**Cons:**
- Requires external file (extra HTTP request)
- More setup complexity
- Not suitable for truly one-off scripts

**Verdict:** ⭐ **RECOMMENDED for production** - Best practice

---

## Comparison Table

| Aspect | Option 1 (`var`) | Option 2 (`window`) | Option 3 (IIFE) | Option 4 (External) |
|--------|------------------|---------------------|-----------------|---------------------|
| **Global Pollution** | ❌ High | ⚠️ Medium | ✅ Low | ✅ Minimal |
| **Reusability** | ❌ Poor | ⚠️ Fair | ⚠️ Good | ✅ Excellent |
| **Testability** | ❌ Hard | ❌ Hard | ⚠️ Moderate | ✅ Easy |
| **Readability** | ✅ Simple | ✅ Simple | ⚠️ Moderate | ✅ Clear |
| **HTMX Compatible** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Multiple Instances** | ❌ No | ⚠️ Manual | ⚠️ Manual | ✅ Built-in |
| **Debugging** | ⚠️ Fair | ⚠️ Fair | ⚠️ Fair | ✅ Excellent |
| **Performance** | ✅ Good | ✅ Good | ✅ Good | ✅ Cached |
| **Maintenance** | ❌ Poor | ⚠️ Fair | ⚠️ Good | ✅ Excellent |
| **Best Practice** | ❌ No | ⚠️ Acceptable | ⚠️ Good | ✅ Yes |

---

## HTMX-Specific Considerations

### 1. Script Re-execution
When HTMX swaps content, inline `<script>` tags are **re-executed**. This means:

```javascript
// BAD - Will create duplicate event listeners
document.body.addEventListener('htmx:afterSwap', function() {
    initializeSortable();
});
```

```javascript
// GOOD - Use { once: true } for one-time listeners
document.body.addEventListener('htmx:afterSwap', function() {
    initializeSortable();
}, { once: true });

// BETTER - Check if listener already exists (external JS handles this)
```

### 2. Element Targeting
Always check if element exists before initializing:

```javascript
function initializeSortable() {
    const element = document.getElementById('gallery-grid');
    if (!element) {
        console.warn('Gallery grid not found');
        return;
    }

    // ... initialize
}
```

### 3. HTMX Events to Hook Into

```javascript
// After content swapped
document.body.addEventListener('htmx:afterSwap', function(event) {
    if (event.detail.target.id === 'gallery-grid') {
        reinitializeSortable();
    }
});

// Before swap (cleanup)
document.body.addEventListener('htmx:beforeSwap', function(event) {
    if (event.detail.target.id === 'gallery-grid') {
        cleanupSortable();
    }
});

// After request (loading states)
document.body.addEventListener('htmx:afterRequest', function(event) {
    // Hide loading spinner
});
```

### 4. Data Attributes for Configuration

Instead of passing parameters via inline scripts:

```html
<!-- BAD - Inline script with parameters -->
<div id="gallery-grid">...</div>
<script>
    initializeSortable('gallery-grid', {{ accommodation.pk }}, '{{ site.slug }}');
</script>

<!-- GOOD - Data attributes -->
<div
    id="gallery-grid"
    data-gallery-sortable
    data-accommodation-id="{{ accommodation.pk }}"
    data-site-slug="{{ site.slug }}">
    ...
</div>

<!-- External JS reads data attributes -->
```

### 5. Avoid Inline onclick Handlers

```html
<!-- BAD - Inline handler -->
<button onclick="deleteImage({{ image.pk }})">Delete</button>

<!-- GOOD - Event delegation in external JS -->
<button class="delete-image-btn" data-image-id="{{ image.pk }}">Delete</button>
```

```javascript
// External JS
document.addEventListener('click', function(event) {
    if (event.target.classList.contains('delete-image-btn')) {
        const imageId = event.target.dataset.imageId;
        deleteImage(imageId);
    }
});
```

---

## Final Recommendation

### For Production (Ribera Calma Project)

**Use Option 4: External JS with Module Pattern**

**Implementation Steps:**

1. **Create external file:**
   ```
   static/js/gallery-manager.js
   ```

2. **Load in base template:**
   ```html
   <!-- base.html -->
   <script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.0/Sortable.min.js"></script>
   <script src="{% static 'js/gallery-manager.js' %}"></script>
   ```

3. **Update templates to use data attributes:**
   ```html
   <div
       id="gallery-grid"
       data-gallery-sortable
       data-accommodation-id="{{ accommodation.pk }}"
       data-site-slug="{{ site.slug }}">
       {% for image in accommodation.images.all %}
           {% include 'accommodations/admin/partials/image_card.html' %}
       {% endfor %}
   </div>
   ```

4. **No inline scripts needed** - Auto-initialized by external JS

### For Rapid Prototyping Only

**Use Option 2: window object with guard pattern**

```html
<script>
window.sortableInstance = window.sortableInstance || null;

function initializeSortable() {
    if (window.sortableInstance) {
        window.sortableInstance.destroy();
    }

    const element = document.getElementById('gallery-grid');
    if (!element) return;

    window.sortableInstance = new Sortable(element, { ... });
}

document.addEventListener('DOMContentLoaded', initializeSortable);
</script>
```

**Migrate to Option 4 before production deployment.**

---

## Security Implications

### XSS Risks
All options are safe from XSS if:
1. No `eval()` or `Function()` constructors used
2. No `innerHTML` with user input
3. Django template escaping enabled (default)
4. CSRF tokens on HTMX POST requests

### CSP (Content Security Policy)
Inline scripts may violate CSP:

```python
# settings.py - Strict CSP
SECURE_CONTENT_SECURITY_POLICY = {
    'script-src': ["'self'", 'cdn.jsdelivr.net'],
    # 'unsafe-inline' NOT allowed
}
```

**Option 4 (external JS) is CSP-compliant.**
**Options 1-3 (inline scripts) require `'unsafe-inline'` in CSP.**

---

## Performance Implications

### Option 4 Advantages
- File cached by browser (faster subsequent loads)
- Can be minified
- Loaded once, used everywhere
- Reduces HTML size

### Inline Script Disadvantages (Options 1-3)
- Re-downloaded on every partial load
- Cannot be cached separately
- Increases HTML payload size

---

## Testing Considerations

### Option 4 is Testable

```javascript
// tests/static/test-gallery-manager.js (using Jasmine or Jest)
describe('GalleryManager', function() {
    beforeEach(function() {
        // Setup DOM
        document.body.innerHTML = `
            <div id="test-gallery"
                 data-gallery-sortable
                 data-accommodation-id="1"
                 data-site-slug="test-site">
            </div>
        `;
    });

    afterEach(function() {
        window.GalleryManager.destroyAll();
    });

    it('should initialize sortable', function() {
        window.GalleryManager.init('test-gallery', '1', 'test-site');
        expect(window.GalleryManager.isActive('test-gallery')).toBe(true);
    });

    it('should destroy existing instance before reinit', function() {
        window.GalleryManager.init('test-gallery', '1', 'test-site');
        window.GalleryManager.init('test-gallery', '1', 'test-site');

        // Should not throw error
        expect(window.GalleryManager.isActive('test-gallery')).toBe(true);
    });
});
```

**Options 1-3 are hard to test in isolation.**

---

## Migration Path (Current Project)

### Current State
You likely have inline scripts in `image_gallery_inline.html`.

### Step 1: Extract to External File (Week 4 - Day 1)

1. Create `static/js/gallery-manager.js` with Module Pattern
2. Load in `base.html`
3. Update templates to use data attributes
4. Test HTMX re-initialization

### Step 2: Remove Inline Scripts (Week 4 - Day 2)

1. Delete `<script>` tags from partials
2. Verify auto-initialization works
3. Test drag-drop functionality
4. Test HTMX partial updates

### Step 3: Add Tests (Week 4 - Day 3)

1. Write unit tests for `GalleryManager`
2. Test multi-instance handling
3. Test HTMX event integration

---

## Conclusion

**For Ribera Calma Project:**

✅ **Use Option 4 (External JS with Module Pattern)**

**Reasons:**
1. Professional code organization
2. Follows CLAUDE.md philosophy (separation of concerns)
3. Testable and maintainable
4. Works perfectly with HTMX
5. No global pollution
6. CSP-compliant
7. Cacheable and performant

**Fast Track Exception:**
For truly one-off inline scripts (e.g., analytics tracking), Option 2 is acceptable.

**Never use:**
- Option 1 (`var`) - Outdated, not best practice
- `eval()` or `Function()` constructors - Security risk

---

## References

- HTMX Documentation: https://htmx.org/docs/
- Sortable.js Documentation: https://github.com/SortableJS/Sortable
- JavaScript Module Pattern: https://addyosmani.com/resources/essentialjsdesignpatterns/book/#modulepatternjavascript
- CSP Guide: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- Django Static Files: https://docs.djangoproject.com/en/5.1/howto/static-files/

---

## Author
Frontend Integrator Agent

## Review Status
Ready for implementation
