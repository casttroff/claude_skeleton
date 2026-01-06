# Backend Research: Payment Integration (MercadoPago)

## Overview

Complete backend architecture for dual-revenue payment system:
1. **Subscription Payments** - Site owners pay monthly for SaaS access (to platform)
2. **Reservation Payments** - End-users pay for reservations (to site owner, 5% platform commission)

**Key Architectural Decisions:**
- Virtual units model (quantity field, no separate AccommodationUnity model)
- Hybrid subscription billing (MercadoPago API + local state tracking)
- Split payment routing (95% to site owner, 5% to platform)
- 7-day grace period with exponential backoff retry
- Atomic availability checking with select_for_update()
- Idempotency patterns for webhooks
- PCI compliance (tokenization, no card storage)

---

## Django Models

### Model 1: Plan

**Purpose**: Subscription plan tiers (GLOBAL - not site-specific)

**Fields**:
```python
# apps/subscriptions/models.py

from django.db import models
from django.core.validators import MinValueValidator
from decimal import Decimal
from framework.models import BaseModel


class Plan(BaseModel):
    """
    Subscription plan tiers (GLOBAL - not site-specific).

    Defines limits and pricing for basic/intermediate/enterprise tiers.
    """

    TIER_CHOICES = [
        ("basic", "Basic"),
        ("intermediate", "Intermediate"),
        ("enterprise", "Enterprise"),
    ]

    tier = models.CharField(max_length=20, choices=TIER_CHOICES, unique=True)
    name = models.CharField(max_length=100)

    # Pricing (dual currency)
    price_monthly_usd = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(Decimal("0.00"))],
        help_text="Monthly price in USD"
    )

    price_monthly_ars = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(Decimal("0.00"))],
        help_text="Monthly price in ARS (updated monthly based on exchange rate)"
    )

    # Limits
    max_sites = models.PositiveIntegerField()
    max_accommodations_per_site = models.PositiveIntegerField()
    max_admin_users = models.PositiveIntegerField()
    storage_mb = models.PositiveIntegerField(help_text="Storage limit in MB")

    # Features (JSON for flexibility)
    features_json = models.JSONField(default=dict)

    is_active = models.BooleanField(default=True)
    order = models.PositiveIntegerField(default=0)

    class Meta:
        db_table = "subscription_plan"
        ordering = ["order", "price_monthly_usd"]

    def __str__(self) -> str:
        return f"{self.name} (${self.price_monthly_usd}/mes)"
```

**Relationships**:
- 1:N with Subscription: Each plan has multiple subscribers

**Validation**:
- Field-level: MinValueValidator on prices ensures positive values
- Model-level: tier unique constraint prevents duplicate tiers

**Indexes**:
- `tier` (unique): Fast plan lookup
- `(order, price_monthly_usd)`: Fast ordering for display

---

### Model 2: Subscription

**Purpose**: User subscription to a plan (one per site owner)

**Fields**:
```python
# apps/subscriptions/models.py

class Subscription(BaseModel):
    """
    User subscription to a plan (one per site owner).

    Tracks trial period, billing cycle, payment failures, and grace period.
    """

    STATUS_CHOICES = [
        ("trial", "Free Trial"),
        ("active", "Active"),
        ("payment_failed", "Payment Failed (Grace Period)"),
        ("suspended", "Suspended"),
        ("cancelled", "Cancelled"),
    ]

    user = models.OneToOneField(
        "users.User",
        on_delete=models.CASCADE,
        related_name="subscription"
    )

    plan = models.ForeignKey(Plan, on_delete=models.PROTECT, related_name="subscriptions")
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="trial", db_index=True)

    # MercadoPago tracking
    mercadopago_subscription_id = models.CharField(max_length=100, blank=True, unique=True)
    mercadopago_payer_id = models.CharField(max_length=100, blank=True)

    # Trial period
    trial_start_date = models.DateField(null=True, blank=True)
    trial_end_date = models.DateField(null=True, blank=True)

    # Billing period
    current_period_start = models.DateField()
    current_period_end = models.DateField()

    # Cancellation
    cancelled_at = models.DateTimeField(null=True, blank=True)

    # Payment failure tracking (7-day grace period)
    payment_failed_at = models.DateTimeField(null=True, blank=True)
    payment_retry_count = models.PositiveIntegerField(default=0)
    grace_period_ends_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        db_table = "subscription"
        indexes = [
            models.Index(fields=["user"]),
            models.Index(fields=["status"]),
            models.Index(fields=["current_period_end"]),
        ]

    @property
    def is_active(self) -> bool:
        """Check if subscription allows platform access."""
        return self.status in ["trial", "active", "payment_failed"]

    @property
    def is_in_trial(self) -> bool:
        """Check if in free trial period."""
        from datetime import date

        if self.status != "trial":
            return False

        return self.trial_end_date and date.today() <= self.trial_end_date

    @property
    def days_until_renewal(self) -> int:
        """Calculate days until next billing cycle."""
        from datetime import date

        if not self.current_period_end:
            return 0

        delta = self.current_period_end - date.today()
        return max(0, delta.days)
```

**Relationships**:
- 1:1 with User: Each user has at most one subscription
- N:1 with Plan: Many subscriptions to one plan
- 1:N with PaymentLog: Track all payment transactions

**Validation**:
- Model-level: OneToOneField prevents duplicate subscriptions per user
- Business logic: is_active property enforces access rules

**Indexes**:
- `user` (unique via OneToOneField): Fast user lookup
- `status`: Fast filtering by subscription state
- `current_period_end`: Efficient renewal checks

---

### Model 3: PaymentLog

**Purpose**: Comprehensive payment tracking for debugging and compliance

**Fields**:
```python
# apps/subscriptions/models.py

class PaymentLog(BaseModel):
    """
    Comprehensive payment tracking for debugging and compliance.

    Stores complete MercadoPago response for both subscription and reservation payments.
    """

    PAYMENT_TYPE_CHOICES = [
        ("subscription", "Subscription Payment"),
        ("reservation", "Reservation Payment"),
    ]

    payment_type = models.CharField(max_length=20, choices=PAYMENT_TYPE_CHOICES, db_index=True)

    # Relations (only one will be set)
    subscription = models.ForeignKey(
        Subscription,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="payment_logs"
    )

    reservation = models.ForeignKey(
        "reservations.Reservation",
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="payment_logs"
    )

    # MercadoPago response (complete JSON)
    mercadopago_response = models.JSONField(default=dict)

    # Parsed fields (for queries)
    mercadopago_payment_id = models.CharField(max_length=100, db_index=True)
    status = models.CharField(max_length=50, db_index=True)
    status_detail = models.CharField(max_length=100, blank=True)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    currency = models.CharField(max_length=3, default="ARS")
    payment_method_id = models.CharField(max_length=50, blank=True)

    # Webhook tracking
    webhook_received_at = models.DateTimeField(null=True, blank=True)
    processed_at = models.DateTimeField(null=True, blank=True)
    error_message = models.TextField(blank=True)

    class Meta:
        db_table = "payment_log"
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["payment_type", "-created_at"]),
            models.Index(fields=["mercadopago_payment_id"]),
            models.Index(fields=["status"]),
        ]
```

**Relationships**:
- N:1 with Subscription: Track all subscription payments
- N:1 with Reservation: Track all reservation payments

**Validation**:
- Model-level: Check constraint ensures only one of (subscription, reservation) is set
- Field-level: payment_type validates against allowed choices

