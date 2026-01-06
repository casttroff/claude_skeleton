# Frontend Research: Google Maps Integration (RIBERA-018)

## Overview

RIBERA-018 requires embedding an interactive Google Map on the Home page showing the resort location with a marker and InfoWindow. The implementation must be responsive, accessible, and follow vanilla JavaScript patterns (no frameworks).

**Feature Type**: Fast Track Mode (secondary feature, UI-focused)

**Tech Stack**:
- Google Maps JavaScript API
- Vanilla JavaScript (no frameworks)
- DaisyUI for container styling
- HTMX for potential dynamic loading (optional)

---

## Google Maps JavaScript API Integration

### API Loading Strategy

**Recommended: Async Dynamic Loading**

Use the modern `importLibrary` approach for better performance and tree-shaking:

```html
<!-- In base.html or home.html <head> -->
<script>
(g=>{
  var h,a,k,p="The Google Maps JavaScript API",c="google",l="importLibrary",q="__ib__",m=document,b=window;
  b=b[c]||(b[c]={});
  var d=b.maps||(b.maps={}),r=new Set,e=new URLSearchParams;
  u=()=>h||(h=new Promise(async(f,n)=>{
    a=m.createElement("script");
    e.set("libraries",[...r]+"");
    for(k in g)e.set(k.replace(/[A-Z]/g,t=>"_"+t[0].toLowerCase()),g[k]);
    e.set("callback",c+".maps."+q);
    a.src=`https://maps.${c}apis.com/maps/api/js?`+e;
    d[q]=f;
    a.onerror=()=>h=n(Error(p+" could not load."));
    a.nonce=m.querySelector("script[nonce]")?.nonce||"";
    m.head.append(a)
  }));
  d[l]?console.warn(p+" only loads once. Ignoring:",g):d[l]=(f,...n)=>r.add(f)&&u().then(()=>d[l](f,...n))
})({key: "{{ GOOGLE_MAPS_API_KEY }}", v: "weekly"});
</script>
```

**Alternative: Simple Script Tag (Easier for Fast Track)**

```html
<!-- Simpler approach for Fast Track Mode -->
<script async defer
  src="https://maps.googleapis.com/maps/api/js?key={{ GOOGLE_MAPS_API_KEY }}&callback=initMap&libraries=marker">
</script>
```

**Recommendation**: Use the simple script tag approach for Fast Track Mode. It's easier to debug and sufficient for basic map display.

---

## Map Initialization Pattern

### HTML Structure

```html
<!-- templates/home.html -->
{% extends "base.html" %}

{% block content %}
<!-- Other home content -->

<!-- Google Maps Section -->
<section class="py-12 bg-base-200" id="location-section">
  <div class="container mx-auto px-4">
    <h2 class="text-3xl font-bold text-center mb-8">¿Cómo llegar?</h2>

    <!-- Map Container with DaisyUI styling -->
    <div class="card bg-base-100 shadow-xl">
      <div class="card-body p-0">
        <!-- Map element with responsive height -->
        <div id="map"
             class="w-full h-96 lg:h-[500px] rounded-2xl"
             role="application"
             aria-label="Mapa de ubicación de {{ site.name }}"
             data-lat="{{ site.location_lat }}"
             data-lng="{{ site.location_lng }}"
             data-name="{{ site.name }}"
             data-address="{{ site.address }}">
        </div>
      </div>
    </div>

    <!-- Directions Link -->
    <div class="text-center mt-6">
      <a href="https://www.google.com/maps/dir/?api=1&destination={{ site.location_lat }},{{ site.location_lng }}"
         target="_blank"
         rel="noopener noreferrer"
         class="btn btn-primary btn-lg">
        <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                d="M9 20l-5.447-2.724A1 1 0 013 16.382V5.618a1 1 0 011.447-.894L9 7m0 13l6-3m-6 3V7m6 10l4.553 2.276A1 1 0 0021 18.382V7.618a1 1 0 00-.553-.894L15 4m0 13V4m0 0L9 7"/>
        </svg>
        Cómo llegar
      </a>
    </div>
  </div>
</section>

<!-- Map loading error fallback -->
<div id="map-error" class="hidden alert alert-error mt-4">
  <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
          d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
  </svg>
  <span>No se pudo cargar el mapa. Por favor, intente más tarde.</span>
</div>

{% endblock %}

{% block extra_js %}
<!-- Map initialization script -->
<script src="{% static 'js/google-maps.js' %}"></script>
{% endblock %}
```

**Key HTML Features**:
- `data-*` attributes store site coordinates and info
- DaisyUI classes: `card`, `card-body`, `btn`, `btn-primary`
- Responsive heights: `h-96` (mobile), `lg:h-[500px]` (desktop)
- `role="application"` for screen readers
- `aria-label` for accessibility
- Error fallback with `alert` component

---

## JavaScript Implementation

### Map Initialization (static/js/google-maps.js)

```javascript
/**
 * Google Maps integration for resort location
 * Vanilla JavaScript - no frameworks
 *
 * @requires Google Maps JavaScript API
 */

let map;
let marker;
let infoWindow;

/**
 * Initialize Google Map with site location
 * Called by Google Maps API callback
 *
 * @returns {void}
 */
