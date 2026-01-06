# Flatpickr Implementation Guide - Best Practices

## Critical Lessons Learned

**Date:** 2025-11-13
**Context:** After multiple attempts, discovered the correct way to implement Flatpickr with proper visibility and positioning.

---

## The Problem

Initial implementations of Flatpickr had calendar visibility issues:

1. **Template Block Name Issue**: JavaScript in wrong block (`extra_js` instead of `extra_scripts`)
2. **Opacity Issue**: `animate` class causing `opacity: 0`
3. **Position Issue**: `position: fixed` + `appendTo: document.body` rendering calendar far from input

---

## The Correct Solution

### 1. HTML Structure with Position Relative Wrapper

**CRITICAL:** Wrap the input in a `position: relative` container so Flatpickr calendar renders relative to it.

```html
<!-- Date Range Input -->
<div class="form-control">
    <label class="label">
        <span class="label-text">Fechas</span>
    </label>

    <!-- CRITICAL: position: relative wrapper for Flatpickr calendar positioning -->
    <div style="position: relative; z-index: 100;">
        <input
            type="text"
            id="date-range"
            name="date_range"
            placeholder="Seleccionar fechas"
            class="input input-bordered w-full"
            readonly
        />
    </div>

    <input type="hidden" id="check-in" name="check_in" />
    <input type="hidden" id="check-out" name="check_out" />
</div>
```

**Why it works:**
- Flatpickr calendar has `position: absolute` by default
- Calendar positions itself relative to nearest parent with `position: relative`
- `z-index: 100` ensures calendar appears above form elements

---

### 2. Flatpickr JavaScript Configuration

```javascript
const flatpickrConfig = {
    mode: "range",
    minDate: "today",
    locale: "es",
    dateFormat: "Y-m-d",
    disable: [],  // Blocked dates array

    // CRITICAL: DO NOT use appendTo: document.body
    // Let calendar render relative to input wrapper (position: relative)

    animate: false,  // CRITICAL: Prevents opacity:0 issue
    static: false,   // Use default positioning (absolute, relative to wrapper)

    onChange: function(selectedDates) {
        if (selectedDates.length === 2) {
            // Handle date selection
        }
    }
};

const dateRangePicker = flatpickr("#date-range", flatpickrConfig);
```

**Key Configuration Options:**

| Option | Value | Why |
|--------|-------|-----|
| `animate` | `false` | Prevents `opacity: 0` bug from CSS animation |
| `static` | `false` | Allows absolute positioning (default) |
| `appendTo` | **NOT USED** | Let calendar render next to input, not in body |

---

### 3. Parent Container with Overflow Visible

**CRITICAL:** If Flatpickr input is inside a card/container with `overflow: hidden`, the calendar will be cut off.

```html
<!-- CRITICAL: overflow: visible to prevent cutting off Flatpickr calendar -->
<div class="card bg-base-100 shadow-xl" style="overflow: visible;">
    <div class="card-body" style="overflow: visible;">
        <!-- Flatpickr input here -->
    </div>
</div>
```

**Common containers that need `overflow: visible`:**
- DaisyUI cards (`<div class="card">`)
- Sticky sidebars
- Fixed position elements
- Modals (though modals should use different approach - see below)

---

### 4. Global CSS Configuration

In `base_only_body.html` or global CSS file:

```css
/* Flatpickr z-index fix for DaisyUI modals */
/* CRITICAL: DO NOT force position:fixed - let Flatpickr use absolute positioning */
.flatpickr-calendar:not(.static) {
    z-index: 99999 !important;
    /* position: absolute by default (relative to input wrapper with position:relative) */
}

.flatpickr-calendar.open:not(.static) {
    z-index: 99999 !important;
}

/* Static calendars (inline mode) - render in normal flow */
.flatpickr-calendar.static {
    position: relative !important;
    display: block !important;
    margin-top: 0.5rem;
    z-index: 1;
}
```