**Indexes**:
- `(payment_type, -created_at)`: Fast filtering by type and date
- `mercadopago_payment_id`: Fast lookup for webhook idempotency
- `status`: Fast filtering by payment status

---

### Model 4: AccommodationType (Modified)

**Purpose**: Add virtual units support via quantity field

**Fields**:
```python
# apps/accommodations/models.py

class AccommodationType(BaseModel):
    """
    Accommodation type with virtual units.

    Example: 3 identical cabañas = quantity=3
    Availability: count(active_reservations) < 3

    This pattern is Booking.com/Airbnb compatible and avoids over-engineering
    with separate AccommodationUnity model.
    """

    # ... existing fields ...

    quantity = models.PositiveIntegerField(
        default=1,
        validators=[MinValueValidator(1)],
        help_text="Number of identical units available (e.g., 3 cabañas)"
    )

    def get_available_units(self, check_in: date, check_out: date) -> int:
        """
        Get number of available units for a date range.

        Args:
            check_in: Start date (inclusive)
            check_out: End date (exclusive)

        Returns:
            int: Number of units available (0 to quantity)
        """
        overlapping_count = self.reservations.filter(
            status__in=["pending", "confirmed"],
            check_in__lt=check_out,
            check_out__gt=check_in
        ).count()

        return max(0, self.quantity - overlapping_count)
```

**Migration**:
```python
# apps/accommodations/migrations/XXXX_add_quantity_field.py

from django.db import migrations, models
from django.core.validators import MinValueValidator


class Migration(migrations.Migration):

    dependencies = [
        ('accommodations', 'XXXX_previous_migration'),
    ]

    operations = [
        migrations.AddField(
            model_name='accommodationtype',
            name='quantity',
            field=models.PositiveIntegerField(
                default=1,
                validators=[MinValueValidator(1)],
                help_text='Number of identical units available'
            ),
        ),
    ]
```

---

### Model 5: Reservation (Modified)

**Purpose**: Add payment tracking and timeout fields

**Fields**:
```python
# apps/reservations/models.py

class Reservation(models.Model):
    # ... existing fields ...

    # NEW: Payment tracking fields
    payment_status = models.CharField(
        max_length=20,
        choices=[
            ("pending", "Pending Payment"),
            ("authorized", "Payment Authorized"),
            ("approved", "Payment Approved"),
            ("rejected", "Payment Rejected"),
            ("cancelled", "Payment Cancelled"),
        ],
        default="pending",
        db_index=True,
    )

    mercadopago_payment_id = models.CharField(max_length=100, blank=True, unique=True)
    mercadopago_preference_id = models.CharField(max_length=100, blank=True)
    payment_method = models.CharField(max_length=50, blank=True)

    # NEW: Timeout tracking (10 min for payment)
    reservation_expires_at = models.DateTimeField(
        null=True,
        blank=True,
        db_index=True,
        help_text="When pending reservation expires (10 min from creation)"
    )

    def save(self, *args, **kwargs):
        """Set expiration time for new pending reservations."""
        from datetime import timedelta
        from django.utils import timezone

        if not self.pk and self.payment_status == "pending":
            self.reservation_expires_at = timezone.now() + timedelta(minutes=10)

        super().save(*args, **kwargs)

    @property
    def is_expired(self) -> bool:
        """Check if reservation has expired."""
        from django.utils import timezone

        if not self.reservation_expires_at:
            return False

        return (
            self.payment_status == "pending" and
            timezone.now() > self.reservation_expires_at
        )

    def cancel_if_expired(self) -> bool:
        """
        Cancel reservation if payment timeout expired.

        Returns:
            bool: True if cancelled, False if not expired
        """
        if not self.is_expired:
            return False

        self.status = "cancelled"
        self.payment_status = "cancelled"
        self.admin_notes = "Automatically cancelled - payment timeout"
        self.save()
        return True
```

**Migration**:
```python
# apps/reservations/migrations/XXXX_add_payment_fields.py

from django.db import migrations, models
from django.utils import timezone
from datetime import timedelta


class Migration(migrations.Migration):

    dependencies = [
        ('reservations', 'XXXX_previous_migration'),
    ]

    operations = [
        migrations.AddField(
            model_name='reservation',
            name='payment_status',
            field=models.CharField(
                max_length=20,
                choices=[
                    ('pending', 'Pending Payment'),
                    ('authorized', 'Payment Authorized'),
                    ('approved', 'Payment Approved'),
                    ('rejected', 'Payment Rejected'),
                    ('cancelled', 'Payment Cancelled'),
                ],
                default='pending',
                db_index=True,
            ),
        ),
        migrations.AddField(
            model_name='reservation',
            name='mercadopago_payment_id',
            field=models.CharField(max_length=100, blank=True, unique=True),
        ),
        migrations.AddField(
            model_name='reservation',
            name='mercadopago_preference_id',
            field=models.CharField(max_length=100, blank=True),
        ),
        migrations.AddField(
            model_name='reservation',
            name='payment_method',
            field=models.CharField(max_length=50, blank=True),
        ),
        migrations.AddField(
            model_name='reservation',
            name='reservation_expires_at',
            field=models.DateTimeField(
                null=True,
                blank=True,
                db_index=True,
                help_text='When pending reservation expires (10 min from creation)'
            ),
        ),
    ]
```

---

### Model 6: Site (Modified)

**Purpose**: Add MercadoPago collector ID for split payments

**Fields**:
```python
# apps/core/models.py

class Site(BaseModel):
    # ... existing fields ...

    # NEW: MercadoPago integration
    mercadopago_collector_id = models.CharField(
        max_length=100,
        blank=True,
        null=True,
        help_text="Site owner's MercadoPago account ID (for receiving payments)"
    )

    mercadopago_onboarding_completed = models.BooleanField(
        default=False,
        help_text="Whether site owner completed MercadoPago setup"
    )

    def can_receive_reservations(self) -> bool:
        """Check if site can receive paid reservations."""
        return self.mercadopago_onboarding_completed
```

**Migration**:
```python
# apps/core/migrations/XXXX_add_mercadopago_fields.py

from django.db import migrations, models


class Migration(migrations.Migration):

    dependencies = [
        ('core', 'XXXX_previous_migration'),
    ]

    operations = [
        migrations.AddField(
            model_name='site',
            name='mercadopago_collector_id',
            field=models.CharField(
                max_length=100,
                blank=True,
                null=True,
                help_text="Site owner's MercadoPago account ID"
            ),
        ),
        migrations.AddField(
            model_name='site',
            name='mercadopago_onboarding_completed',
            field=models.BooleanField(
                default=False,
                help_text='Whether site owner completed MercadoPago setup'
            ),
        ),
    ]
```

---

## Managers and QuerySets

### Manager: AccommodationTypeManager

**Purpose**: Business logic for accommodation availability

**Location**: `apps/accommodations/managers.py`