function initMap() {
  const mapElement = document.getElementById('map');

  // Guard: Map element must exist
  if (!mapElement) {
    console.error('Map element not found');
    return;
  }

  // Extract site data from data attributes
  const lat = parseFloat(mapElement.dataset.lat);
  const lng = parseFloat(mapElement.dataset.lng);
  const siteName = mapElement.dataset.name;
  const siteAddress = mapElement.dataset.address;

  // Validate coordinates
  if (isNaN(lat) || isNaN(lng)) {
    showMapError('Coordenadas inválidas');
    return;
  }

  const position = { lat: lat, lng: lng };

  try {
    // Initialize map
    map = new google.maps.Map(mapElement, {
      center: position,
      zoom: 15,
      mapTypeControl: true,
      mapTypeControlOptions: {
        style: google.maps.MapTypeControlStyle.DROPDOWN_MENU,
        position: google.maps.ControlPosition.TOP_RIGHT
      },
      streetViewControl: true,
      streetViewControlOptions: {
        position: google.maps.ControlPosition.RIGHT_BOTTOM
      },
      zoomControl: true,
      zoomControlOptions: {
        position: google.maps.ControlPosition.RIGHT_BOTTOM
      },
      fullscreenControl: true,
      fullscreenControlOptions: {
        position: google.maps.ControlPosition.RIGHT_TOP
      }
    });

    // Create marker
    marker = new google.maps.Marker({
      position: position,
      map: map,
      title: siteName,
      animation: google.maps.Animation.DROP
    });

    // Create InfoWindow
    const infoWindowContent = `
      <div class="p-3">
        <h3 class="font-bold text-lg mb-2">${siteName}</h3>
        <p class="text-sm text-gray-600 mb-3">${siteAddress}</p>
        <a href="https://www.google.com/maps/dir/?api=1&destination=${lat},${lng}"
           target="_blank"
           rel="noopener noreferrer"
           class="text-primary hover:underline text-sm font-semibold">
          Ver indicaciones →
        </a>
      </div>
    `;

    infoWindow = new google.maps.InfoWindow({
      content: infoWindowContent
    });

    // Open InfoWindow on marker click
    marker.addListener('click', () => {
      infoWindow.open({
        anchor: marker,
        map: map
      });
    });

    // Auto-open InfoWindow on load (optional, for better UX)
    setTimeout(() => {
      infoWindow.open({
        anchor: marker,
        map: map
      });
    }, 500);

  } catch (error) {
    console.error('Map initialization error:', error);
    showMapError('Error al cargar el mapa');
  }
}

/**
 * Show map error message to user
 *
 * @param {string} message - Error message to display
 * @returns {void}
 */
function showMapError(message) {
  const mapElement = document.getElementById('map');
  const errorElement = document.getElementById('map-error');

  if (mapElement) {
    mapElement.style.display = 'none';
  }

  if (errorElement) {
    errorElement.classList.remove('hidden');
    const errorText = errorElement.querySelector('span');
    if (errorText) {
      errorText.textContent = message;
    }
  }
}

/**
 * Handle Google Maps API load errors
 * Called if script fails to load
 *
 * @returns {void}
 */
function gm_authFailure() {
  showMapError('Error de autenticación con Google Maps. Verifique la API key.');
}

// Expose to window for Google Maps callback
window.initMap = initMap;
window.gm_authFailure = gm_authFailure;
```

**Key JavaScript Features**:
- **Type safety**: JSDoc comments for documentation
- **Error handling**: Try-catch, validation, fallback UI
- **Accessibility**: ARIA labels, keyboard navigation (native to Google Maps)
- **Performance**: Lazy InfoWindow opening (500ms delay)
- **Clean code**: Single Responsibility Principle, early returns
- **No console pollution**: Logging only for errors

---

## Responsive Design

### Mobile-First Approach

```css
/* static/css/custom.css (if needed for additional styling) */

/* Base mobile styles */
#map {
  height: 384px; /* h-96 in Tailwind = 24rem = 384px */
  width: 100%;
  border-radius: 1rem;
}

/* Tablet and up */
@media (min-width: 1024px) {
  #map {
    height: 500px;
  }
}

/* Ensure map controls are accessible on mobile */
.gm-style .gm-style-iw-c {
  max-width: 90vw !important;
}

.gm-style .gm-style-iw-d {
  overflow: auto !important;
  max-height: 300px !important;
}
```

**Responsive Breakpoints**:
- **Mobile (< 640px)**: 384px height (h-96)
- **Tablet (640-1024px)**: 384px height
- **Desktop (> 1024px)**: 500px height (lg:h-[500px])

**DaisyUI Responsive Classes**:
- `container`: Responsive padding and max-width
- `mx-auto`: Center container
- `px-4`: Horizontal padding
- `lg:h-[500px]`: Height adjustment for large screens

---

## Accessibility Considerations

### WCAG 2.1 Level A Compliance

**HTML Accessibility**:
```html
<!-- Map container -->
<div id="map"
     role="application"
     aria-label="Mapa interactivo mostrando la ubicación de {{ site.name }}"
     tabindex="0">
</div>

<!-- Directions button -->
<a href="https://www.google.com/maps/dir/?api=1&destination=..."
   target="_blank"
   rel="noopener noreferrer"
   class="btn btn-primary"
   aria-label="Obtener indicaciones para llegar a {{ site.name }} en Google Maps">
  Cómo llegar
</a>
```

**Keyboard Navigation**:
- Google Maps API includes native keyboard navigation:
  - **Arrow keys**: Pan map
  - **+/- keys**: Zoom in/out
  - **Tab**: Navigate between controls
  - **Enter/Space**: Activate controls

**Screen Reader Support**:
- `role="application"`: Tells screen readers this is an interactive widget
- `aria-label`: Describes the map purpose
- InfoWindow content is readable by screen readers
- All interactive elements (marker, controls) are focusable

**Color Contrast**:
- Default Google Maps colors meet WCAG AA
- InfoWindow text uses readable contrast
- Custom styles avoid color-only information

**Alternative Text**:
```html
<!-- Fallback for users who cannot see the map -->
<noscript>
  <div class="alert alert-warning">
    <p>
      Para ver el mapa interactivo, por favor habilite JavaScript en su navegador.
      Ubicación: {{ site.address }}
    </p>
    <a href="https://www.google.com/maps/search/?api=1&query={{ site.location_lat }},{{ site.location_lng }}"
       target="_blank"
       class="btn btn-outline">
      Ver en Google Maps
    </a>
  </div>
</noscript>
```

---

## Event Handlers

### Marker Click Event

```javascript
// Open InfoWindow on marker click
marker.addListener('click', () => {
  infoWindow.open({
    anchor: marker,
    map: map
  });
});
```

### Map Events (Optional Enhancements)

```javascript
// Track when user interacts with map (for analytics)
map.addListener('click', (event) => {
  console.log('Map clicked at:', event.latLng.toJSON());
  // Could send to analytics via HTMX:
  // htmx.ajax('POST', '/api/analytics/map-interaction', {...})
});