**DO NOT include:**
```css
/* ❌ WRONG - forces fixed positioning */
.flatpickr-calendar:not(.static) {
    position: fixed !important;  /* ❌ This breaks relative positioning */
}
```

---

### 5. Template Block Naming

**CRITICAL:** Use correct template block name.

```django
{% extends 'base.html' %}

{% block extra_scripts %}  <!-- ✅ CORRECT - matches base_only_body.html line 341 -->
<script>
    // Flatpickr initialization code here
</script>
{% endblock %}
```

**NOT:**
```django
{% block extra_js %}  <!-- ❌ WRONG - block doesn't exist -->
```

To verify correct block name, check parent template:
```bash
grep "block extra_" templates/base_only_body.html
```

---

## Special Case: Flatpickr in Modals

When Flatpickr is inside a DaisyUI `<dialog>` modal, use `appendTo` to render calendar in modal:

```javascript
const modalDatePicker = flatpickr("#modal-date-input", {
    mode: "range",
    locale: "es",
    appendTo: document.getElementById('my-modal'),  // Render inside modal
    animate: false,
    onChange: function(selectedDates) {
        // Handle dates
    }
});
```

**Why:** Modals use `<dialog>` element which creates a stacking context. Calendar needs to be inside modal to inherit correct z-index.

---

## Debugging Checklist

If Flatpickr calendar is not visible:

### 1. Check JavaScript Execution
```javascript
console.log('[DEBUG] Flatpickr loaded:', typeof flatpickr !== 'undefined');
console.log('[DEBUG] Flatpickr instance:', dateRangePicker);
```

### 2. Check Calendar in DOM
```javascript
setTimeout(() => {
    const calendar = document.querySelector('.flatpickr-calendar');
    if (calendar) {
        console.log('[DEBUG] Calendar found:', calendar);
        console.log('[DEBUG] Calendar parent:', calendar.parentElement);
    } else {
        console.error('[ERROR] Calendar NOT in DOM');
    }
}, 500);
```

### 3. Check Computed Styles
```javascript
const calendar = document.querySelector('.flatpickr-calendar');
const styles = window.getComputedStyle(calendar);

console.log('[DEBUG] position:', styles.position);      // Should be "absolute"
console.log('[DEBUG] display:', styles.display);        // Should be "block" or "none"
console.log('[DEBUG] visibility:', styles.visibility);  // Should be "visible"
console.log('[DEBUG] opacity:', styles.opacity);        // Should be "1"
console.log('[DEBUG] z-index:', styles.zIndex);         // Should be "99999"
console.log('[DEBUG] top:', styles.top);                // Should have px value
console.log('[DEBUG] left:', styles.left);              // Should have px value
```

### 4. Expected Output (Working)
```
[DEBUG] position: absolute
[DEBUG] display: block
[DEBUG] visibility: visible
[DEBUG] opacity: 1
[DEBUG] z-index: 99999
[DEBUG] top: 123.45px
[DEBUG] left: 67.89px
```

### 5. Common Problems

| Symptom | Cause | Fix |
|---------|-------|-----|
| Calendar not visible | `opacity: 0` | Add `animate: false` |
| Calendar far from input | `position: fixed` + `appendTo: body` | Remove `appendTo`, use relative wrapper |
| Calendar cut off | `overflow: hidden` on parent | Add `overflow: visible` to parent |
| JavaScript not running | Wrong template block name | Use `{% block extra_scripts %}` |
| Calendar behind modal | Low z-index | Ensure `.flatpickr-calendar` has `z-index: 99999` |

---

## Complete Working Example