**Methods**:
```python
# apps/accommodations/managers.py

from django.db import models
from django.db.models import Count, Q, F
from datetime import date
from typing import Optional
from apps.core.models import Site


class AccommodationTypeManager(models.Manager):
    """
    Manager for AccommodationType with availability logic.

    Encapsulates complex queries for accommodation filtering.
    """

    def get_available(
        self,
        site: Site,
        check_in: date,
        check_out: date,
        min_capacity: int = 1
    ):
        """
        Get accommodations with at least 1 available unit for date range.

        Args:
            site: Site for multi-tenant filtering
            check_in: Check-in date (inclusive)
            check_out: Check-out date (exclusive)
            min_capacity: Minimum guest capacity

        Returns:
            QuerySet: Available accommodations with annotated available_units field
        """
        return self.filter(
            site=site,
            is_active=True,
            capacity_max__gte=min_capacity
        ).annotate(
            # Count overlapping reservations
            occupied_units=Count(
                'reservations',
                filter=Q(
                    reservations__status__in=["pending", "confirmed"],
                    reservations__check_in__lt=check_out,
                    reservations__check_out__gt=check_in
                )
            )
        ).annotate(
            # Calculate available units
            available_units=F('quantity') - F('occupied_units')
        ).filter(
            available_units__gt=0
        )
```

---

### Manager: SubscriptionManager

**Purpose**: Business logic for subscription queries

**Location**: `apps/subscriptions/managers.py`

**Methods**:
```python
# apps/subscriptions/managers.py

from django.db import models
from django.db.models import Q
from datetime import date
from django.utils import timezone


class SubscriptionManager(models.Manager):
    """
    Manager for Subscription with renewal and grace period logic.
    """

    def ending_trials(self) -> models.QuerySet:
        """
        Get subscriptions with trials ending today.

        Returns:
            QuerySet: Subscriptions ready for activation
        """
        today = date.today()

        return self.filter(
            status="trial",
            trial_end_date__lte=today
        )

    def in_grace_period(self) -> models.QuerySet:
        """
        Get subscriptions in payment failure grace period.

        Returns:
            QuerySet: Subscriptions that need payment retry
        """
        now = timezone.now()

        return self.filter(
            status="payment_failed",
            grace_period_ends_at__gt=now
        )

    def grace_period_expired(self) -> models.QuerySet:
        """
        Get subscriptions with expired grace period (need suspension).

        Returns:
            QuerySet: Subscriptions to suspend
        """
        now = timezone.now()

        return self.filter(
            status="payment_failed",
            grace_period_ends_at__lte=now
        )

    def active_for_user(self, user) -> Optional['Subscription']:
        """
        Get active subscription for user (including grace period).

        Args:
            user: User instance

        Returns:
            Subscription or None: Active subscription if exists
        """
        try:
            subscription = self.get(user=user)
            return subscription if subscription.is_active else None
        except self.model.DoesNotExist:
            return None
```

---

## Services Layer

### Service: SubscriptionService

**Purpose**: MercadoPago subscription operations

**Location**: `apps/subscriptions/services.py`

**Key Methods**:
```python
# apps/subscriptions/services.py

import mercadopago
from django.conf import settings
from apps.subscriptions.models import Subscription
from typing import Dict, Any
import logging

logger = logging.getLogger(__name__)


class SubscriptionService:
    """Service for subscription operations with MercadoPago."""

    @staticmethod
    def create_customer(email: str, payment_token: str) -> str:
        """
        Create MercadoPago customer with tokenized card (NO CHARGE YET).

        Args:
            email: User email
            payment_token: MercadoPago card token from frontend

        Returns:
            str: MercadoPago payer_id (customer ID)

        Raises:
            ValueError: If MercadoPago API fails
        """
        sdk = mercadopago.SDK(settings.MERCADOPAGO_ACCESS_TOKEN)

        customer_data = {
            "email": email,
            "card_token": payment_token,
        }

        response = sdk.customer().create(customer_data)

        if response["status"] != 201:
            logger.error(f"Failed to create MercadoPago customer: {response}")
            raise ValueError(f"Failed to create customer: {response}")

        return response["response"]["id"]

    @staticmethod
    def activate_subscription(subscription: Subscription) -> str:
        """
        Create MercadoPago recurring subscription after trial ends.

        Args:
            subscription: Subscription instance

        Returns:
            str: MercadoPago subscription ID

        Raises:
            ValueError: If MercadoPago API fails
        """
        sdk = mercadopago.SDK(settings.MERCADOPAGO_ACCESS_TOKEN)

        preapproval_data = {
            "payer_id": subscription.mercadopago_payer_id,
            "back_url": f"https://{settings.SITE_DOMAIN}/subscriptions/success/",
            "reason": f"Suscripción {subscription.plan.name}",
            "auto_recurring": {
                "frequency": 1,
                "frequency_type": "months",
                "transaction_amount": float(subscription.plan.price_monthly_ars),
                "currency_id": "ARS",
            },
            "external_reference": str(subscription.id),
        }

        response = sdk.preapproval().create(preapproval_data)

        if response["status"] != 201:
            logger.error(f"Failed to create MercadoPago subscription: {response}")
            raise ValueError(f"Failed to create subscription: {response}")

        return response["response"]["id"]

    @staticmethod
    def cancel_subscription(subscription: Subscription) -> bool:
        """
        Cancel MercadoPago recurring subscription.

        Args:
            subscription: Subscription instance

        Returns:
            bool: True if cancelled successfully

        Raises:
            ValueError: If MercadoPago API fails
        """
        if not subscription.mercadopago_subscription_id:
            return True  # Nothing to cancel

        sdk = mercadopago.SDK(settings.MERCADOPAGO_ACCESS_TOKEN)

        response = sdk.preapproval().update(
            subscription.mercadopago_subscription_id,
            {"status": "cancelled"}
        )

        if response["status"] not in [200, 204]:
            logger.error(f"Failed to cancel MercadoPago subscription: {response}")
            raise ValueError(f"Failed to cancel subscription: {response}")

        return True

    @staticmethod
    def retry_payment(subscription: Subscription) -> bool:
        """
        Retry failed subscription payment via MercadoPago.

        Args:
            subscription: Subscription instance

        Returns:
            bool: True if payment successful, False if failed
        """
        sdk = mercadopago.SDK(settings.MERCADOPAGO_ACCESS_TOKEN)

        # Get latest payment attempt
        response = sdk.preapproval().search(
            filters={"external_reference": str(subscription.id)}
        )

        if response["status"] != 200:
            logger.error(f"Failed to get subscription payments: {response}")
            return False

        payments = response["response"].get("results", [])
        if not payments:
            return False

        latest_payment = payments[0]
        return latest_payment["status"] == "approved"

    @staticmethod
    def get_payment_info(payment_id: str) -> Dict[str, Any]:
        """
        Get payment details from MercadoPago.

        Args:
            payment_id: MercadoPago payment ID

        Returns:
            dict: Payment information

        Raises:
            ValueError: If payment not found
        """
        sdk = mercadopago.SDK(settings.MERCADOPAGO_ACCESS_TOKEN)

        response = sdk.payment().get(payment_id)

        if response["status"] != 200:
            logger.error(f"Failed to get payment info: {response}")
            raise ValueError(f"Payment not found: {payment_id}")

        return response["response"]
```

**Dependencies**:
- Uses: Subscription model, MercadoPago SDK
- Called by: Views (checkout, webhook), Celery tasks (renewal, retry)

---

### Service: ReservationPaymentService

**Purpose**: MercadoPago split payment for reservations

**Location**: `apps/reservations/services.py`