// Detect zoom changes
map.addListener('zoom_changed', () => {
  console.log('Zoom level:', map.getZoom());
});

// Center changed (user panned map)
map.addListener('center_changed', () => {
  console.log('Center:', map.getCenter().toJSON());
});
```

**Recommendation for Fast Track**: Skip optional event handlers. Only implement marker click for InfoWindow.

---

## Error Handling

### API Load Failures

**Scenario 1: Invalid API Key**

```javascript
function gm_authFailure() {
  showMapError('Error de autenticación. Por favor contacte al administrador.');

  // Log to backend for monitoring
  if (typeof htmx !== 'undefined') {
    htmx.ajax('POST', '/api/errors/log', {
      values: {
        error_type: 'google_maps_auth_failure',
        message: 'Invalid Google Maps API key'
      }
    });
  }
}
```

**Scenario 2: Network Error**

```html
<script async defer
  src="https://maps.googleapis.com/maps/api/js?key={{ GOOGLE_MAPS_API_KEY }}&callback=initMap"
  onerror="handleMapScriptError()">
</script>

<script>
function handleMapScriptError() {
  showMapError('No se pudo cargar Google Maps. Verifique su conexión a internet.');
}
</script>
```

**Scenario 3: Invalid Coordinates**

```javascript
function initMap() {
  const lat = parseFloat(mapElement.dataset.lat);
  const lng = parseFloat(mapElement.dataset.lng);

  // Validate coordinates
  if (isNaN(lat) || isNaN(lng)) {
    showMapError('Coordenadas de ubicación no disponibles');
    return;
  }

  // Additional validation: coordinates within valid ranges
  if (lat < -90 || lat > 90 || lng < -180 || lng > 180) {
    showMapError('Coordenadas inválidas');
    return;
  }

  // Proceed with map initialization
  // ...
}
```

**Scenario 4: Missing Site Data**

```python
# In Django view (home.html context)
def home_view(request, site_slug):
    site = get_object_or_404(Site, slug=site_slug)

    # Validate location data exists
    if not site.location_lat or not site.location_lng:
        messages.warning(request, 'La ubicación del mapa no está configurada.')

    return render(request, 'home.html', {'site': site})
```

```html
<!-- In template: Only show map section if coordinates exist -->
{% if site.location_lat and site.location_lng %}
<section id="location-section">
  <!-- Map here -->
</section>
{% else %}
<section class="py-12">
  <div class="container mx-auto">
    <div class="alert alert-info">
      <p>La ubicación en el mapa estará disponible próximamente.</p>
    </div>
  </div>
</section>
{% endif %}
```

---

## DaisyUI Integration

### Map Container Styling

```html
<!-- Card wrapper for consistent styling -->
<div class="card bg-base-100 shadow-xl">
  <div class="card-body p-0">
    <div id="map" class="w-full h-96 lg:h-[500px] rounded-2xl"></div>
  </div>
</div>
```

**DaisyUI Classes Used**:
- `card`: Container with rounded corners and shadow
- `bg-base-100`: Background color from theme
- `shadow-xl`: Large shadow for depth
- `card-body`: Padding wrapper (set to `p-0` to remove padding for map)
- `rounded-2xl`: Large border radius

### Directions Button

```html
<a href="..." class="btn btn-primary btn-lg">
  <svg class="w-5 h-5 mr-2">...</svg>
  Cómo llegar
</a>
```

**DaisyUI Classes**:
- `btn`: Base button styles
- `btn-primary`: Primary color from site theme
- `btn-lg`: Large button size
- Icon with `mr-2` for spacing

### Error Alert

```html
<div class="alert alert-error">
  <svg class="w-6 h-6">...</svg>
  <span>Error message</span>
</div>
```

**DaisyUI Classes**:
- `alert`: Alert container
- `alert-error`: Error styling (red background)
- Flexbox layout with icon and text

---

## Performance Optimization

### Lazy Loading Strategy

**Option 1: Load on scroll (Intersection Observer)**

```javascript
/**
 * Lazy load map when section is visible
 * Improves initial page load performance
 */
document.addEventListener('DOMContentLoaded', () => {
  const mapSection = document.getElementById('location-section');

  if (!mapSection) return;

  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        // Load Google Maps script when section is visible
        loadGoogleMapsScript();
        observer.unobserve(mapSection);
      }
    });
  }, {
    rootMargin: '100px' // Load 100px before section is visible
  });

  observer.observe(mapSection);
});

function loadGoogleMapsScript() {
  const script = document.createElement('script');
  script.src = `https://maps.googleapis.com/maps/api/js?key={{ GOOGLE_MAPS_API_KEY }}&callback=initMap`;
  script.async = true;
  script.defer = true;
  script.onerror = handleMapScriptError;
  document.head.appendChild(script);
}
```

**Option 2: Load on user interaction (Click to load)**

```html
<!-- Placeholder with load button -->
<div id="map-placeholder" class="w-full h-96 bg-base-200 rounded-2xl flex items-center justify-center">
  <button onclick="loadMap()" class="btn btn-primary btn-lg">
    Cargar mapa
  </button>
</div>

<div id="map" class="hidden w-full h-96 rounded-2xl"></div>

<script>
function loadMap() {
  document.getElementById('map-placeholder').classList.add('hidden');
  document.getElementById('map').classList.remove('hidden');
  loadGoogleMapsScript();
}
</script>
```

**Recommendation for Fast Track**: Use simple eager loading (script in `<head>`). Lazy loading adds complexity and may not be necessary for a single map on the home page.

### Caching

Google Maps API automatically caches tiles and resources. No additional caching needed.

---

## HTMX Integration (Optional)

### Dynamic Map Update

If site coordinates change via admin, update map without page reload:

```html
<!-- In admin panel -->
<form hx-post="{% url 'sites:update-location' site.id %}"
      hx-target="#map-container"
      hx-swap="outerHTML">
  {% csrf_token %}
  <input type="number" name="location_lat" step="0.0000001" value="{{ site.location_lat }}">
  <input type="number" name="location_lng" step="0.0000001" value="{{ site.location_lng }}">
  <button type="submit" class="btn btn-primary">Actualizar ubicación</button>
