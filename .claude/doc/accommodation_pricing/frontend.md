# Dynamic Pricing System - Frontend Architecture

## Overview

This document defines the frontend implementation for the dynamic pricing system, focusing on UI updates to the accommodation list filter and detail pages.

---

## 1. Widget Design: Capacity Input with +/- Buttons

### HTML Structure (DaisyUI Components)

```html
<!-- templates/accommodations/partials/capacity_widget.html -->
<div class="form-control">
    <label class="label">
        <span class="label-text font-semibold">Capacidad</span>
    </label>
    <div class="join">
        <button
            type="button"
            class="btn btn-sm join-item"
            id="capacity-decrease"
            aria-label="Disminuir capacidad">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M20 12H4" />
            </svg>
        </button>
        <input
            type="number"
            name="capacity"
            id="capacity-input"
            class="input input-sm input-bordered join-item w-20 text-center"
            value="1"
            min="1"
            max="99"
            readonly
            aria-label="Número de huéspedes"
        />
        <button
            type="button"
            class="btn btn-sm join-item"
            id="capacity-increase"
            aria-label="Aumentar capacidad">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4" />
            </svg>
        </button>
    </div>
    <label class="label">
        <span class="label-text-alt text-muted">Mínimo: 1 persona</span>
    </label>
</div>
```

### JavaScript Module: Capacity Widget Manager

```javascript
// static/js/capacity-widget.js
export default class CapacityWidget {
    constructor(config) {
        this.inputId = config.inputId || 'capacity-input';
        this.decreaseId = config.decreaseId || 'capacity-decrease';
        this.increaseId = config.increaseId || 'capacity-increase';
        this.minValue = config.minValue || 1;
        this.maxValue = config.maxValue || 99;
        this.onChange = config.onChange || null;
    }

    init() {
        this.input = document.getElementById(this.inputId);
        this.decreaseBtn = document.getElementById(this.decreaseId);
        this.increaseBtn = document.getElementById(this.increaseId);

        if (!this.input || !this.decreaseBtn || !this.increaseBtn) {
            console.error('CapacityWidget: Required elements not found');
            return;
        }

        this._attachEventListeners();
        this._updateButtonStates();
    }

    _attachEventListeners() {
        this.decreaseBtn.addEventListener('click', () => this._decrease());
        this.increaseBtn.addEventListener('click', () => this._increase());
        this.input.addEventListener('change', () => this._updateButtonStates());
    }

    _decrease() {
        const currentValue = parseInt(this.input.value) || this.minValue;
        if (currentValue > this.minValue) {
            this.input.value = currentValue - 1;
            this._triggerChange();
            this._updateButtonStates();
        }
    }

    _increase() {
        const currentValue = parseInt(this.input.value) || this.minValue;
        if (currentValue < this.maxValue) {
            this.input.value = currentValue + 1;
            this._triggerChange();
            this._updateButtonStates();
        }
    }

    _updateButtonStates() {
        const currentValue = parseInt(this.input.value) || this.minValue;

        // Disable decrease button at minimum
        if (currentValue <= this.minValue) {
            this.decreaseBtn.disabled = true;
            this.decreaseBtn.classList.add('btn-disabled');
        } else {
            this.decreaseBtn.disabled = false;
            this.decreaseBtn.classList.remove('btn-disabled');
        }

        // Disable increase button at maximum
        if (currentValue >= this.maxValue) {
            this.increaseBtn.disabled = true;
            this.increaseBtn.classList.add('btn-disabled');
        } else {
            this.increaseBtn.disabled = false;
            this.increaseBtn.classList.remove('btn-disabled');
        }
    }

    _triggerChange() {
        // Dispatch custom event for HTMX or other listeners
        const event = new CustomEvent('capacityChanged', {
            detail: { capacity: parseInt(this.input.value) }
        });
        this.input.dispatchEvent(event);

        // Trigger HTMX change event
        htmx.trigger(this.input, 'change');

        // Call onChange callback if provided
        if (this.onChange && typeof this.onChange === 'function') {
            this.onChange(parseInt(this.input.value));
        }
    }

    getValue() {
        return parseInt(this.input.value) || this.minValue;
    }

    setValue(value) {
        const newValue = Math.max(this.minValue, Math.min(this.maxValue, value));
        this.input.value = newValue;
        this._updateButtonStates();
    }

    destroy() {
        // Clean up event listeners
        this.decreaseBtn.removeEventListener('click', this._decrease);
        this.increaseBtn.removeEventListener('click', this._increase);
        this.input.removeEventListener('change', this._updateButtonStates);
    }
}
```

---

## 2. Filter Page Updates (`/alojamiento/`)

### Template Changes

**REMOVE:** capacity_min and capacity_max separate filters
**ADD:** Single "capacidad" input with +/- widget

### View Logic