**Key Methods**:
```python
# apps/reservations/services.py

import mercadopago
from django.conf import settings
from apps.reservations.models import Reservation
from decimal import Decimal
from typing import Dict, Any
import logging

logger = logging.getLogger(__name__)


class ReservationPaymentService:
    """Service for reservation payments with MercadoPago."""

    @staticmethod
    def create_payment_preference(reservation: Reservation) -> Dict[str, Any]:
        """
        Create MercadoPago payment preference with split payment.

        Money routing:
        - 95% → Site owner (mercadopago_collector_id)
        - 5% → Platform (marketplace_fee)

        Args:
            reservation: Reservation instance

        Returns:
            dict: MercadoPago preference response with init_point URL

        Raises:
            ValueError: If site owner hasn't completed MercadoPago onboarding
        """
        # Verify site owner has MercadoPago account connected
        if not reservation.site.mercadopago_collector_id:
            raise ValueError(
                f"Site {reservation.site.name} must complete MercadoPago onboarding first"
            )

        sdk = mercadopago.SDK(settings.MERCADOPAGO_ACCESS_TOKEN)

        # Platform commission (5%)
        platform_fee = reservation.total_price * Decimal("0.05")

        preference_data = {
            "items": [
                {
                    "title": f"Reserva - {reservation.accommodation_type.name}",
                    "quantity": 1,
                    "unit_price": float(reservation.total_price),
                    "currency_id": "ARS",
                }
            ],
            "payer": {
                "email": reservation.guest_email,
                "name": reservation.guest_name,
                "phone": {
                    "number": reservation.guest_phone
                },
            },
            "marketplace": settings.MERCADOPAGO_MARKETPLACE_ID,  # Platform ID
            "marketplace_fee": float(platform_fee),  # 5% to platform
            "collector_id": reservation.site.mercadopago_collector_id,  # Site owner receives 95%
            "back_urls": {
                "success": f"https://{reservation.site.domain}/reservas/success/{reservation.id}/",
                "failure": f"https://{reservation.site.domain}/reservas/failure/{reservation.id}/",
                "pending": f"https://{reservation.site.domain}/reservas/pending/{reservation.id}/",
            },
            "auto_return": "approved",
            "external_reference": str(reservation.id),
            "expires": True,
            "expiration_date_from": reservation.created_at.isoformat(),
            "expiration_date_to": reservation.reservation_expires_at.isoformat(),
        }

        response = sdk.preference().create(preference_data)

        if response["status"] != 201:
            logger.error(f"Failed to create payment preference: {response}")
            raise ValueError(f"Failed to create payment preference: {response}")

        return response["response"]
```

**Dependencies**:
- Uses: Reservation model, Site model, MercadoPago SDK
- Called by: ReservationCheckoutView

---

## Forms

### Form: SubscriptionCheckoutForm

**Purpose**: Validate subscription checkout with card tokenization

**Location**: `apps/subscriptions/forms.py`

**Fields**:
```python
# apps/subscriptions/forms.py

from django import forms
from django.core.exceptions import ValidationError
import bleach


class SubscriptionCheckoutForm(forms.Form):
    """
    Subscription checkout form with MercadoPago card tokenization.

    Card tokenization happens in frontend (MercadoPago SDK).
    Backend receives payment_token only (PCI compliant).
    """

    payment_token = forms.CharField(
        max_length=255,
        required=True,
        widget=forms.HiddenInput(),
        help_text="MercadoPago card token (from frontend SDK)"
    )

    honeypot = forms.CharField(
        required=False,
        widget=forms.HiddenInput()
    )

    def clean_honeypot(self):
        """Bot detection via honeypot field."""
        honeypot = self.cleaned_data.get("honeypot")
        if honeypot:
            raise ValidationError("Bot detected")
        return honeypot

    def clean_payment_token(self):
        """Validate payment token format."""
        token = self.cleaned_data.get("payment_token")
        token = bleach.clean(token, tags=[], strip=True)

        if not token or len(token) < 10:
            raise ValidationError("Invalid payment token")

        return token
```

**Validation Strategy**:
- Sanitization: Bleach on payment_token (remove any HTML)
- Business rules: Honeypot for bot detection
- Multi-tenant: N/A (user-level, not site-specific)

---

### Form: SubscriptionCancelForm

**Purpose**: Optional feedback on subscription cancellation

**Location**: `apps/subscriptions/forms.py`

**Fields**:
```python
# apps/subscriptions/forms.py

class SubscriptionCancelForm(forms.Form):
    """
    Optional feedback form on cancellation.

    All fields are optional - user can cancel without feedback.
    """

    REASON_CHOICES = [
        ("", "-- Selecciona un motivo (opcional) --"),
        ("too_expensive", "Muy caro"),
        ("not_using", "No lo estoy usando"),
        ("missing_features", "Faltan funcionalidades"),
        ("switching_competitor", "Cambio a otro sistema"),
        ("business_closed", "Cerré mi negocio"),
        ("other", "Otro"),
    ]

    rating = forms.IntegerField(
        required=False,
        min_value=1,
        max_value=5,
        widget=forms.NumberInput(attrs={"class": "rating rating-lg"})
    )

    reason = forms.ChoiceField(
        required=False,
        choices=REASON_CHOICES,
        widget=forms.Select(attrs={"class": "select select-bordered"})
    )

    comments = forms.CharField(
        required=False,
        max_length=500,
        widget=forms.Textarea(attrs={"class": "textarea textarea-bordered", "rows": 4})
    )

    def clean_comments(self):
        """Sanitize comments field."""
        comments = self.cleaned_data.get("comments", "")
        return bleach.clean(comments, tags=[], strip=True)
```

---

### Form: ReservationCheckoutForm

**Purpose**: Guest information for reservation payment

**Location**: `apps/reservations/forms.py`

**Fields**:
```python
# apps/reservations/forms.py

from django import forms
from datetime import date
import bleach


class ReservationCheckoutForm(forms.Form):
    """
    Guest information form for reservation checkout.

    Dates and num_guests come from URL params, this form collects guest details.
    """

    # Guest information
    guest_name = forms.CharField(
        max_length=200,
        required=True,
        widget=forms.TextInput(attrs={"class": "input input-bordered"})
    )

    guest_email = forms.EmailField(
        required=True,
        widget=forms.EmailInput(attrs={"class": "input input-bordered"})
    )

    guest_phone = forms.CharField(
        max_length=50,
        required=True,
        widget=forms.TextInput(attrs={"class": "input input-bordered"})
    )

    guest_notes = forms.CharField(
        required=False,
        max_length=500,
        widget=forms.Textarea(attrs={"class": "textarea textarea-bordered", "rows": 4})
    )

    # Hidden fields from URL params
    check_in = forms.DateField(widget=forms.HiddenInput())
    check_out = forms.DateField(widget=forms.HiddenInput())
    num_guests = forms.IntegerField(widget=forms.HiddenInput())

    honeypot = forms.CharField(required=False, widget=forms.HiddenInput())

    def clean_guest_name(self):
        """Sanitize guest name."""
        name = self.cleaned_data.get("guest_name", "")
        return bleach.clean(name, tags=[], strip=True)

    def clean_guest_notes(self):
        """Sanitize guest notes."""
        notes = self.cleaned_data.get("guest_notes", "")
        return bleach.clean(notes, tags=[], strip=True)

    def clean(self):
        """Cross-field validation."""
        cleaned_data = super().clean()

        # Honeypot check
        if cleaned_data.get("honeypot"):
            raise forms.ValidationError("Bot detected")

        # Date validation
        check_in = cleaned_data.get("check_in")
        check_out = cleaned_data.get("check_out")

        if check_in and check_out:
            # Check in must be in future
            if check_in < date.today():
                raise forms.ValidationError("Check-in date must be in the future")

            # Check out must be after check in
            if check_out <= check_in:
                raise forms.ValidationError("Check-out must be after check-in")

        return cleaned_data
```