</form>

<div id="map-container">
  <div id="map" data-lat="{{ site.location_lat }}" data-lng="{{ site.location_lng }}"></div>
</div>
```

```python
# views.py
def update_location(request, site_id):
    site = get_object_or_404(Site, id=site_id)

    site.location_lat = Decimal(request.POST['location_lat'])
    site.location_lng = Decimal(request.POST['location_lng'])
    site.save()

    # Return updated map container HTML
    html = render_to_string('sites/partials/map_container.html', {
        'site': site
    }, request=request)

    response = HttpResponse(html)
    # Trigger map re-initialization
    response['HX-Trigger'] = 'mapUpdated'
    return response
```

```javascript
// Listen for map update event
document.body.addEventListener('mapUpdated', () => {
  // Reinitialize map with new coordinates
  initMap();
});
```

**Recommendation for Fast Track**: Skip HTMX integration. Coordinates rarely change; page reload is acceptable.

---

## Testing Checklist

### Manual Testing

- [ ] Map loads correctly on desktop
- [ ] Map loads correctly on mobile
- [ ] Map loads correctly on tablet
- [ ] Marker displays at correct location
- [ ] InfoWindow opens on marker click
- [ ] InfoWindow displays site name and address
- [ ] Directions link opens Google Maps in new tab
- [ ] Directions link works on mobile (opens Google Maps app)
- [ ] Map controls are accessible (zoom, street view, map type)
- [ ] Keyboard navigation works (arrow keys, tab, enter)
- [ ] Screen reader announces map and controls
- [ ] Error message displays if API fails to load
- [ ] Error message displays if coordinates are invalid
- [ ] Map is responsive (height adjusts on different screens)
- [ ] No console errors

### Browser Compatibility

- [ ] Chrome (latest)
- [ ] Firefox (latest)
- [ ] Safari (latest)
- [ ] Edge (latest)
- [ ] Mobile Safari (iOS)
- [ ] Chrome Mobile (Android)

### Accessibility Testing

- [ ] Tab through page - map is focusable
- [ ] Use arrow keys - map pans
- [ ] Use +/- keys - map zooms
- [ ] Screen reader reads map label
- [ ] Screen reader reads InfoWindow content
- [ ] Color contrast meets WCAG AA
- [ ] No color-only information

---

## Security Considerations

### API Key Protection

**DO**:
- Store API key in environment variable: `GOOGLE_MAPS_API_KEY`
- Restrict API key to specific domains in Google Cloud Console
- Restrict API key to Maps JavaScript API only (not all Google APIs)
- Use HTTP referrer restrictions

**DON'T**:
- Hardcode API key in JavaScript
- Commit API key to git
- Use same API key for server-side and client-side

**Django Template Pattern**:
```html
<!-- Safe: API key from settings -->
<script src="https://maps.googleapis.com/maps/api/js?key={{ GOOGLE_MAPS_API_KEY }}&callback=initMap"></script>
```

```python
# settings.py
GOOGLE_MAPS_API_KEY = os.getenv('GOOGLE_MAPS_API_KEY')

# Context processor
def google_maps_api_key(request):
    return {'GOOGLE_MAPS_API_KEY': settings.GOOGLE_MAPS_API_KEY}
```

### XSS Prevention

**InfoWindow Content Sanitization**:
```javascript
// BAD: User input directly in InfoWindow (XSS vulnerability)
const infoWindowContent = `<h3>${userInput}</h3>`;

// GOOD: Sanitize in backend before rendering
// Django templates auto-escape by default
const siteName = mapElement.dataset.name; // Already escaped by Django
const infoWindowContent = `<h3 class="font-bold">${siteName}</h3>`;
```

**Django Template Auto-Escaping**:
```html
<!-- Django auto-escapes {{ site.name }} -->
<div id="map" data-name="{{ site.name }}"></div>
```

### HTTPS Requirement

Google Maps API requires HTTPS in production:
- Development: `http://localhost` is allowed
- Production: Must use `https://`

---

## Recommendations

### 1. Use Simple Script Tag Loading (Fast Track)

For Fast Track Mode, avoid complex async loaders. Use simple script tag:

```html
<script async defer
  src="https://maps.googleapis.com/maps/api/js?key={{ GOOGLE_MAPS_API_KEY }}&callback=initMap">
</script>
```

**Why**: Easier to debug, sufficient for single map, follows Fast Track philosophy.

### 2. Auto-Open InfoWindow on Load

Improve UX by opening InfoWindow automatically:

```javascript
setTimeout(() => {
  infoWindow.open({
    anchor: marker,
    map: map
  });
}, 500);
```

**Why**: User immediately sees site name and address without clicking.

### 3. Add Directions Link in InfoWindow

Make it easy to get directions:

```html
<a href="https://www.google.com/maps/dir/?api=1&destination=${lat},${lng}"
   target="_blank">
  Ver indicaciones →
</a>
```

**Why**: Common user action, reduces friction.

### 4. Use DaisyUI Card Wrapper

Consistent styling with rest of site:

```html
<div class="card bg-base-100 shadow-xl">
  <div class="card-body p-0">
    <div id="map"></div>
  </div>
</div>
```

**Why**: Matches design system, looks polished.

### 5. Validate Coordinates in Backend

Prevent errors by validating before rendering:

```python
if not site.location_lat or not site.location_lng:
    # Don't show map section
```

**Why**: Graceful degradation, better UX.

### 6. Skip Lazy Loading (For Now)

Eager load map for simplicity:

```html
<!-- Load immediately, not on scroll -->
<script src="..."></script>
```

**Why**: Fast Track Mode prioritizes simplicity. Optimize later if needed.

### 7. Use Data Attributes for Configuration

Store site data in HTML, not JavaScript:

```html
<div id="map"
     data-lat="{{ site.location_lat }}"
     data-lng="{{ site.location_lng }}"
     data-name="{{ site.name }}">
</div>
```

**Why**: Separates configuration from logic, easier to maintain.

### 8. Provide Fallback for No JavaScript

