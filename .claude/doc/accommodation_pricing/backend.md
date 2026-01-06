# Accommodation Pricing System - Backend Architecture

## Overview

The dynamic pricing system allows configuring per-person discounts for accommodations based on occupancy. This enables incentivizing partial occupancy (e.g., 2 people in a 6-person cabin pays less per night than 6 people).

**Key Principles:**
- Auto-create pricing records when Accommodation created/updated
- Capacity-based discount system (percentage or fixed amount)
- Base price always at capacity_max (no discount)
- Multi-tenant isolation through Accommodation FK
- Type-safe price calculation

---

## Model Design

### AccommodationPricesPerPerson

**Location:** `apps/accommodations/models.py`

```python
from django.db import models
from django.core.exceptions import ValidationError
from decimal import Decimal
from typing import Optional


class AccommodationPricesPerPerson(models.Model):
    """
    Per-person pricing configuration for accommodations.

    Allows defining discounts based on number of guests. The record with
    capacity == accommodation.capacity_max always has discount=0 (base price).

    Multi-tenant isolation: Through accommodation FK to AccommodationType (which has Site FK)
    """

    DISCOUNT_MODE_CHOICES = [
        ('%', 'Porcentaje'),
        ('Fijo', 'Monto Fijo'),
    ]

    accommodation = models.ForeignKey(
        'AccommodationType',
        on_delete=models.CASCADE,
        related_name='pricing_records',
        help_text="Accommodation type this pricing applies to"
    )

    capacity = models.PositiveIntegerField(
        help_text="Number of guests for this pricing tier (1 to capacity_max)"
    )

    discount = models.PositiveIntegerField(
        null=True,
        blank=True,
        help_text="Discount amount (percentage or fixed, depends on mode). 0 for base price."
    )

    mode = models.CharField(
        max_length=10,
        choices=DISCOUNT_MODE_CHOICES,
        default='%',
        help_text="Discount calculation mode: '%' for percentage, 'Fijo' for fixed amount"
    )

    is_active = models.BooleanField(
        default=True,
        help_text="Whether this pricing tier is currently active"
    )

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['accommodation', 'capacity']
        verbose_name = "Precio por Persona"
        verbose_name_plural = "Precios por Persona"
        indexes = [
            models.Index(fields=['accommodation', 'capacity']),
            models.Index(fields=['accommodation', 'is_active']),
        ]
        unique_together = [['accommodation', 'capacity']]

    def __str__(self) -> str:
        discount_display = f"{self.discount}{self.mode}" if self.discount else "Sin descuento"
        return f"{self.accommodation.name} - {self.capacity} personas ({discount_display})"

    def clean(self) -> None:
        """
        Validate pricing record business rules.

        Rules:
        1. If is_active=True, discount cannot be null
        2. Cannot modify discount if capacity == accommodation.capacity_max (base price)
        3. Capacity must be within accommodation's capacity_min to capacity_max range
        4. Discount cannot exceed 100 if mode is '%'
        5. Discount cannot exceed price_per_night if mode is 'Fijo'

        Raises:
            ValidationError: If any validation rule fails
        """
        super().clean()

        # Rule 1: Active records must have discount defined
        if self.is_active and self.discount is None:
            raise ValidationError({
                'discount': 'El descuento no puede estar vacío si el precio está activo.'
            })

        # Rule 2: Cannot modify discount for base price tier
        if self.accommodation_id and self.capacity == self.accommodation.capacity_max:
            if self.discount != 0:
                raise ValidationError({
                    'discount': f'El precio base (capacidad máxima {self.accommodation.capacity_max}) '
                                f'debe tener descuento 0. No se puede modificar.'
                })

        # Rule 3: Capacity within accommodation range
        if self.accommodation_id:
            if not (self.accommodation.capacity_min <= self.capacity <= self.accommodation.capacity_max):
                raise ValidationError({
                    'capacity': f'La capacidad debe estar entre {self.accommodation.capacity_min} '
                                f'y {self.accommodation.capacity_max} personas.'
                })

        # Rule 4: Percentage discount validation
        if self.mode == '%' and self.discount is not None:
            if self.discount > 100:
                raise ValidationError({
                    'discount': 'El descuento porcentual no puede ser mayor a 100%.'
                })

        # Rule 5: Fixed discount validation
        if self.mode == 'Fijo' and self.discount is not None and self.accommodation_id:
            if Decimal(self.discount) > self.accommodation.price_per_night:
                raise ValidationError({
                    'discount': f'El descuento fijo no puede superar el precio por noche '
                                f'(${self.accommodation.price_per_night}).'
                })

    def save(self, *args, **kwargs) -> None:
        """Save with validation."""
        self.full_clean()
        super().save(*args, **kwargs)

    def calculate_final_price(self) -> Decimal:
        """
        Calculate final price per night for this capacity tier.

        Formula:
        - mode=='%': final_price = price_per_night - ((price_per_night * discount) / 100)
        - mode=='Fijo': final_price = price_per_night - discount

        Returns:
            Decimal: Final price per night after discount

        Raises:
            ValueError: If discount is null for active record
        """
        if self.is_active and self.discount is None:
            raise ValueError(
                f"Cannot calculate price: discount is null for active record "
                f"(accommodation={self.accommodation_id}, capacity={self.capacity})"
            )

        base_price = self.accommodation.price_per_night

        # No discount case (base price tier or inactive)
        if self.discount == 0 or not self.is_active:
            return base_price

        if self.mode == '%':
            discount_amount = (base_price * Decimal(self.discount)) / Decimal(100)
            return base_price - discount_amount

        elif self.mode == 'Fijo':
            return base_price - Decimal(self.discount)

        # Should never reach here due to choices constraint
        return base_price
```

