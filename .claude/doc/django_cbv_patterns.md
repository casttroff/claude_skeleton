# Django CBV Patterns Library

## Standard Patterns for Django Projects

This document provides copy-paste CBV patterns for common scenarios in multi-tenant Django applications.

---

## 1. List View (Multi-tenant)

**Use when:** Displaying a list of objects filtered by site

```python
from django.views.generic import ListView
from django.shortcuts import get_object_or_404

class AccommodationListView(ListView):
    """List accommodations for site.

    Standard pattern:
    - Multi-tenant filtering in get_queryset
    - Site added to context in get_context_data
    - Template name explicit
    - Pagination enabled
    """
    model = AccommodationType
    template_name = "accommodations/list.html"
    context_object_name = "accommodations"
    paginate_by = 12

    def get_queryset(self):
        """Filter by site and active status."""
        site = get_object_or_404(Site, slug=self.kwargs["site_slug"], is_active=True)
        return AccommodationType.objects.filter(
            site=site,
            is_active=True
        ).select_related("site").order_by("order", "name")

    def get_context_data(self, **kwargs):
        """Add site to context."""
        context = super().get_context_data(**kwargs)
        context["site"] = get_object_or_404(Site, slug=self.kwargs["site_slug"])
        return context
```

**URLs:**

```python
path("<slug:site_slug>/accommodations/", AccommodationListView.as_view(), name="list")
```

---

## 2. Detail View (Multi-tenant)

**Use when:** Displaying single object detail

```python
from django.views.generic import DetailView
from django.shortcuts import get_object_or_404

class AccommodationDetailView(DetailView):
    """Show accommodation detail.

    Standard pattern:
    - Filters by site in get_queryset (security)
    - Uses slug for lookup
    - Site added to context
    """
    model = AccommodationType
    template_name = "accommodations/detail.html"
    context_object_name = "accommodation"

    def get_queryset(self):
        """Filter by site for security."""
        site = get_object_or_404(Site, slug=self.kwargs["site_slug"], is_active=True)
        return AccommodationType.objects.filter(site=site).select_related("site")

    def get_context_data(self, **kwargs):
        """Add site to context."""
        context = super().get_context_data(**kwargs)
        context["site"] = self.object.site
        return context
```

**URLs:**

```python
path("<slug:site_slug>/accommodations/<slug:slug>/", AccommodationDetailView.as_view(), name="detail")
```

---

## 3. Create View (HTMX-enabled)

**Use when:** Creating new objects with forms

```python
from django.views.generic.edit import CreateView
from django.http import HttpResponse
from django.template.loader import render_to_string
from django.shortcuts import get_object_or_404, redirect

class AccommodationCreateView(CreateView):
    """Create accommodation with HTMX support.

    Standard pattern:
    - form_valid returns HTMX response or redirect
    - form_invalid returns partial with errors
    - Site added to form kwargs
    """
    model = AccommodationType
    form_class = AccommodationForm
    template_name = "accommodations/form.html"

    def get_form_kwargs(self):
        """Add site to form kwargs."""
        kwargs = super().get_form_kwargs()
        kwargs["site"] = get_object_or_404(Site, slug=self.kwargs["site_slug"])
        return kwargs

    def form_valid(self, form):
        """Handle successful submission (HTMX-aware)."""
        accommodation = form.save()

        if self.request.headers.get("HX-Request"):
            # HTMX partial response
            response = HttpResponse()
            response["HX-Trigger"] = "accommodationCreated"
            return response

        # Regular form submission
        return redirect("accommodations:detail",
                       site_slug=accommodation.site.slug,
                       slug=accommodation.slug)

    def form_invalid(self, form):
        """Handle validation errors (HTMX-aware)."""
        if self.request.headers.get("HX-Request"):
            html = render_to_string(
                "accommodations/partials/form_errors.html",
                {"form": form},
                request=self.request
            )
            response = HttpResponse(html)
            response["HX-Retarget"] = "#form-errors"
            return response

        return super().form_invalid(form)
```

**URLs:**

```python
path("<slug:site_slug>/accommodations/create/", AccommodationCreateView.as_view(), name="create")
```

---

## 4. Update View (HTMX-enabled)

**Use when:** Editing existing objects

```python
from django.views.generic.edit import UpdateView
from django.http import HttpResponse
from django.template.loader import render_to_string
from django.shortcuts import get_object_or_404, redirect
from django.core.exceptions import PermissionDenied

class AccommodationUpdateView(UpdateView):
    """Update accommodation with HTMX support.

    Standard pattern:
    - Permission check in get_object
    - HTMX-aware responses
    """
    model = AccommodationType
    form_class = AccommodationForm
    template_name = "accommodations/form.html"

    def get_object(self, queryset=None):
        """Get object and check permissions."""
        obj = super().get_object(queryset)

        # Multi-tenant security check
        if not (obj.site.owner == self.request.user or
                self.request.user in obj.site.admins.all()):
            raise PermissionDenied("You don't have permission to edit this accommodation.")

        return obj

    def form_valid(self, form):
        """Save and return success response for HTMX."""
        self.object = form.save()

        if self.request.headers.get("HX-Request"):
            response = HttpResponse()
            response["HX-Trigger"] = "accommodationUpdated"
            return response

        return redirect("accommodations:detail",
                       site_slug=self.object.site.slug,
                       slug=self.object.slug)

    def form_invalid(self, form):
        """Return form with errors for HTMX."""
        if self.request.headers.get("HX-Request"):
            html = render_to_string(
                self.template_name,
                {"form": form, "accommodation": self.object, "mode": "edit"},
                request=self.request
            )
            response = HttpResponse(html)
            response["HX-Retarget"] = "#modal-content"
            return response

        return super().form_invalid(form)
```