```html
<noscript>
  <div class="alert alert-warning">
    <p>Para ver el mapa, habilite JavaScript.</p>
    <a href="https://www.google.com/maps/search/?api=1&query=..." class="btn">
      Ver en Google Maps
    </a>
  </div>
</noscript>
```

**Why**: Accessibility, works for all users.

---

## Implementation Checklist

- [ ] Add `GOOGLE_MAPS_API_KEY` to `.env`
- [ ] Add API key to settings.py
- [ ] Create context processor for API key
- [ ] Add Google Maps script tag to base.html or home.html
- [ ] Create `static/js/google-maps.js`
- [ ] Add map section to home.html template
- [ ] Add DaisyUI styling to map container
- [ ] Add directions button
- [ ] Add error fallback UI
- [ ] Add `<noscript>` fallback
- [ ] Test on desktop
- [ ] Test on mobile
- [ ] Test keyboard navigation
- [ ] Test with screen reader
- [ ] Test with invalid coordinates
- [ ] Test with missing API key
- [ ] Restrict API key in Google Cloud Console
- [ ] Commit changes

---

## File Structure

```
ribera_calma/
├── static/
│   ├── js/
│   │   └── google-maps.js         # Map initialization logic
│   └── css/
│       └── custom.css              # Optional responsive overrides
├── templates/
│   ├── base.html                   # Include script tag (or in home.html)
│   ├── home.html                   # Map section
│   └── partials/
│       └── map_section.html        # Optional: Reusable map partial
└── apps/
    └── core/
        ├── context_processors.py   # GOOGLE_MAPS_API_KEY context
        └── views.py                # home_view with site data
```

---

## Open Questions

- [ ] Should map load eagerly or lazily? **Answer: Eagerly (Fast Track)**
- [ ] Should InfoWindow auto-open? **Answer: Yes (better UX)**
- [ ] Should we add custom map styles? **Answer: No (keep default for Fast Track)**
- [ ] Should we track map interactions in analytics? **Answer: Optional, implement later**
- [ ] Should we add multiple markers (e.g., nearby attractions)? **Answer: No (V2 feature)**

---

## Map Provider Comparison: Google Maps vs Leaflet

### Overview

This section compares two options for implementing the resort location map feature (RIBERA-018):
- **Option A**: Google Maps JavaScript API (proprietary, freemium)
- **Option B**: Leaflet.js + OpenStreetMap (open source, completely free)

**User requirement**:
- Display ONE marker at resort location
- Show site name and address in popup/InfoWindow
- Responsive design
- NO payment required for production use
- Must be 100% free for small resort website

---

### Google Maps JavaScript API - Free Tier Analysis

#### Pricing Changes (March 1, 2025)

**CRITICAL UPDATE**: Google changed their pricing model on March 1, 2025:

**Previous Model (Before March 1, 2025)**:
- $200 monthly credit
- Applied flexibly across all Maps Platform APIs
- Covered ~28,000 map loads per month ($0.007 per load)

**New Model (Starting March 1, 2025)**:
- **10,000 free API calls per month** per SKU (Essentials tier)
- Maps JavaScript API is in Essentials tier
- Each page load = 1 API call
- NO monthly credit - hard limit of 10,000 loads

**What This Means for RIBERA-018**:
```
Scenario: Small resort website

Assumptions:
- 500 unique visitors per month (realistic for small resort)
- Each visitor views home page 2 times on average
- Total page loads = 500 × 2 = 1,000 loads/month

Result: 1,000 loads < 10,000 free limit = WITHIN FREE TIER
```

**Risk Scenarios**:
1. **Marketing campaign**: Traffic spike to 5,000 visitors/month = 10,000 loads (at limit)
2. **Peak season**: Traffic doubles = 2,000 loads/month (still safe)
3. **Viral social media**: Traffic spike to 15,000 visitors = 30,000 loads = **EXCEEDS FREE TIER**

**What Happens When You Exceed Free Tier?**:
- Map stops working immediately (shows error message)
- Google requires credit card and billing account setup
- Charges $0.007 per additional map load
- Example: 20,000 loads = 10,000 free + 10,000 paid = $70/month

**Hidden Costs?**:
- Simple marker + InfoWindow: NO extra cost (included in map load)
- Street View: NO extra cost if already loaded
- Directions API: Separate charge (not used in our implementation)

**Conclusion**: Safe for small resort, but **risky if traffic grows** beyond 10,000 monthly visitors.

---

#### Google Maps Pros

1. **Familiar UX**: Users know Google Maps interface
2. **Street View**: Built-in, no extra work
3. **Accurate data**: Google's map data is highly detailed
4. **Mobile integration**: "Open in Google Maps" works seamlessly
5. **Simple implementation**: Well-documented, many tutorials
6. **Professional look**: Polished, modern design
7. **Free tier sufficient** for small sites (< 10,000 monthly visitors)

#### Google Maps Cons

1. **Hard usage limit**: 10,000 map loads per month (changed March 2025)
2. **Risk of billing**: If you exceed limit, map breaks unless you add payment
3. **No warning before limit**: Google doesn't warn you before hitting 10,000
4. **Proprietary**: Locked into Google ecosystem
5. **Terms can change**: Google changed pricing twice (2018, 2025)
6. **Requires API key**: Configuration overhead
7. **Requires HTTPS**: Cannot use on http:// in production

---

### Leaflet.js + OpenStreetMap - Free Tier Analysis

#### Pricing Model

**COMPLETELY FREE**:
- Leaflet.js: 100% open source (BSD 2-Clause License)
- OpenStreetMap tiles: Free for small to medium websites
- No API key required
- No usage limits (within fair use policy)
- No billing, ever

**OpenStreetMap Tile Usage Policy**:

**Fair Use Requirements**:
1. **Attribution**: Display "© OpenStreetMap contributors" on map
2. **User-Agent**: Send clear User-Agent header
3. **Caching**: Cache tiles for at least 7 days
4. **No bulk downloading**: No offline use

**Usage Limits**:
- **No specific numerical limit** published
- Policy states: "Funded by donations, limited capacity"
- Heavy use (distributing an app to thousands of users) requires permission
- Small website (500-5,000 visitors/month) is **acceptable fair use**