**Validation Strategy**:
- Sanitization: Bleach on name and notes
- Business rules: Date validation, honeypot detection
- Multi-tenant: Site context passed from view (not form field)

---

## Celery Tasks

### Task: check_subscriptions_for_renewal

**Purpose**: Daily check for trial endings and billing renewals

**Location**: `apps/subscriptions/tasks.py`

**Implementation**:
```python
# apps/subscriptions/tasks.py

from celery import shared_task
from datetime import date
from django.utils import timezone
import logging

logger = logging.getLogger(__name__)


@shared_task
def check_subscriptions_for_renewal():
    """
    Daily task: Check trials ending and billing renewals.

    Runs: Every day at 00:00 (Celery Beat)
    """
    from apps.subscriptions.models import Subscription
    from apps.subscriptions.services import SubscriptionService
    from apps.subscriptions.tasks import send_subscription_activated_email

    today = date.today()

    # End trials (activate recurring billing)
    trials_ending = Subscription.objects.filter(
        status="trial",
        trial_end_date__lte=today
    )

    for subscription in trials_ending:
        try:
            # Create MercadoPago recurring subscription
            mp_subscription_id = SubscriptionService.activate_subscription(subscription)

            subscription.status = "active"
            subscription.mercadopago_subscription_id = mp_subscription_id
            subscription.save()

            # Send activation email
            send_subscription_activated_email.delay(subscription.id)

            logger.info(f"Activated subscription {subscription.id}")

        except Exception as e:
            logger.error(f"Failed to activate subscription {subscription.id}: {e}")
```

**Trigger**: Celery Beat daily at 00:00
**Retry Strategy**: No retry (runs daily, will catch on next run)
**Error Handling**: Log error, continue with next subscription

---

### Task: retry_failed_subscription_payment

**Purpose**: Retry failed subscription payment during 7-day grace period

**Location**: `apps/subscriptions/tasks.py`

**Implementation**:
```python
# apps/subscriptions/tasks.py

@shared_task(
    bind=True,
    max_retries=5,
    retry_backoff=True,
    default_retry_delay=21600  # 6 hours
)
def retry_failed_subscription_payment(self, subscription_id: int):
    """
    Retry failed subscription payment during grace period.

    Args:
        subscription_id: Subscription ID

    Runs: Every 6 hours for payment_failed subscriptions
    Grace period: 7 days (industry standard)
    Retry strategy: Exponential backoff (6h, 12h, 24h, 48h, 96h)
    """
    from apps.subscriptions.models import Subscription
    from apps.subscriptions.services import SubscriptionService
    from apps.subscriptions.tasks import (
        send_payment_success_email,
        send_suspension_email
    )

    subscription = Subscription.objects.get(id=subscription_id)

    # Check if already resolved
    if subscription.status != "payment_failed":
        logger.info(f"Subscription {subscription_id} already resolved")
        return

    # Check grace period (7 days)
    if timezone.now() > subscription.grace_period_ends_at:
        # Suspend subscription
        subscription.status = "suspended"
        subscription.save()

        # Send suspension email
        send_suspension_email.delay(subscription.id)
        logger.warning(f"Suspended subscription {subscription_id} after grace period")
        return

    # Retry payment
    try:
        success = SubscriptionService.retry_payment(subscription)

        if success:
            subscription.status = "active"
            subscription.payment_failed_at = None
            subscription.payment_retry_count = 0
            subscription.grace_period_ends_at = None
            subscription.save()

            # Send success email
            send_payment_success_email.delay(subscription.id)
            logger.info(f"Payment retry successful for subscription {subscription_id}")
        else:
            subscription.payment_retry_count += 1
            subscription.save()

            # Retry later (up to 5 attempts)
            if subscription.payment_retry_count < 5:
                logger.info(f"Payment retry failed, attempt {subscription.payment_retry_count}/5")
                raise self.retry(countdown=21600)  # Retry in 6 hours

    except Exception as e:
        logger.error(f"Payment retry failed for subscription {subscription_id}: {e}")
        raise self.retry(countdown=21600)
```

**Trigger**: Webhook (payment failed) triggers first retry, then every 6 hours
**Retry Strategy**: Max 5 retries with exponential backoff (6h base delay)
**Error Handling**: Log error, retry up to 5 times, then suspend

---

### Task: cancel_expired_reservations

**Purpose**: Cancel reservations with payment timeout (10 min)

**Location**: `apps/reservations/tasks.py`

**Implementation**:
```python
# apps/reservations/tasks.py

from celery import shared_task
from django.utils import timezone
import logging

logger = logging.getLogger(__name__)


@shared_task
def cancel_expired_reservations():
    """
    Cancel reservations with payment timeout (10 min).

    Runs: Every 5 minutes (Celery Beat)
    """
    from apps.reservations.models import Reservation
    from apps.reservations.tasks import send_reservation_timeout_email

    now = timezone.now()

    # Find expired pending reservations
    expired = Reservation.objects.filter(
        payment_status="pending",
        reservation_expires_at__lte=now
    )

    for reservation in expired:
        if reservation.cancel_if_expired():
            logger.info(f"Cancelled expired reservation {reservation.id}")

            # Send timeout email to guest
            send_reservation_timeout_email.delay(reservation.id)
```

**Trigger**: Celery Beat every 5 minutes
**Retry Strategy**: No retry (runs every 5 min, will catch on next run)
**Error Handling**: Log error, continue with next reservation

---

### Celery Beat Schedule

**Configuration**:
```python
# calma/celery.py

from celery.schedules import crontab

app.conf.beat_schedule = {
    # Subscription renewals (daily at 00:00)
    'check-subscriptions-for-renewal': {
        'task': 'apps.subscriptions.tasks.check_subscriptions_for_renewal',
        'schedule': crontab(hour=0, minute=0),
    },

    # Cancel expired reservations (every 5 min)
    'cancel-expired-reservations': {
        'task': 'apps.reservations.tasks.cancel_expired_reservations',
        'schedule': crontab(minute='*/5'),
    },
}
```

---

## Django Signals

### Signal: Payment Status Update (Subscription)

**Purpose**: Update subscription status when MercadoPago webhook received

**Location**: `apps/subscriptions/signals.py`

**Implementation**:
```python
# apps/subscriptions/signals.py

from django.db.models.signals import post_save
from django.dispatch import receiver
from apps.subscriptions.models import PaymentLog
from datetime import timedelta
import logging

logger = logging.getLogger(__name__)


@receiver(post_save, sender=PaymentLog)
def update_subscription_on_payment(sender, instance, created, **kwargs):
    """
    Update subscription status when payment log created via webhook.

    Args:
        sender: PaymentLog model
        instance: PaymentLog instance
        created: True if new record
    """
    if not created or instance.payment_type != "subscription":
        return

    subscription = instance.subscription
    if not subscription:
        return

    # Handle payment status
    if instance.status == "approved":
        # Payment success
        subscription.status = "active"
        subscription.current_period_end += timedelta(days=30)
        subscription.payment_failed_at = None
        subscription.payment_retry_count = 0
        subscription.grace_period_ends_at = None
        subscription.save()

        logger.info(f"Subscription {subscription.id} renewed successfully")

    elif instance.status in ["rejected", "cancelled"]:
        # Payment failed
        from django.utils import timezone

        subscription.status = "payment_failed"
        subscription.payment_failed_at = timezone.now()
        subscription.grace_period_ends_at = timezone.now() + timedelta(days=7)
        subscription.save()

        # Send failure email + schedule retry
        from apps.subscriptions.tasks import (
            send_payment_failed_email,
            retry_failed_subscription_payment
        )

        send_payment_failed_email.delay(subscription.id)
        retry_failed_subscription_payment.apply_async(
            args=[subscription.id],
            countdown=21600  # Retry in 6 hours
        )

        logger.warning(f"Subscription {subscription.id} payment failed, starting grace period")
```