**URLs:**

```python
path("<slug:site_slug>/accommodations/<slug:slug>/edit/", AccommodationUpdateView.as_view(), name="edit")
```

---

## 5. Delete View

**Use when:** Deleting objects with confirmation

```python
from django.views.generic.edit import DeleteView
from django.urls import reverse_lazy
from django.http import HttpResponse
from django.core.exceptions import PermissionDenied

class AccommodationDeleteView(DeleteView):
    """Delete accommodation with confirmation.

    Standard pattern:
    - Permission check in get_object
    - Success URL uses reverse_lazy
    - HTMX-aware delete response
    """
    model = AccommodationType
    template_name = "accommodations/confirm_delete.html"
    success_url = reverse_lazy("accommodations:admin-list")

    def get_object(self, queryset=None):
        """Get object and check permissions."""
        obj = super().get_object(queryset)

        if not (obj.site.owner == self.request.user or
                self.request.user in obj.site.admins.all()):
            raise PermissionDenied("You don't have permission to delete this accommodation.")

        return obj

    def delete(self, request, *args, **kwargs):
        """Handle HTMX delete response."""
        response = super().delete(request, *args, **kwargs)

        if request.headers.get("HX-Request"):
            return HttpResponse("")  # Empty response, HTMX handles removal

        return response
```

**URLs:**

```python
path("accommodations/<int:pk>/delete/", AccommodationDeleteView.as_view(), name="delete")
```

---

## 6. HTMX Partial Rendering

**Use when:** Rendering partial HTML via HTMX (filters, dynamic content)

```python
from django.views import View
from django.http import HttpResponse
from django.template.loader import render_to_string
from django.shortcuts import get_object_or_404

class AccommodationFilterView(View):
    """Filter accommodations via HTMX.

    Standard pattern:
    - Single responsibility (render partial)
    - GET only
    - Returns HTML fragment
    """

    def get(self, request, site_slug):
        """Render filtered accommodations."""
        site = get_object_or_404(Site, slug=site_slug, is_active=True)

        # Start with base queryset
        accommodations = AccommodationType.objects.filter(site=site, is_active=True)

        # Apply filters
        capacity = request.GET.get("capacity")
        if capacity:
            try:
                accommodations = accommodations.filter(capacity_max__gte=int(capacity))
            except ValueError:
                pass

        price_min = request.GET.get("price_min")
        if price_min:
            try:
                accommodations = accommodations.filter(price_per_night__gte=Decimal(price_min))
            except (ValueError, TypeError):
                pass

        # Render partial
        html = render_to_string(
            "accommodations/partials/grid.html",
            {"accommodations": accommodations, "site": site},
            request=request
        )

        return HttpResponse(html)
```

**URLs:**

```python
path("<slug:site_slug>/accommodations/filter/", AccommodationFilterView.as_view(), name="filter")
```

---

## 7. Simple Template View

**Use when:** Rendering static pages with minimal context

```python
from django.views.generic import TemplateView
from django.shortcuts import get_object_or_404

class AboutPageView(TemplateView):
    """Static about page.

    Standard pattern:
    - Use TemplateView for static content
    - Add site to context
    """
    template_name = "pages/about.html"

    def get_context_data(self, **kwargs):
        """Add site to context."""
        context = super().get_context_data(**kwargs)
        context["site"] = get_object_or_404(Site, slug=self.kwargs["site_slug"])
        return context
```

**URLs:**

```python
path("<slug:site_slug>/about/", AboutPageView.as_view(), name="about")
```

---

## 8. API Endpoint (JSON Response)

**Use when:** Returning JSON data for HTMX or JavaScript

```python
from django.views import View
from django.http import JsonResponse
from django.shortcuts import get_object_or_404
from datetime import datetime, timedelta

class BlockedDatesAPIView(View):
    """Return blocked dates as JSON for calendar.

    Standard pattern:
    - GET only (use @method_decorator if needed)
    - Returns JsonResponse
    """

    def get(self, request, site_slug, accommodation_id):
        """Get blocked dates for accommodation."""
        site = get_object_or_404(Site, slug=site_slug, is_active=True)
        accommodation = get_object_or_404(AccommodationType, site=site, pk=accommodation_id)

        # Calculate date range (next 6 months)
        start_date = datetime.now().date()
        end_date = start_date + timedelta(days=180)

        # Get blocked dates
        blocked_dates = Reservation.get_blocked_dates(accommodation, start_date, end_date)

        return JsonResponse({
            "blocked_dates": [d.isoformat() for d in blocked_dates],
            "accommodation_id": accommodation_id
        })
```