---

## Auto-creation Logic: Signal vs save()

### Decision: Django Signal (post_save on AccommodationType)

**Chosen Approach:** Use `post_save` signal on `AccommodationType`

**Rationale:**

**Pros of Signal Approach:**
- ✅ Separation of concerns (AccommodationType doesn't know about pricing records)
- ✅ Easier to test in isolation (signal can be mocked)
- ✅ Cleaner AccommodationType model (no pricing logic in save())
- ✅ Can trigger from ANY location (admin, API, management command, etc.)
- ✅ Django best practice for cross-model side effects

**Cons of save() Override:**
- ❌ Tight coupling (pricing logic inside AccommodationType)
- ❌ Harder to test (must instantiate full model)
- ❌ Mixes concerns (model save + pricing management)

**Implementation:**

**Location:** `apps/accommodations/signals.py`

```python
from django.db.models.signals import post_save
from django.dispatch import receiver
from apps.accommodations.models import AccommodationType, AccommodationPricesPerPerson
import logging

logger = logging.getLogger(__name__)


@receiver(post_save, sender=AccommodationType)
def manage_pricing_records(sender, instance: AccommodationType, created: bool, **kwargs):
    """
    Auto-create/update pricing records when AccommodationType is saved.

    Triggered on EVERY save (create or update).

    Actions:
    1. Create pricing records for capacity_min to capacity_max (if not exist)
    2. Set capacity_max record discount=0 (base price)
    3. Delete records outside current capacity range

    Args:
        sender: Model class (AccommodationType)
        instance: The saved AccommodationType instance
        created: True if new record, False if update
        **kwargs: Additional signal arguments
    """
    from apps.accommodations.managers import AccommodationPricingManager

    # Use manager method for business logic
    manager = AccommodationPricingManager(instance)

    # Create missing pricing records
    created_count = manager.create_pricing_records()

    # Cleanup records outside capacity range
    deleted_count = manager.cleanup_pricing_records()

    logger.info(
        f"Pricing records for {instance.name} (site={instance.site.slug}): "
        f"created={created_count}, deleted={deleted_count}"
    )
```

**Register signal in apps.py:**

```python
# apps/accommodations/apps.py

from django.apps import AppConfig


class AccommodationsConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.accommodations'

    def ready(self):
        """Import signals when app is ready."""
        import apps.accommodations.signals  # noqa
```

---

## Manager/QuerySet Design

### AccommodationPricingManager

**Location:** `apps/accommodations/managers.py`

```python
from django.db import models, transaction
from decimal import Decimal
from typing import Tuple, Optional
import logging

logger = logging.getLogger(__name__)


class AccommodationPricingManager:
    """
    Manager for AccommodationPricesPerPerson business logic.

    Handles:
    - Auto-creation of pricing records
    - Cascade cleanup on capacity changes
    - Price calculation for reservations
    """

    def __init__(self, accommodation: 'AccommodationType'):
        """
        Initialize manager for specific accommodation.

        Args:
            accommodation: AccommodationType instance to manage pricing for
        """
        self.accommodation = accommodation

    @transaction.atomic
    def create_pricing_records(self) -> int:
        """
        Create pricing records for capacity_min to capacity_max.

        Uses get_or_create pattern to avoid duplicates.
        Ensures capacity_max record has discount=0 (base price).

        Returns:
            int: Number of new records created

        Example:
            If accommodation has capacity_min=2, capacity_max=6:
            Creates records for capacity=2,3,4,5,6
            Capacity=6 record has discount=0 (base price)
        """
        from apps.accommodations.models import AccommodationPricesPerPerson

        created_count = 0

        for capacity in range(self.accommodation.capacity_min, self.accommodation.capacity_max + 1):
            # Capacity_max is always base price (discount=0)
            discount = 0 if capacity == self.accommodation.capacity_max else None

            record, created = AccommodationPricesPerPerson.objects.get_or_create(
                accommodation=self.accommodation,
                capacity=capacity,
                defaults={
                    'discount': discount,
                    'mode': '%',
                    'is_active': True if discount == 0 else False,  # Only base price active by default
                }
            )

            # Update capacity_max record if it exists but has wrong discount
            if not created and capacity == self.accommodation.capacity_max:
                if record.discount != 0:
                    record.discount = 0
                    record.is_active = True
                    record.save()
                    logger.info(
                        f"Updated base price record for {self.accommodation.name} "
                        f"(capacity={capacity}, discount=0)"
                    )

            if created:
                created_count += 1
                logger.info(
                    f"Created pricing record: {self.accommodation.name} - "
                    f"capacity={capacity}, discount={discount}"
                )

        return created_count

    @transaction.atomic
    def cleanup_pricing_records(self) -> int:
        """
        Delete pricing records outside current capacity range.

        Called when accommodation capacity_min or capacity_max changes.

        Returns:
            int: Number of records deleted

        Example:
            If capacity was 2-6, now 3-5:
            Deletes records for capacity=2 and capacity=6
        """
        from apps.accommodations.models import AccommodationPricesPerPerson

        deleted_result = AccommodationPricesPerPerson.objects.filter(
            accommodation=self.accommodation
        ).exclude(
            capacity__gte=self.accommodation.capacity_min,
            capacity__lte=self.accommodation.capacity_max
        ).delete()

        deleted_count = deleted_result[0]

        if deleted_count > 0:
            logger.info(
                f"Deleted {deleted_count} pricing records outside range "
                f"{self.accommodation.capacity_min}-{self.accommodation.capacity_max} "
                f"for {self.accommodation.name}"
            )

        return deleted_count

    def get_price_for_capacity(self, num_guests: int) -> Decimal:
        """
        Calculate price per night for given number of guests.

        Logic:
        1. Query: capacity >= num_guests, is_active=True, order by capacity ASC
        2. Get first match (smallest capacity tier that fits)
        3. If no match, use accommodation.price_per_night
        4. Calculate final price using discount mode

        Args:
            num_guests: Number of guests for the reservation

        Returns:
            Decimal: Final price per night

        Example:
            Accommodation: capacity_min=2, capacity_max=6, price_per_night=100
            Pricing records:
            - capacity=2, discount=40, mode='%' → $60/night
            - capacity=4, discount=20, mode='%' → $80/night
            - capacity=6, discount=0, mode='%' → $100/night

            Query num_guests=3:
            - Finds capacity=4 record (smallest >= 3)
            - Returns $80/night (20% discount)
        """
        from apps.accommodations.models import AccommodationPricesPerPerson

        # Find smallest capacity tier that fits num_guests
        pricing_record = AccommodationPricesPerPerson.objects.filter(
            accommodation=self.accommodation,
            capacity__gte=num_guests,
            is_active=True
        ).order_by('capacity').first()

        # No active pricing found → use base price
        if not pricing_record:
            logger.info(
                f"No active pricing for {self.accommodation.name} with {num_guests} guests. "
                f"Using base price ${self.accommodation.price_per_night}"
            )
            return self.accommodation.price_per_night

        # Calculate final price
        final_price = pricing_record.calculate_final_price()

        logger.info(
            f"Price for {self.accommodation.name} with {num_guests} guests: "
            f"${final_price} (tier: capacity={pricing_record.capacity}, "
            f"discount={pricing_record.discount}{pricing_record.mode})"
        )

        return final_price


class AccommodationPricesPerPersonQuerySet(models.QuerySet):
    """Custom queryset for AccommodationPricesPerPerson."""

    def active(self) -> 'AccommodationPricesPerPersonQuerySet':
        """Filter to active pricing records only."""
        return self.filter(is_active=True)

    def for_site(self, site: 'Site') -> 'AccommodationPricesPerPersonQuerySet':
        """
        Filter pricing records by site (multi-tenant isolation).

        Args:
            site: Site instance to filter by

        Returns:
            Filtered queryset with only records for given site
        """
        return self.filter(accommodation__site=site)

    def for_accommodation(self, accommodation: 'AccommodationType') -> 'AccommodationPricesPerPersonQuerySet':
        """Filter pricing records for specific accommodation."""
        return self.filter(accommodation=accommodation)


class AccommodationPricesPerPersonManager(models.Manager):
    """Custom manager for AccommodationPricesPerPerson."""

    def get_queryset(self) -> AccommodationPricesPerPersonQuerySet:
        """Return custom queryset."""
        return AccommodationPricesPerPersonQuerySet(self.model, using=self._db)

    def active(self) -> AccommodationPricesPerPersonQuerySet:
        """Shortcut for active pricing records."""
        return self.get_queryset().active()

    def for_site(self, site: 'Site') -> AccommodationPricesPerPersonQuerySet:
        """Shortcut for site filtering."""
        return self.get_queryset().for_site(site)
```

**Add manager to model:**

```python
# In AccommodationPricesPerPerson model
objects = AccommodationPricesPerPersonManager()
```

---

## Integration Points

### 1. Reservation Model - Price Calculation

**Location:** `apps/reservations/models.py`

```python
class Reservation(models.Model):
    # ... existing fields ...

    def calculate_total_price(self) -> Decimal:
        """
        Calculate total price using dynamic pricing.

        Steps:
        1. Get price per night for num_guests from pricing manager
        2. Calculate num_nights from check_in/check_out
        3. total_price = price_per_night * num_nights

        Updates:
        - self.price_per_night
        - self.num_nights
        - self.total_price

        Returns:
            Decimal: Total price for reservation
        """
        from apps.accommodations.managers import AccommodationPricingManager

        # Calculate num_nights
        self.num_nights = (self.check_out - self.check_in).days

        # Get dynamic price per night
        pricing_manager = AccommodationPricingManager(self.accommodation_type)
        self.price_per_night = pricing_manager.get_price_for_capacity(self.num_guests)

        # Calculate total
        self.total_price = self.price_per_night * self.num_nights

        return self.total_price
```

### 2. Price Preview API Endpoint

**Location:** `apps/reservations/api_views.py`

```python
from django.http import JsonResponse
from django.views import View
from django.shortcuts import get_object_or_404
from apps.accommodations.models import AccommodationType
from apps.accommodations.managers import AccommodationPricingManager
from decimal import Decimal
from datetime import datetime


class PricePreviewView(View):
    """
    HTMX endpoint for price preview in reservation form.

    GET params:
    - accommodation_id: int
    - num_guests: int
    - check_in: YYYY-MM-DD
    - check_out: YYYY-MM-DD

    Returns:
    {
        "price_per_night": "80.00",
        "num_nights": 3,
        "total_price": "240.00",
        "discount_applied": "20%",
        "capacity_tier": 4
    }
    """

    def get(self, request, site_slug: str):
        """Calculate price preview."""
        site = get_object_or_404(Site, slug=site_slug, is_active=True)

        # Validate params
        try:
            accommodation_id = int(request.GET.get('accommodation_id'))
            num_guests = int(request.GET.get('num_guests'))
            check_in = datetime.strptime(request.GET.get('check_in'), '%Y-%m-%d').date()
            check_out = datetime.strptime(request.GET.get('check_out'), '%Y-%m-%d').date()
        except (ValueError, TypeError) as e:
            return JsonResponse({'error': 'Invalid parameters'}, status=400)

        # Multi-tenant check
        accommodation = get_object_or_404(
            AccommodationType,
            id=accommodation_id,
            site=site,
            is_active=True
        )

        # Calculate price
        pricing_manager = AccommodationPricingManager(accommodation)
        price_per_night = pricing_manager.get_price_for_capacity(num_guests)
        num_nights = (check_out - check_in).days
        total_price = price_per_night * num_nights

        # Get pricing record for metadata
        from apps.accommodations.models import AccommodationPricesPerPerson
        pricing_record = AccommodationPricesPerPerson.objects.filter(
            accommodation=accommodation,
            capacity__gte=num_guests,
            is_active=True
        ).order_by('capacity').first()

        discount_display = None
        capacity_tier = None

        if pricing_record:
            discount_display = f"{pricing_record.discount}{pricing_record.mode}" if pricing_record.discount else "Sin descuento"
            capacity_tier = pricing_record.capacity

        return JsonResponse({
            'price_per_night': str(price_per_night),
            'num_nights': num_nights,
            'total_price': str(total_price),
            'discount_applied': discount_display,
            'capacity_tier': capacity_tier,
        })
```

**URL:**

```python
# apps/reservations/urls.py
path('<slug:site_slug>/api/price-preview/', PricePreviewView.as_view(), name='price-preview'),
```

---

## Testing Strategy

**Location:** `tests/accommodations/test_pricing_system.py`

See security.md for complete test suites (70%+ coverage target).

---

## Admin Interface Multi-tenant Security

**Location:** `apps/accommodations/admin.py`

```python
from django.contrib import admin
from django.db.models import Q
from apps.accommodations.models import AccommodationPricesPerPerson


@admin.register(AccommodationPricesPerPerson)
class AccommodationPricesPerPersonAdmin(admin.ModelAdmin):
    """Admin interface for pricing records with multi-tenant isolation."""

    list_display = ['accommodation', 'capacity', 'discount', 'mode', 'is_active']
    list_filter = ['accommodation__site', 'is_active', 'mode']
    search_fields = ['accommodation__name']
    ordering = ['accommodation', 'capacity']

    def get_queryset(self, request):
        """Filter pricing records by user's sites (multi-tenant isolation)."""
        qs = super().get_queryset(request)

        if request.user.is_superuser:
            return qs

        return qs.filter(
            Q(accommodation__site__owner=request.user) |
            Q(accommodation__site__admins=request.user)
        ).distinct()

    def formfield_for_foreignkey(self, db_field, request, **kwargs):
        """Filter accommodation choices by user's sites."""
        if db_field.name == "accommodation":
            if not request.user.is_superuser:
                kwargs["queryset"] = AccommodationType.objects.filter(
                    Q(site__owner=request.user) |
                    Q(site__admins=request.user)
                ).distinct()

        return super().formfield_for_foreignkey(db_field, request, **kwargs)

    def has_change_permission(self, request, obj=None):
        """Check change permission (multi-tenant)."""
        if not super().has_change_permission(request, obj):
            return False

        if obj is None:
            return True

        if request.user.is_superuser:
            return True

        site = obj.accommodation.site
        return site.owner == request.user or request.user in site.admins.all()

    def has_delete_permission(self, request, obj=None):
        """Check delete permission (multi-tenant)."""
        if not super().has_delete_permission(request, obj):
            return False

        if obj is None:
            return True

        if request.user.is_superuser:
            return True

        site = obj.accommodation.site
        return site.owner == request.user or request.user in site.admins.all()
```

---

## Summary

**Architecture Decision Summary:**

1. **Signal-based auto-creation** (post_save on AccommodationType)
   - ✅ Separation of concerns
   - ✅ Testable in isolation
   - ✅ Django best practice

2. **Manager pattern for business logic**
   - `AccommodationPricingManager` handles creation, cleanup, price calculation
   - Keeps models thin, testable

3. **Multi-tenant isolation through FK chain**
   - AccommodationPricesPerPerson → AccommodationType → Site
   - All queries filtered by Site

4. **Type-safe price calculation**
   - Returns Decimal (not float)
   - Validates inputs
   - No None returns

5. **Comprehensive validation**
   - clean() method enforces business rules
   - ValidationError with Spanish messages
   - Edge case protection (base price, capacity range)