---

### Signal: Payment Status Update (Reservation)

**Purpose**: Update reservation status when MercadoPago webhook received

**Location**: `apps/reservations/signals.py`

**Implementation**:
```python
# apps/reservations/signals.py

from django.db.models.signals import post_save
from django.dispatch import receiver
from apps.subscriptions.models import PaymentLog
import logging

logger = logging.getLogger(__name__)


@receiver(post_save, sender=PaymentLog)
def update_reservation_on_payment(sender, instance, created, **kwargs):
    """
    Update reservation status when payment log created via webhook.

    Args:
        sender: PaymentLog model
        instance: PaymentLog instance
        created: True if new record
    """
    if not created or instance.payment_type != "reservation":
        return

    reservation = instance.reservation
    if not reservation:
        return

    # Handle payment status
    if instance.status == "approved":
        # Payment approved
        reservation.payment_status = "approved"
        reservation.status = "confirmed"
        reservation.mercadopago_payment_id = instance.mercadopago_payment_id
        reservation.save()

        # Send confirmation emails
        from apps.reservations.tasks import (
            send_reservation_confirmation,
            send_reservation_notification_admin
        )

        send_reservation_confirmation.delay(reservation.id)
        send_reservation_notification_admin.delay(reservation.id)

        logger.info(f"Reservation {reservation.id} confirmed, payment approved")

    elif instance.status == "in_process":
        # Payment processing
        reservation.payment_status = "in_process"
        reservation.save()

        logger.info(f"Reservation {reservation.id} payment in process")

    elif instance.status in ["rejected", "cancelled"]:
        # Payment rejected
        reservation.payment_status = "rejected"
        reservation.status = "cancelled"
        reservation.save()

        # Send failure email
        from apps.reservations.tasks import send_reservation_payment_failed_email
        send_reservation_payment_failed_email.delay(reservation.id)

        logger.warning(f"Reservation {reservation.id} payment rejected")
```

---

## Webhook Endpoints

### Endpoint: POST /subscriptions/webhook/

**Purpose**: Receive MercadoPago subscription payment updates

**Authentication**: HMAC signature verification

**Implementation**:
```python
# apps/subscriptions/views.py

from django.views import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
from django.http import JsonResponse
from apps.subscriptions.services import SubscriptionService
from apps.subscriptions.utils import verify_mercadopago_webhook
from apps.subscriptions.models import Subscription, PaymentLog
import json
import logging

logger = logging.getLogger(__name__)


@method_decorator(csrf_exempt, name='dispatch')
class SubscriptionWebhookView(View):
    """
    MercadoPago webhook for subscription payments.

    Endpoint: POST /subscriptions/webhook/
    Security: HMAC signature verification + idempotency
    """

    def post(self, request):
        """
        Handle MercadoPago webhook for subscription payment updates.

        Webhook events:
        - payment.created: New payment attempt
        - payment.updated: Payment status changed

        Returns:
            JsonResponse: {"status": "ok"} or error
        """
        # Verify webhook signature (CRITICAL)
        if not verify_mercadopago_webhook(request):
            logger.warning("Invalid webhook signature")
            return JsonResponse({"error": "Invalid signature"}, status=401)

        try:
            data = json.loads(request.body)
        except json.JSONDecodeError:
            logger.error("Invalid JSON in webhook")
            return JsonResponse({"error": "Invalid JSON"}, status=400)

        # Extract payment info
        payment_id = data.get("data", {}).get("id")
        action = data.get("action")  # "payment.created", "payment.updated"

        if action not in ["payment.created", "payment.updated"]:
            logger.info(f"Ignored webhook action: {action}")
            return JsonResponse({"status": "ignored"}, status=200)

        # Idempotency check
        if PaymentLog.objects.filter(mercadopago_payment_id=payment_id).exists():
            logger.info(f"Payment {payment_id} already processed (idempotent)")
            return JsonResponse({"status": "already_processed"}, status=200)

        # Get payment details from MercadoPago
        try:
            payment_info = SubscriptionService.get_payment_info(payment_id)
        except ValueError as e:
            logger.error(f"Failed to get payment info: {e}")
            return JsonResponse({"error": str(e)}, status=400)

        # Find subscription by external_reference
        subscription_id = payment_info.get("external_reference")
        if not subscription_id:
            logger.error("Missing external_reference in payment")
            return JsonResponse({"error": "Missing external_reference"}, status=400)

        try:
            subscription = Subscription.objects.get(id=subscription_id)
        except Subscription.DoesNotExist:
            logger.error(f"Subscription {subscription_id} not found")
            return JsonResponse({"error": "Subscription not found"}, status=404)

        # Log payment (triggers signal that updates subscription)
        PaymentLog.objects.create(
            payment_type="subscription",
            subscription=subscription,
            mercadopago_response=payment_info,
            mercadopago_payment_id=payment_id,
            status=payment_info["status"],
            status_detail=payment_info.get("status_detail", ""),
            amount=payment_info["transaction_amount"],
            currency=payment_info["currency_id"],
            payment_method_id=payment_info.get("payment_method_id", ""),
            webhook_received_at=timezone.now(),
        )

        logger.info(f"Processed webhook for subscription {subscription_id}, payment {payment_id}")
        return JsonResponse({"status": "ok"}, status=200)
```

**HMAC Signature Verification**:
```python
# apps/subscriptions/utils.py

import hmac
import hashlib
from django.conf import settings


def verify_mercadopago_webhook(request) -> bool:
    """
    Verify MercadoPago webhook HMAC signature.

    Args:
        request: Django HttpRequest

    Returns:
        bool: True if signature valid, False otherwise
    """
    signature = request.headers.get("X-Signature")

    if not signature:
        return False

    payload = request.body

    expected_signature = hmac.new(
        settings.MERCADOPAGO_WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected_signature)
```

---

### Endpoint: POST /reservas/webhook/

**Purpose**: Receive MercadoPago reservation payment updates

**Authentication**: HMAC signature verification