**URLs:**

```python
path("api/<slug:site_slug>/accommodations/<int:accommodation_id>/blocked-dates/",
     BlockedDatesAPIView.as_view(), name="blocked-dates")
```

---

## Testing CBVs

### Standard Test Pattern

```python
from django.test import TestCase, Client
from django.contrib.auth import get_user_model
from apps.core.models import Site
from apps.accommodations.models import AccommodationType

User = get_user_model()

class AccommodationListViewTest(TestCase):
    def setUp(self):
        """Create test data."""
        self.client = Client()

        self.user = User.objects.create_user(
            username="testuser",
            email="test@example.com",
            password="password123"
        )

        self.site = Site.objects.create(
            name="Test Site",
            slug="test-site",
            owner=self.user,
            phone="123456",
            email="site@example.com",
            address="Test Address",
            location_lat=-34.603722,
            location_lng=-58.381592
        )

        self.accommodation = AccommodationType.objects.create(
            site=self.site,
            name="Test Cabin",
            slug="test-cabin",
            description="Test description",
            price_per_night=100
        )

    def test_list_view_shows_accommodations(self):
        """List view should show site's accommodations."""
        response = self.client.get(f"/{self.site.slug}/accommodations/")

        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Test Cabin")
        self.assertEqual(response.context["site"], self.site)

    def test_multi_tenant_isolation(self):
        """Should not show other sites' accommodations."""
        other_site = Site.objects.create(
            name="Other Site",
            slug="other-site",
            owner=self.user,
            phone="789012",
            email="other@example.com",
            address="Other Address",
            location_lat=-31.420083,
            location_lng=-64.188776
        )

        other_accommodation = AccommodationType.objects.create(
            site=other_site,
            name="Other Cabin",
            slug="other-cabin",
            description="Other description",
            price_per_night=150
        )

        response = self.client.get(f"/{self.site.slug}/accommodations/")

        self.assertContains(response, "Test Cabin")
        self.assertNotContains(response, "Other Cabin")
```

---

## Migration Guide: FBV to CBV

### Example: Migrating accommodation_list()

**Before (FBV):**

```python
def accommodation_list(request, site_slug):
    """List all active accommodations for a site."""
    site = get_object_or_404(Site, slug=site_slug, is_active=True)

    accommodations = AccommodationType.objects.filter(
        site=site, is_active=True
    ).order_by("order", "name")

    context = {
        "site": site,
        "accommodations": accommodations,
        "current_year": datetime.now().year,
    }

    return render(request, "accommodations/list.html", context)
```

**After (CBV):**

```python
class AccommodationListView(ListView):
    """List all active accommodations for a site."""
    model = AccommodationType
    template_name = "accommodations/list.html"
    context_object_name = "accommodations"

    def get_queryset(self):
        """Filter by site and active status."""
        site = get_object_or_404(Site, slug=self.kwargs["site_slug"], is_active=True)
        return AccommodationType.objects.filter(
            site=site,
            is_active=True
        ).select_related("site").order_by("order", "name")

    def get_context_data(self, **kwargs):
        """Add site and current year to context."""
        context = super().get_context_data(**kwargs)
        context["site"] = get_object_or_404(Site, slug=self.kwargs["site_slug"])
        context["current_year"] = datetime.now().year
        return context
```

**URLs change:**

```python
# Before
path("<slug:site_slug>/accommodations/", accommodation_list, name="list")

# After
path("<slug:site_slug>/accommodations/", AccommodationListView.as_view(), name="list")
```

**Benefits:**
1. Each method <60 lines (Rule 4 compliance)
2. Clear separation: queryset logic vs context logic
3. Easy to extend (add pagination, filtering)
4. Testable (mock get_queryset independently)

---

## Common Pitfalls

### 1. Forgetting `.as_view()` in URLs

```python
# ❌ WRONG
path("accommodations/", AccommodationListView, name="list")

# ✅ CORRECT
path("accommodations/", AccommodationListView.as_view(), name="list")
```

### 2. Not Using `self.kwargs` for URL Parameters

```python
# ❌ WRONG
def get_queryset(self):
    site_slug = self.request.GET.get("site_slug")  # URL params not in GET

# ✅ CORRECT
def get_queryset(self):
    site_slug = self.kwargs["site_slug"]  # URL params in self.kwargs
```

### 3. Calling `super()` Without `**kwargs`

```python
# ❌ WRONG
def get_context_data(self):
    context = super().get_context_data()  # Missing **kwargs

# ✅ CORRECT
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
```

---

## Summary

**Default to CBVs for:**
- Consistency across codebase
- Rule 4 compliance (60-line limit per method)
- Maintainability (clear structure)
- Testability (test each method independently)

**Use FBV only for:**
- Very simple views (<30 lines, no logic)
- One-off redirects or static pages

**Remember:** CBVs are Django's built-in pattern, NOT a custom abstraction layer (Rule 8 compliant).