```python
class AccommodationFilterView(ListView):
    """Filter accommodations with dynamic pricing calculation"""

    def get_queryset(self):
        site = get_object_or_404(Site, slug=self.kwargs["site_slug"], is_active=True)
        qs = AccommodationType.objects.filter(site=site, is_active=True)

        # Capacity filter (single input)
        capacity = self.request.GET.get("capacity")
        if capacity:
            try:
                capacity_int = int(capacity)
                qs = qs.filter(
                    capacity_min__lte=capacity_int,
                    capacity_max__gte=capacity_int
                )
            except ValueError:
                pass

        # Annotate with dynamic pricing
        accommodations = qs.select_related("site").order_by("order", "name")

        for accommodation in accommodations:
            if capacity:
                # Show calculated price for selected capacity
                accommodation.selected_capacity = int(capacity)
                accommodation.calculated_price = (
                    accommodation.calculate_price_for_capacity(int(capacity))
                )
            else:
                # Show "Desde" price (minimum capacity)
                accommodation.calculated_price = None
                accommodation.display_price = (
                    accommodation.calculate_price_for_capacity(
                        accommodation.capacity_min
                    )
                )

        return accommodations
```

---

## 3. Detail Page Updates (`/alojamiento/<slug>/`)

### Price Table (if per-person pricing exists)

```django
{% if accommodation.has_per_person_pricing %}
<div class="card bg-base-200 shadow-lg mb-6">
    <div class="card-body">
        <h3 class="card-title text-lg mb-4">Precios por Capacidad</h3>
        <div class="overflow-x-auto">
            <table class="table table-zebra table-sm">
                <thead>
                    <tr>
                        <th>Capacidad</th>
                        <th class="text-right">Precio por Noche</th>
                    </tr>
                </thead>
                <tbody>
                    {% for capacity, price in accommodation.price_table %}
                    <tr{% if capacity == accommodation.capacity_max %} class="font-semibold"{% endif %}>
                        <td>
                            {{ capacity }} persona{{ capacity|pluralize }}
                            {% if capacity == accommodation.capacity_max %}
                            <span class="badge badge-sm badge-primary ml-2">Precio base</span>
                            {% endif %}
                        </td>
                        <td class="text-right">${{ price|floatformat:0|intcomma }}</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</div>
{% endif %}
```

### Price Preview Enhancement

**Current:** Triggers on date-range-picker change only
**New:** Trigger when BOTH date-range AND num_guests are complete

```javascript
// Update price preview when BOTH dates and guests are selected
function updatePricePreview() {
    // Check if both inputs are complete
    if (!selectedDates || !selectedDates.checkIn || !selectedDates.checkOut) {
        hidePricePreview();
        return;
    }

    const numGuests = guestsWidget.getValue();
    if (!numGuests || numGuests < config.capacityMin || numGuests > config.capacityMax) {
        hidePricePreview();
        return;
    }

    // Fetch price preview from API
    const params = new URLSearchParams({
        check_in: formatDate(selectedDates.checkIn),
        check_out: formatDate(selectedDates.checkOut),
        accommodation_type: config.accommodationId,
        num_guests: numGuests
    });

    fetch(`${config.pricePreviewUrl}?${params.toString()}`)
        .then(response => response.json())
        .then(data => showPricePreview(data));
}
```

---

## 4. HTMX Patterns

### Custom Events

```javascript
// Listen for capacity change
document.getElementById('capacity-input').addEventListener('capacityChanged', function(e) {
    console.log('Capacity changed:', e.detail.capacity);
});

// Trigger HTMX refresh
htmx.trigger('#accommodation-grid', 'refreshList');
```

### Error Handling

```javascript
document.body.addEventListener('htmx:responseError', function(event) {
    const statusCode = event.detail.xhr.status;
    let message = 'Error en la solicitud';

    if (statusCode === 400) {
        message = 'Datos inválidos';
    } else if (statusCode === 404) {
        message = 'Recurso no encontrado';
    }

    showToast(message, 'error');
});
```

---

## 5. DaisyUI Components

### Button Groups

```html
<div class="join">
    <button class="btn btn-sm join-item">-</button>
    <input type="number" class="input input-sm input-bordered join-item w-20 text-center" readonly />
    <button class="btn btn-sm join-item">+</button>
</div>
```

### Loading Indicators

```html
<div class="htmx-indicator">
    <span class="loading loading-spinner loading-sm"></span>
    <span class="ml-2">Cargando...</span>
</div>
```

### Alerts

```html
<div class="alert alert-info">
    <svg><!-- icon --></svg>
    <span>El precio se ajusta según la capacidad.</span>
</div>
```

---

## Summary

**Frontend Implementation:**

1. ✅ **CapacityWidget** - ES6 module with +/- buttons
2. ✅ **Filter Page** - Single capacity input with dynamic pricing
3. ✅ **Detail Page** - Price table + enhanced price preview
4. ✅ **HTMX Integration** - Partial rendering, custom events
5. ✅ **DaisyUI Components** - Consistent styling

**Estimated Implementation Time:** 1-2 days (Fast Track Mode)