**Implementation**:
```python
# apps/reservations/views.py

from django.views import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
from django.http import JsonResponse
from apps.subscriptions.services import SubscriptionService
from apps.subscriptions.utils import verify_mercadopago_webhook
from apps.subscriptions.models import PaymentLog
from apps.reservations.models import Reservation
import json
import logging

logger = logging.getLogger(__name__)


@method_decorator(csrf_exempt, name='dispatch')
class ReservationWebhookView(View):
    """
    MercadoPago webhook for reservation payments.

    Endpoint: POST /reservas/webhook/
    Security: HMAC signature verification + idempotency
    """

    def post(self, request):
        """Handle MercadoPago webhook for reservation payment updates."""
        # Verify webhook signature (CRITICAL)
        if not verify_mercadopago_webhook(request):
            logger.warning("Invalid webhook signature")
            return JsonResponse({"error": "Invalid signature"}, status=401)

        try:
            data = json.loads(request.body)
        except json.JSONDecodeError:
            logger.error("Invalid JSON in webhook")
            return JsonResponse({"error": "Invalid JSON"}, status=400)

        # Extract payment info
        payment_id = data.get("data", {}).get("id")
        action = data.get("action")

        if action not in ["payment.created", "payment.updated"]:
            logger.info(f"Ignored webhook action: {action}")
            return JsonResponse({"status": "ignored"}, status=200)

        # Idempotency check
        if PaymentLog.objects.filter(mercadopago_payment_id=payment_id).exists():
            logger.info(f"Payment {payment_id} already processed (idempotent)")
            return JsonResponse({"status": "already_processed"}, status=200)

        # Get payment details from MercadoPago
        try:
            payment_info = SubscriptionService.get_payment_info(payment_id)
        except ValueError as e:
            logger.error(f"Failed to get payment info: {e}")
            return JsonResponse({"error": str(e)}, status=400)

        # Find reservation by external_reference
        reservation_id = payment_info.get("external_reference")
        if not reservation_id:
            logger.error("Missing external_reference in payment")
            return JsonResponse({"error": "Missing external_reference"}, status=400)

        try:
            reservation = Reservation.objects.get(id=reservation_id)
        except Reservation.DoesNotExist:
            logger.error(f"Reservation {reservation_id} not found")
            return JsonResponse({"error": "Reservation not found"}, status=404)

        # Log payment (triggers signal that updates reservation)
        PaymentLog.objects.create(
            payment_type="reservation",
            reservation=reservation,
            mercadopago_response=payment_info,
            mercadopago_payment_id=payment_id,
            status=payment_info["status"],
            status_detail=payment_info.get("status_detail", ""),
            amount=payment_info["transaction_amount"],
            currency=payment_info["currency_id"],
            payment_method_id=payment_info.get("payment_method_id", ""),
            webhook_received_at=timezone.now(),
        )

        logger.info(f"Processed webhook for reservation {reservation_id}, payment {payment_id}")
        return JsonResponse({"status": "ok"}, status=200)
```

---

## Database Queries

### Query Pattern 1: Get Available Accommodations with Units

**Purpose**: Efficiently retrieve accommodations with available units for date range

**Implementation**:
```python
# Optimized query with annotation
from django.db.models import Count, Q, F

accommodations = AccommodationType.objects.filter(
    site=site,  # Multi-tenant
    is_active=True,
    capacity_max__gte=num_guests
).annotate(
    # Count overlapping reservations
    occupied_units=Count(
        'reservations',
        filter=Q(
            reservations__status__in=["pending", "confirmed"],
            reservations__check_in__lt=check_out,
            reservations__check_out__gt=check_in
        )
    )
).annotate(
    # Calculate available units
    available_units=F('quantity') - F('occupied_units')
).filter(
    available_units__gt=0
).select_related('site')  # Prevent N+1
```

**Performance**:
- Expected time: <50ms for 100 accommodations
- Indexes used: `(site, is_active)`, `(check_in, check_out)` on Reservation
- N+1 prevention: `select_related('site')`

---

### Query Pattern 2: Atomic Availability Check with Lock

**Purpose**: Prevent race conditions during reservation creation

**Implementation**:
```python
from django.db import transaction

@transaction.atomic
def create_reservation_with_lock(accommodation, check_in, check_out, **kwargs):
    """
    Create reservation with database lock to prevent overbooking.

    Args:
        accommodation: AccommodationType instance
        check_in: Check-in date
        check_out: Check-out date
        **kwargs: Additional reservation fields

    Returns:
        Reservation: Created reservation

    Raises:
        ValidationError: If no units available
    """
    from django.core.exceptions import ValidationError

    # Lock accommodation row for update
    accommodation = AccommodationType.objects.select_for_update().get(
        id=accommodation.id
    )

    # Check availability (inside transaction lock)
    available_units = accommodation.get_available_units(check_in, check_out)

    if available_units < 1:
        raise ValidationError("No units available for selected dates")

    # Create reservation (within same transaction)
    reservation = Reservation.objects.create(
        accommodation_type=accommodation,
        check_in=check_in,
        check_out=check_out,
        **kwargs
    )

    return reservation
```

**Performance**:
- Expected time: <100ms (includes lock acquisition)
- Indexes used: Primary key index, `(check_in, check_out)` on Reservation
- Concurrency: `select_for_update()` prevents race conditions

---

## Multi-Tenant Isolation

**Critical Points**:
- All subscription queries filter by `user` (user-level isolation)
- All reservation queries filter by `site` (site-level isolation)
- Webhooks validate `external_reference` before processing
- Admin interfaces filter by user's subscriptions/sites

**Pattern**:
```python
# In views
def subscription_view(request):
    # User can only see their own subscription
    subscription = get_object_or_404(Subscription, user=request.user)

def reservation_view(request, site_slug):
    site = get_object_or_404(Site, slug=site_slug)

    # Check user has permission to this site
    if not (site.owner == request.user or request.user in site.admins.all()):
        raise PermissionDenied

    # All queries filter by site
    reservations = Reservation.objects.filter(site=site)

# In webhooks
def webhook_view(request):
    # Validate external_reference belongs to correct tenant
    subscription = Subscription.objects.get(id=external_reference)
    # Process only if valid
```

---

## Type Hints & Docstrings

**Google-style docstrings required on all public methods:**

```python
def create_payment_preference(reservation: Reservation) -> Dict[str, Any]:
    """
    Create MercadoPago payment preference with split payment.

    Money routing:
    - 95% → Site owner (mercadopago_collector_id)
    - 5% → Platform (marketplace_fee)

    Args:
        reservation: Reservation instance with valid site.mercadopago_collector_id

    Returns:
        dict: MercadoPago preference response containing:
            - id (str): Preference ID
            - init_point (str): Checkout URL
            - sandbox_init_point (str): Sandbox checkout URL

    Raises:
        ValueError: If site owner hasn't completed MercadoPago onboarding
        ValueError: If MercadoPago API call fails

    Example:
        >>> preference = ReservationPaymentService.create_payment_preference(reservation)
        >>> redirect_url = preference["init_point"]
    """
```

**Type hints everywhere:**
- Function parameters: `def func(param: Type)`
- Return types: `-> ReturnType`
- Class attributes: `field: Type`
- mypy strict mode must pass

---

## Testing Strategy

**Coverage target**: 70% (Standard Mode - critical payment features)

**Test types needed**:

1. **Model tests**:
   - Subscription.is_active property
   - Subscription.is_in_trial property
   - AccommodationType.get_available_units()
   - Reservation.is_expired property
   - Reservation.cancel_if_expired()

2. **Manager tests**:
   - SubscriptionManager.ending_trials()
   - SubscriptionManager.in_grace_period()
   - AccommodationTypeManager.get_available()

3. **Service tests**:
   - SubscriptionService.create_customer() (mock MercadoPago)
   - SubscriptionService.activate_subscription() (mock MercadoPago)
   - ReservationPaymentService.create_payment_preference() (mock MercadoPago)

4. **Form tests**:
   - SubscriptionCheckoutForm validation
   - ReservationCheckoutForm date validation
   - Honeypot detection