**What Happens If You Exceed Fair Use?**:
- OpenStreetMap Foundation may contact you
- You'd be asked to use alternative tile provider or self-host
- No immediate billing or map breakage
- Gradual communication, not sudden cutoff

**What This Means for RIBERA-018**:
```
Scenario: Small resort website (same assumptions)

Result: WITHIN FAIR USE
- Tile server handles thousands of small websites
- Proper caching reduces server load
- Attribution requirement is easy to meet
- No risk of sudden billing
```

**Alternative Tile Providers (If Needed)**:
- **MapTiler** (free tier: 100,000 tile requests/month)
- **Thunderforest** (free tier: 150,000 tile requests/month, requires registration)
- **Stamen** (free, no registration)
- **Self-hosted tiles** (advanced, not needed for small site)

---

#### Leaflet.js Pros

1. **100% free**: No usage limits, no billing, ever
2. **Open source**: Full control, no vendor lock-in
3. **Lightweight**: 42KB minified (vs Google Maps ~200KB+)
4. **No API key**: Simpler configuration
5. **No vendor lock-in**: Can switch tile providers anytime
6. **Works on HTTP**: No HTTPS requirement (though HTTPS recommended)
7. **Privacy-friendly**: No Google tracking
8. **Flexible**: Easy to customize, extend, style
9. **Active community**: Large ecosystem of plugins

#### Leaflet.js Cons

1. **Less polished**: Not as refined as Google Maps UI
2. **No Street View**: Would need separate integration
3. **Manual mobile integration**: "Open in Maps" requires custom link
4. **More code**: Slightly more setup than Google Maps
5. **Tile server dependency**: Relies on external tile providers
6. **Map data quality**: OpenStreetMap coverage varies by region (usually good in populated areas)
7. **Fair use policy**: Vague limits (but generous for small sites)

---

### Code Comparison

#### Google Maps Implementation

```html
<!-- HTML -->
<div id="map" class="w-full h-96 rounded-2xl"></div>

<script async defer
  src="https://maps.googleapis.com/maps/api/js?key=YOUR_API_KEY&callback=initMap">
</script>
```

```javascript
// JavaScript (static/js/google-maps.js)
function initMap() {
  const position = { lat: -34.6037, lng: -58.3816 };

  const map = new google.maps.Map(document.getElementById('map'), {
    center: position,
    zoom: 15
  });

  const marker = new google.maps.Marker({
    position: position,
    map: map,
    title: 'Ribera Calma'
  });

  const infoWindow = new google.maps.InfoWindow({
    content: '<h3>Ribera Calma</h3><p>Calle Falsa 123</p>'
  });

  marker.addListener('click', () => {
    infoWindow.open(map, marker);
  });
}
```

**Lines of code**: ~20 lines

---

#### Leaflet.js Implementation

```html
<!-- HTML -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<div id="map" class="w-full h-96 rounded-2xl"></div>
```

```javascript
// JavaScript (static/js/leaflet-map.js)
document.addEventListener('DOMContentLoaded', () => {
  const position = [-34.6037, -58.3816]; // Note: [lat, lng] order

  // Initialize map
  const map = L.map('map').setView(position, 15);

  // Add OpenStreetMap tile layer
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    maxZoom: 19
  }).addTo(map);

  // Add marker with popup
  const marker = L.marker(position).addTo(map);

  marker.bindPopup(
    '<h3 class="font-bold text-lg">Ribera Calma</h3>' +
    '<p class="text-sm">Calle Falsa 123</p>' +
    '<a href="https://www.openstreetmap.org/directions?from=&to=-34.6037,-58.3816" ' +
    'target="_blank" class="text-primary hover:underline">Ver indicaciones</a>'
  ).openPopup();
});
```

**Lines of code**: ~22 lines

**Code complexity**: Nearly identical to Google Maps.

---

### Performance Comparison

| Metric | Google Maps | Leaflet.js + OSM |
|--------|-------------|------------------|
| **Library size** | ~200KB | 42KB |
| **Initial load time** | ~500-800ms | ~200-400ms |
| **Tile loading** | Fast (Google CDN) | Fast (OSM CDN) |
| **Mobile performance** | Excellent | Excellent |
| **Caching** | Automatic | Manual (7 days min) |
| **Offline support** | Limited | Limited (with plugins) |

**Winner**: Leaflet.js (lighter, faster initial load)

---

### Accessibility Comparison

| Feature | Google Maps | Leaflet.js |
|---------|-------------|------------|
| **Keyboard navigation** | Yes (native) | Yes (native) |
| **Screen reader support** | Excellent | Good (with ARIA) |
| **ARIA labels** | Automatic | Manual |
| **Focus indicators** | Built-in | Built-in |
| **High contrast mode** | Supported | Supported |

**Winner**: Google Maps (slightly better out-of-box a11y)

---

### DaisyUI Integration

Both libraries integrate identically with DaisyUI:

```html
<!-- Same for both -->
<div class="card bg-base-100 shadow-xl">
  <div class="card-body p-0">
    <div id="map" class="w-full h-96 lg:h-[500px] rounded-2xl"></div>
  </div>
</div>
```

InfoWindow/Popup styling:

**Google Maps**:
```javascript
// Custom HTML in InfoWindow works
const content = '<div class="p-3"><h3 class="font-bold">Title</h3></div>';
```

**Leaflet.js**:
```javascript
// Custom HTML in popup works
marker.bindPopup('<div class="p-3"><h3 class="font-bold">Title</h3></div>');
```

**Winner**: Tie (both support custom HTML styling)

---

### Security Comparison

| Aspect | Google Maps | Leaflet.js |
|--------|-------------|------------|
| **API key exposure** | Required (in HTML) | Not required |
| **XSS risk** | Low (content sanitized) | Low (content sanitized) |
| **Third-party tracking** | Yes (Google Analytics) | No |
| **Privacy** | Google collects data | No tracking |
| **HTTPS requirement** | Yes (production) | No (but recommended) |

**Winner**: Leaflet.js (better privacy, no API key)

---

### Migration Path

**If you start with Google Maps and need to switch to Leaflet**:

1. Remove Google Maps script tag
2. Add Leaflet CSS + JS
3. Update `initMap()` function (~10 lines change)
4. Update marker/popup code (~5 lines change)
5. Update directions link URL

**Effort**: 30-60 minutes

**If you start with Leaflet and need to switch to Google Maps**:

1. Add Google Maps API key to settings
2. Replace Leaflet script tags
3. Update `initMap()` function (~10 lines change)
4. Update marker/InfoWindow code (~5 lines change)
5. Update directions link URL

**Effort**: 30-60 minutes

**Conclusion**: Migration is straightforward in both directions.

---

### Cost Comparison Table

| Scenario | Visitors/Month | Page Views | Google Maps Cost | Leaflet.js Cost |
|----------|----------------|------------|------------------|-----------------|
| **Small site** | 500 | 1,000 | $0 (free tier) | $0 (free) |
| **Growing site** | 2,500 | 5,000 | $0 (free tier) | $0 (free) |
| **Popular site** | 5,000 | 10,000 | $0 (at limit) | $0 (free) |
| **Peak season** | 7,500 | 15,000 | **$35/month** | $0 (free) |
| **Viral traffic** | 15,000 | 30,000 | **$140/month** | $0 (free) |
| **High traffic** | 30,000 | 60,000 | **$350/month** | $0 (free) |

**Formula**: Google Maps = (page_views - 10,000) × $0.007 (if > 10,000)

**Conclusion**: Leaflet.js = $0 forever, Google Maps = $0 until you exceed 10,000 monthly visitors.

---

### Recommendation

**For RIBERA-018 (Resort Location Map):**

**RECOMMENDED: Leaflet.js + OpenStreetMap**

**Reasons**:

1. **Zero cost risk**: No billing, ever. No surprises if traffic grows.
2. **Sufficient for requirement**: Single marker + popup is simple in Leaflet.
3. **Fast Track compatible**: Code complexity nearly identical to Google Maps.
4. **Privacy-friendly**: No Google tracking, better for users.
5. **No API key hassle**: One less configuration step.
6. **Open source**: Full control, no vendor lock-in.
7. **Future-proof**: OpenStreetMap has existed since 2004, will outlive commercial services.

**When to Use Google Maps Instead**:

1. **You need Street View**: Built-in with Google Maps, not available in Leaflet (without plugins).
2. **You want polished UI**: Google Maps looks slightly more professional.
3. **Traffic guaranteed under 10,000/month**: You're certain you won't exceed free tier.
4. **You're already using Google services**: Easier to add to existing Google Cloud project.

**For this project**: Ribera Calma is a **small resort website**. Traffic unlikely to exceed 10,000 monthly visitors initially, but if the site becomes popular (marketing campaigns, viral social media, peak season), Google Maps will start charging. Leaflet.js eliminates this risk entirely.

---

### Implementation Recommendation

**Phase 1 (MVP - Fast Track)**:
- Use **Leaflet.js + OpenStreetMap**
- Simple marker + popup
- Directions link to OpenStreetMap
- Total implementation time: 1-2 hours

**Phase 2 (Optional enhancements)**:
- Add custom marker icon (resort logo)
- Add multiple markers (nearby attractions)
- Add route visualization
- Switch tile provider if needed (MapTiler, Stamen, etc.)

**If you decide to use Google Maps later**:
- Migration path is straightforward (30-60 minutes)
- Keep Leaflet.js code as fallback

---

### Example: Complete Leaflet.js Implementation

```html
<!-- templates/home.html -->
{% extends "base.html" %}

{% block extra_css %}
<!-- Leaflet CSS -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
      integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY="
      crossorigin=""/>
{% endblock %}

{% block content %}
<!-- Map Section -->
<section class="py-12 bg-base-200" id="location-section">
  <div class="container mx-auto px-4">
    <h2 class="text-3xl font-bold text-center mb-8">¿Cómo llegar?</h2>

    <!-- Map Container with DaisyUI styling -->
    <div class="card bg-base-100 shadow-xl">
      <div class="card-body p-0">
        <!-- Map element -->
        <div id="map"
             class="w-full h-96 lg:h-[500px] rounded-2xl"
             role="application"
             aria-label="Mapa de ubicación de {{ site.name }}"
             data-lat="{{ site.location_lat }}"
             data-lng="{{ site.location_lng }}"
             data-name="{{ site.name }}"
             data-address="{{ site.address }}">
        </div>
      </div>
    </div>

    <!-- Directions Link -->
    <div class="text-center mt-6">
      <a href="https://www.openstreetmap.org/directions?from=&to={{ site.location_lat }},{{ site.location_lng }}"
         target="_blank"
         rel="noopener noreferrer"
         class="btn btn-primary btn-lg">
        <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                d="M9 20l-5.447-2.724A1 1 0 013 16.382V5.618a1 1 0 011.447-.894L9 7m0 13l6-3m-6 3V7m6 10l4.553 2.276A1 1 0 0021 18.382V7.618a1 1 0 00-.553-.894L15 4m0 13V4m0 0L9 7"/>
        </svg>
        Cómo llegar
      </a>
    </div>
  </div>
</section>

<!-- Error fallback -->
<noscript>
  <div class="container mx-auto px-4 mt-4">
    <div class="alert alert-warning">
      <p>Para ver el mapa interactivo, por favor habilite JavaScript.</p>
      <p><strong>Ubicación:</strong> {{ site.address }}</p>
    </div>
  </div>
</noscript>
{% endblock %}

{% block extra_js %}
<!-- Leaflet JS -->
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        integrity="sha256-20nQCchB9co0qIjJZRGuk2/Z9VM+kNiyxNV1lvTlZBo="
        crossorigin=""></script>

<!-- Map initialization -->
<script src="{% static 'js/leaflet-map.js' %}"></script>
{% endblock %}
```