```html
<!-- templates/accommodations/detail.html -->
{% extends 'base.html' %}
{% load static %}

{% block content %}
<div class="card bg-base-100 shadow-xl" style="overflow: visible;">
    <div class="card-body" style="overflow: visible;">
        <form class="space-y-4">
            <div class="form-control">
                <label class="label">
                    <span class="label-text">Fechas</span>
                </label>

                <!-- Position relative wrapper -->
                <div style="position: relative; z-index: 100;">
                    <input
                        type="text"
                        id="date-range"
                        placeholder="Seleccionar fechas"
                        class="input input-bordered w-full"
                        readonly
                    />
                </div>
            </div>
        </form>
    </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script>
function initDatePicker() {
    const dateRangePicker = flatpickr("#date-range", {
        mode: "range",
        minDate: "today",
        locale: "es",
        dateFormat: "Y-m-d",
        disable: [],
        animate: false,  // CRITICAL
        static: false,   // CRITICAL
        onChange: function(selectedDates) {
            if (selectedDates.length === 2) {
                console.log('Dates selected:', selectedDates);
            }
        }
    });

    console.log('[DEBUG] Flatpickr initialized:', dateRangePicker);
}

// Initialize when DOM ready
if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initDatePicker);
} else {
    initDatePicker();
}
</script>
{% endblock %}
```

---

## Performance Considerations

### CDN Loading
Flatpickr CSS and JS should be loaded from CDN in `base_only_body.html`:

```html
<!-- Flatpickr -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr@4.6.13/dist/flatpickr.min.css">
<script src="https://cdn.jsdelivr.net/npm/flatpickr@4.6.13/dist/flatpickr.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/flatpickr@4.6.13/dist/l10n/es.js"></script>
```

**Don't re-load** in individual templates - it's already global.

### Lazy Initialization
For pages with multiple Flatpickr instances, initialize only when needed:

```javascript
// Initialize only when user focuses input
document.getElementById('date-range').addEventListener('focus', function() {
    if (!this._flatpickr) {
        this._flatpickr = flatpickr(this, flatpickrConfig);
        this._flatpickr.open();
    }
}, { once: true });
```

---

## Testing Checklist

Before marking Flatpickr implementation as complete:

- [ ] Calendar appears when clicking input
- [ ] Calendar positioned directly below input (not far away)
- [ ] Calendar has `position: absolute` (check computed styles)
- [ ] Calendar has `opacity: 1` (check computed styles)
- [ ] Calendar not cut off by parent containers
- [ ] Dates can be selected in range mode
- [ ] onChange handler fires correctly
- [ ] Blocked dates are disabled
- [ ] Works on mobile (touch events)
- [ ] Works in all major browsers (Chrome, Firefox, Safari, Edge)
- [ ] No console errors
- [ ] Template block name correct (`extra_scripts`)

---

## References

- **Flatpickr Documentation:** https://flatpickr.js.org/
- **Flatpickr Options:** https://flatpickr.js.org/options/
- **Spanish Locale:** https://flatpickr.js.org/localization/
- **DaisyUI Cards:** https://daisyui.com/components/card/
- **DaisyUI Modals:** https://daisyui.com/components/modal/

---

## Version History

- **2025-11-13:** Initial guide created based on successful implementation in `accommodation:detail`
- **Tested in:** `templates/accommodations/detail.html` (reservation form inline)
- **Browser Compatibility:** Chrome 120+, Firefox 121+, Safari 17+, Edge 120+
- **Flatpickr Version:** 4.6.13
- **DaisyUI Version:** 4.12+

---

## Quick Reference Card

```javascript
// ✅ CORRECT Flatpickr Implementation
const picker = flatpickr("#date-input", {
    mode: "range",
    locale: "es",
    animate: false,  // Prevents opacity:0
    static: false,   // Allows absolute positioning
    // NO appendTo - let it render relative to wrapper
});
```

```html
<!-- ✅ CORRECT HTML Structure -->
<div style="position: relative; z-index: 100;">
    <input id="date-input" type="text" readonly />
</div>
```

```css
/* ✅ CORRECT Global CSS */
.flatpickr-calendar:not(.static) {
    z-index: 99999 !important;
    /* position: absolute by default */
}
```

```django
<!-- ✅ CORRECT Template Block -->
{% block extra_scripts %}
<script>
    // Flatpickr code here
</script>
{% endblock %}
```