5. **Webhook tests**:
   - HMAC signature verification
   - Idempotency check
   - Signal triggering
   - PaymentLog creation

6. **Integration tests**:
   - Full subscription flow (trial → activation → payment)
   - Full reservation flow (create → payment → confirmation)
   - Payment failure → grace period → retry → suspension

**Multi-tenant test pattern**:
```python
def test_multi_tenant_isolation(self):
    """User A cannot access User B's subscription"""
    user_a = User.objects.create_user(username="user_a")
    user_b = User.objects.create_user(username="user_b")

    subscription_a = Subscription.objects.create(user=user_a, plan=plan)
    subscription_b = Subscription.objects.create(user=user_b, plan=plan)

    # User A should only see their subscription
    client.login(username="user_a")
    response = client.get("/subscriptions/dashboard/")
    assert subscription_a in response.context["subscription"]
    assert subscription_b not in response.context
```

**Atomic availability test**:
```python
def test_concurrent_bookings_prevented(self):
    """Test race condition prevention with select_for_update"""
    import threading
    from django.db import transaction

    accommodation = AccommodationType.objects.create(
        site=site,
        quantity=1,  # Only 1 unit
        # ... other fields
    )

    results = []

    def create_reservation():
        try:
            with transaction.atomic():
                reservation = create_reservation_with_lock(
                    accommodation,
                    check_in=date(2025, 12, 1),
                    check_out=date(2025, 12, 5),
                    guest_name="Test Guest",
                    # ... other fields
                )
                results.append(reservation)
        except ValidationError:
            results.append(None)

    # Simulate 2 concurrent requests
    t1 = threading.Thread(target=create_reservation)
    t2 = threading.Thread(target=create_reservation)

    t1.start()
    t2.start()
    t1.join()
    t2.join()

    # One should succeed, one should fail
    successful = [r for r in results if r is not None]
    assert len(successful) == 1
```

---

## Performance Considerations

**Database**:
- Indexes on:
  - Subscription: `(user)`, `(status)`, `(current_period_end)`
  - PaymentLog: `(payment_type, -created_at)`, `(mercadopago_payment_id)`, `(status)`
  - Reservation: `(payment_status)`, `(reservation_expires_at)`
  - AccommodationType: `(site, is_active)`
- Query optimization:
  - `select_related('site')` on AccommodationType queries
  - `select_related('plan')` on Subscription queries
  - Annotation for available_units (avoid N+1)
- Caching:
  - Plan data (rarely changes): 1 hour TTL
  - Accommodation availability (changes frequently): No cache (query is fast)

**Celery**:
- Which operations must be async:
  - Subscription renewal (daily batch)
  - Payment retry (every 6h during grace period)
  - Reservation timeout cleanup (every 5min)
  - Email sending (all emails)
- Task priorities:
  - High: Payment retry, reservation timeout
  - Normal: Subscription renewal
  - Low: Email sending

**MercadoPago API**:
- Rate limiting: 1000 requests/hour (production)
- Circuit breaker: After 3 consecutive failures, wait 5 min before retry
- Timeout: 10 seconds for API calls

---

## Security Checklist

- [x] All queries filter by user/site (multi-tenant isolation)
- [x] Forms sanitize input with bleach
- [x] Type hints on all functions
- [x] Docstrings on all public methods
- [x] No raw SQL (use ORM parameterized queries)
- [x] No print() statements (use logging)
- [x] Proper error handling (try/except with logging)
- [x] HMAC signature verification on webhooks
- [x] Idempotency checks (prevent duplicate processing)
- [x] PCI compliance (no card storage, tokenization only)
- [x] HTTPS enforced (production)
- [x] CSRF exempt only on webhooks (with HMAC verification)
- [x] select_for_update() for atomic availability checks
- [x] Honeypot fields for bot detection

---

## Recommendations

1. **Use Hybrid Subscription Billing**: MercadoPago handles recurring charges, we track state locally for grace periods and suspension logic

2. **Use Split Payment for Reservations**: Avoids tax liability (platform only taxed on 5% commission, not full reservation amount)

3. **7-Day Grace Period**: Industry standard (not 2 days) - accounts for Argentine payment system unreliability

4. **Virtual Units Pattern**: Simpler than separate AccommodationUnity model, Booking.com compatible

5. **Atomic Availability Checks**: Always use `select_for_update()` to prevent race conditions during concurrent bookings

6. **Idempotency Everywhere**: Check `mercadopago_payment_id` before processing webhooks to prevent duplicate charges/emails

7. **Log Everything**: All MercadoPago API calls, webhook events, payment state transitions - critical for debugging payment issues

8. **Test with MercadoPago Sandbox**: Use test cards before production deployment

---

## Open Questions

- [x] Grace period duration: 7 days (recommended, industry standard)
- [x] Platform commission: 5% (balanced)
- [x] Plan limits: 10/50/unlimited accommodations (recommended)
- [x] AccommodationUnity: Virtual units (no separate model)
- [x] Free trial: 30 days, no CC required (lower friction)
- [ ] Integration priority: Both subscription + reservation in parallel? (awaiting user decision)

---

## Dependencies

**Python Packages**:
```
mercadopago==2.2.0
celery==5.3.4
redis==5.0.1
bleach==6.1.0
```

**Environment Variables**:
```
MERCADOPAGO_ACCESS_TOKEN=your_access_token
MERCADOPAGO_PUBLIC_KEY=your_public_key
MERCADOPAGO_MARKETPLACE_ID=your_marketplace_id
MERCADOPAGO_CLIENT_ID=your_client_id
MERCADOPAGO_CLIENT_SECRET=your_client_secret
MERCADOPAGO_REDIRECT_URI=https://yourdomain.com/sites/mercadopago/callback/
MERCADOPAGO_WEBHOOK_SECRET=your_webhook_secret

# Sandbox (for development)
MERCADOPAGO_SANDBOX_ACCESS_TOKEN=your_sandbox_access_token
MERCADOPAGO_SANDBOX_PUBLIC_KEY=your_sandbox_public_key
```

---

## Implementation Phases

**Phase 1: Data Models** (2-3 days)
- Create models: Plan, Subscription, PaymentLog
- Modify models: AccommodationType (quantity), Reservation (payment fields), Site (mercadopago_collector_id)
- Create migrations
- Create seed_plans management command

**Phase 2: Subscription Payment Integration** (4-5 days)
- Install mercadopago SDK
- Create SubscriptionService
- Create views: SelectPlanView, CheckoutView, WebhookView
- Create Celery tasks: renewal, retry, reminders
- Create email templates

**Phase 3: Reservation Payment Integration** (3-4 days)
- Create ReservationPaymentService
- Modify ReservationCheckoutView
- Create webhook view
- Create Celery task: cancel_expired_reservations

**Phase 4: Site Owner MercadoPago Onboarding** (2-3 days)
- Create OAuth flow views
- Update Site model with collector_id
- Create dashboard UI
- Add validation for onboarding completion

**Phase 5: SubscriptionRequiredMixin** (1-2 days)
- Create mixin
- Apply to all Site-related views
- Implement plan limit checks

**Phase 6: Security Audit & Testing** (2-3 days)
- Run security-auditor
- Test with MercadoPago sandbox
- Load testing (concurrent reservations)
- OWASP Top 10 audit

**Total**: 15-20 days (3-4 weeks)