```javascript
// static/js/leaflet-map.js

/**
 * Leaflet.js + OpenStreetMap integration for resort location
 * 100% free, no API key required
 *
 * @requires Leaflet.js
 */

let map;
let marker;

/**
 * Initialize Leaflet map with OpenStreetMap tiles
 * Called on DOMContentLoaded
 *
 * @returns {void}
 */
function initMap() {
  const mapElement = document.getElementById('map');

  // Guard: Map element must exist
  if (!mapElement) {
    console.error('Map element not found');
    return;
  }

  // Extract site data from data attributes
  const lat = parseFloat(mapElement.dataset.lat);
  const lng = parseFloat(mapElement.dataset.lng);
  const siteName = mapElement.dataset.name;
  const siteAddress = mapElement.dataset.address;

  // Validate coordinates
  if (isNaN(lat) || isNaN(lng)) {
    showMapError('Coordenadas inválidas');
    return;
  }

  // Validate coordinate ranges
  if (lat < -90 || lat > 90 || lng < -180 || lng > 180) {
    showMapError('Coordenadas fuera de rango');
    return;
  }

  const position = [lat, lng]; // Leaflet uses [lat, lng] order

  try {
    // Initialize map
    map = L.map('map', {
      center: position,
      zoom: 15,
      zoomControl: true,
      scrollWheelZoom: true
    });

    // Add OpenStreetMap tile layer
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
      maxZoom: 19,
      minZoom: 3
    }).addTo(map);

    // Create marker
    marker = L.marker(position, {
      title: siteName,
      alt: siteName
    }).addTo(map);

    // Create popup content with DaisyUI styling
    const popupContent = `
      <div class="p-3">
        <h3 class="font-bold text-lg mb-2">${siteName}</h3>
        <p class="text-sm text-gray-600 mb-3">${siteAddress}</p>
        <a href="https://www.openstreetmap.org/directions?from=&to=${lat},${lng}"
           target="_blank"
           rel="noopener noreferrer"
           class="text-primary hover:underline text-sm font-semibold">
          Ver indicaciones →
        </a>
      </div>
    `;

    // Bind popup to marker
    marker.bindPopup(popupContent, {
      maxWidth: 300,
      closeButton: true,
      autoClose: false
    });

    // Auto-open popup on load (better UX)
    setTimeout(() => {
      marker.openPopup();
    }, 500);

  } catch (error) {
    console.error('Map initialization error:', error);
    showMapError('Error al cargar el mapa');
  }
}

/**
 * Show map error message to user
 *
 * @param {string} message - Error message to display
 * @returns {void}
 */
function showMapError(message) {
  const mapElement = document.getElementById('map');

  if (mapElement) {
    mapElement.innerHTML = `
      <div class="flex items-center justify-center h-full bg-base-200 rounded-2xl">
        <div class="alert alert-error max-w-md">
          <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
          </svg>
          <span>${message}</span>
        </div>
      </div>
    `;
  }
}

// Initialize map when DOM is ready
document.addEventListener('DOMContentLoaded', initMap);
```

**Total lines of code**: ~120 lines (HTML + JS)
**Implementation time**: 1-2 hours
**Cost**: $0 forever

---

### Alternative Tile Providers (If Needed)

If you want different map styles or need to switch from OpenStreetMap:

#### 1. Stamen Terrain (Free, No Registration)

```javascript
L.tileLayer('https://stamen-tiles-{s}.a.ssl.fastly.net/terrain/{z}/{x}/{y}.jpg', {
  attribution: 'Map tiles by <a href="http://stamen.com">Stamen Design</a>, under <a href="http://creativecommons.org/licenses/by/3.0">CC BY 3.0</a>',
  maxZoom: 18
}).addTo(map);
```

#### 2. MapTiler (Free Tier: 100,000 tiles/month)

```javascript
// Requires registration for API key
L.tileLayer('https://api.maptiler.com/maps/streets/{z}/{x}/{y}.png?key=YOUR_API_KEY', {
  attribution: '<a href="https://www.maptiler.com/copyright/">MapTiler</a>',
  maxZoom: 18
}).addTo(map);
```

#### 3. Thunderforest (Free Tier: 150,000 tiles/month)

```javascript
// Requires registration for API key
L.tileLayer('https://{s}.tile.thunderforest.com/cycle/{z}/{x}/{y}.png?apikey=YOUR_API_KEY', {
  attribution: 'Maps © <a href="http://www.thunderforest.com">Thunderforest</a>',
  maxZoom: 22
}).addTo(map);
```

**Recommendation**: Start with OpenStreetMap (no API key). Switch to MapTiler or Thunderforest only if you need custom styling or exceed fair use.

---

### Summary Table

| Criteria | Google Maps | Leaflet.js + OSM | Winner |
|----------|-------------|------------------|--------|
| **Cost (small site)** | $0 | $0 | Tie |
| **Cost (high traffic)** | $35-350/month | $0 | Leaflet |
| **Risk of billing** | Yes (> 10k visitors) | No | Leaflet |
| **Code complexity** | Simple | Simple | Tie |
| **Library size** | 200KB | 42KB | Leaflet |
| **API key required** | Yes | No | Leaflet |
| **Privacy** | Google tracks | No tracking | Leaflet |
| **Street View** | Yes | No | Google Maps |
| **UI polish** | Excellent | Good | Google Maps |
| **Open source** | No | Yes | Leaflet |
| **Accessibility** | Excellent | Good | Google Maps |
| **Mobile integration** | Native | Manual | Google Maps |
| **Vendor lock-in** | Yes | No | Leaflet |
| **Future-proof** | Risky (pricing changes) | Safe | Leaflet |

**Overall winner for RIBERA-018**: **Leaflet.js + OpenStreetMap**

**Score**: Leaflet 8, Google Maps 4, Tie 2

---

## References

- [Google Maps JavaScript API Docs](https://developers.google.com/maps/documentation/javascript)
- [Google Maps Pricing (2025)](https://mapsplatform.google.com/pricing/)
- [Leaflet.js Official Site](https://leafletjs.com/)
- [Leaflet Quick Start Guide](https://leafletjs.com/examples/quick-start/)
- [OpenStreetMap Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/)
- [Leaflet Providers Demo](https://leaflet-extras.github.io/leaflet-providers/preview/)
- [DaisyUI Card Component](https://daisyui.com/components/card/)
- [DaisyUI Button Component](https://daisyui.com/components/button/)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
