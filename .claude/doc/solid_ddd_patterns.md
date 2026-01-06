# SOLID & DDD Patterns Implementation Guide

**Comprehensive reference for SOLID principles and DDD patterns implemented in this project.**

---

## Table of Contents

1. [Overview](#overview)
2. [Value Objects](#1-value-objects-frozen-dataclass)
3. [DTO Pattern](#2-dto-pattern-reservationdata)
4. [Domain Services](#3-domain-services)
5. [Builder Pattern](#4-builder-pattern)
6. [Specification Pattern](#5-specification-pattern)
7. [Command Pattern](#6-command-pattern)
8. [Integration in Views](#7-integration-in-views-and-forms)
9. [Project Structure](#8-project-structure)
10. [Testing Considerations](#9-testing-considerations)

---

## Overview

### DDD Patterns Implemented

| Pattern | Purpose | Implementation | File Location |
|---------|---------|----------------|---------------|
| Value Object | Immutable domain concepts | `DateRange`, `GuestInfo`, `Money` | `src/domain/value_objects/` |
| DTO | Data transfer without behavior | `ReservationData` | `src/domain/value_objects/reservation_data.py` |
| Builder | Complex object construction | `ReservationBuilder` | `src/services/reservation_builder.py` |
| Specification | Business rule validation | `NoOverlapSpecification`, etc. | `apps/reservations/specifications.py` |
| Command | Encapsulated operations | `ApproveCommand`, `CancelCommand` | `apps/reservations/commands.py` |
| Domain Service | Cross-entity operations | `AvailabilityService` | `src/services/availability_service.py` |

### SOLID Principles Applied

| Principle | Application |
|-----------|-------------|
| **S**ingle Responsibility | Each class has one job (e.g., `DateRange` only handles dates) |
| **O**pen/Closed | Specifications extend via composition (`.and_()`, `.or_()`) |
| **L**iskov Substitution | All Specifications are interchangeable via base class |
| **I**nterface Segregation | Small, focused interfaces (e.g., `is_satisfied_by()`) |
| **D**ependency Inversion | Views depend on abstractions (Specification), not implementations |

---

## 1. Value Objects (frozen dataclass)

**Pattern:** Immutable objects without identity, defined by their attributes.

### DateRange Implementation

**File:** `src/domain/value_objects/date_range.py`

```python
from dataclasses import dataclass
from datetime import date

@dataclass(frozen=True)
class DateRange:
    """
    Immutable value object representing a date range.

    Attributes:
        check_in: Start date (inclusive)
        check_out: End date (exclusive)

    Properties:
        nights: Number of nights in range

    Methods:
        overlaps_with: Check overlap with another DateRange
        contains: Check if date is in range
    """
    check_in: date
    check_out: date

    def __post_init__(self) -> None:
        """Validate check_in < check_out."""
        if self.check_in >= self.check_out:
            raise ValueError(
                f"check_in ({self.check_in}) must be before check_out ({self.check_out})"
            )

    @property
    def nights(self) -> int:
        """Calculate number of nights."""
        return (self.check_out - self.check_in).days

    def overlaps_with(self, other: "DateRange") -> bool:
        """Check if ranges overlap (adjacent ranges don't overlap)."""
        return self.check_in < other.check_out and self.check_out > other.check_in

    def contains(self, target_date: date) -> bool:
        """Check if date is in range [check_in, check_out)."""
        return self.check_in <= target_date < self.check_out
```

### GuestInfo Implementation

**File:** `src/domain/value_objects/guest_info.py`

```python
@dataclass(frozen=True)
class GuestInfo:
    """Guest contact information value object with XSS sanitization."""
    name: str
    email: str
    phone: str = ""

    def __post_init__(self) -> None:
        if not self.name or not self.name.strip():
            raise ValueError("name cannot be empty")
        if not self.email or "@" not in self.email:
            raise ValueError("valid email is required")

    def sanitize(self) -> "GuestInfo":
        """Return XSS-sanitized copy."""
        from src.utils.sanitizers import sanitize_text
        return GuestInfo(
            name=sanitize_text(self.name),
            email=self.email,  # Email doesn't need sanitization
            phone=sanitize_text(self.phone)
        )
```

### Money Implementation

```python
@dataclass(frozen=True)
class Money:
    """Immutable money with currency validation."""
    amount: Decimal
    currency: str = "ARS"

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Currency mismatch")
        return Money(self.amount + other.amount, self.currency)

    def apply_discount(self, percentage: Decimal) -> "Money":
        discount = (self.amount * percentage) / Decimal("100")
        return Money(self.amount - discount, self.currency)
```

### Key Design Decisions

1. `frozen=True` makes them immutable (hashable, usable in sets/dicts)
2. `__post_init__` validates invariants at construction
3. Business logic encapsulated (overlap detection, arithmetic)
4. No database dependencies (pure Python)

**When to use:** Dates, money, guest info, coordinates - any concept without identity.

---

## 2. DTO Pattern (ReservationData)

**Pattern:** Immutable container for transferring data between layers without behavior.

### Purpose

DTOs validate data BEFORE creating Django models, enabling:
- Validation without touching the database
- Clean separation between form data and domain models
- Specification pattern compatibility

### Implementation

**File:** `src/domain/value_objects/reservation_data.py`

```python
@dataclass(frozen=True)
class ReservationData:
    """
    Immutable DTO for reservation validation.

    Used to validate data BEFORE creating Django models.
    Supports multiple factory methods for different data sources.
    """
    site_id: int
    accommodation_type_id: int
    date_range: DateRange
    num_guests: int
    guest_info: Optional[GuestInfo] = None
    accommodation_unity_id: Optional[int] = None
    guest_notes: str = ""
    exclude_reservation_id: Optional[int] = None

    def __post_init__(self) -> None:
        """Validate all required fields."""
        if not isinstance(self.site_id, int) or self.site_id <= 0:
            raise ValueError("site_id must be positive integer")
        if not isinstance(self.accommodation_type_id, int) or self.accommodation_type_id <= 0:
            raise ValueError("accommodation_type_id must be positive integer")
        if not isinstance(self.num_guests, int) or self.num_guests <= 0:
            raise ValueError("num_guests must be positive integer")
        if not isinstance(self.date_range, DateRange):
            raise TypeError("date_range must be DateRange instance")

    @classmethod
    def from_reservation(cls, reservation: Reservation) -> "ReservationData":
        """Create from existing Reservation model (for updates)."""
        return cls(
            site_id=reservation.site_id,
            accommodation_type_id=reservation.accommodation_type_id,
            date_range=DateRange(
                check_in=reservation.check_in,
                check_out=reservation.check_out
            ),
            num_guests=reservation.num_guests,
            guest_info=GuestInfo(
                name=reservation.guest_name,
                email=reservation.guest_email,
                phone=reservation.guest_phone or ""
            ),
            exclude_reservation_id=reservation.id  # Exclude self for overlap check
        )

    @classmethod
    def from_form_data(cls, form_data: dict) -> "ReservationData":
        """Create from form cleaned_data dictionary."""
        def get_id(value):
            return value.id if hasattr(value, 'id') else value

        return cls(
            site_id=get_id(form_data['site']),
            accommodation_type_id=get_id(form_data['accommodation_type']),
            date_range=DateRange(
                check_in=form_data['check_in'],
                check_out=form_data['check_out']
            ),
            num_guests=form_data['num_guests'],
            guest_info=GuestInfo(
                name=form_data['guest_name'],
                email=form_data['guest_email'],
                phone=form_data.get('guest_phone', '')
            )
        )

    @classmethod
    def from_builder(cls, builder: ReservationBuilder) -> "ReservationData":
        """Create from ReservationBuilder instance."""
        if not builder._guest_info:
            raise ValueError("Builder must have guest_info set")
        if not builder._date_range:
            raise ValueError("Builder must have date_range set")
        if not builder._num_guests:
            raise ValueError("Builder must have num_guests set")

        return cls(
            site_id=builder._site.id,
            accommodation_type_id=builder._accommodation_type.id,
            date_range=builder._date_range,
            num_guests=builder._num_guests,
            guest_info=builder._guest_info
        )
```

### Usage Examples

```python
# From form data in Form.clean()
reservation_data = ReservationData(
    site_id=self.site.id,
    accommodation_type_id=self.accommodation_type.id,
    date_range=DateRange(check_in=self.check_in, check_out=self.check_out),
    num_guests=self.num_guests,
    guest_info=GuestInfo(
        name=cleaned_data["guest_name"],
        email=cleaned_data["guest_email"],
        phone=cleaned_data.get("guest_phone", "")
    )
)

# Validate with Specification
spec = ReservationValidationSpecification.create_standard_validation()
if not spec.is_satisfied_by(reservation_data):
    raise ValidationError(spec.get_error_message())
```

---

## 3. Domain Services

**Pattern:** Stateless operations that don't belong to entities.

### AvailabilityService Implementation

**File:** `src/services/availability_service.py`

```python
@dataclass(frozen=True)
class AvailabilityResult:
    """Immutable result object."""
    available: bool
    reason: Optional[str] = None
    available_units_count: int = 0


class AvailabilityService:
    """Stateless domain service for availability logic."""

    # Statuses that block availability
    BLOCKING_STATUSES = ["initialized", "pending", "confirmed"]

    @staticmethod
    def check_type_availability(
        accommodation_type: AccommodationType,
        date_range: DateRange,
        exclude_reservation_id: Optional[int] = None
    ) -> AvailabilityResult:
        """Check if accommodation type has available units."""
        available_units = AvailabilityService.get_all_available_units(
            accommodation_type=accommodation_type,
            date_range=date_range,
            exclude_reservation_id=exclude_reservation_id,
        )

        units_count = available_units.count()
        is_available = units_count > 0

        if not is_available:
            return AvailabilityResult(
                available=False,
                reason="No units available for selected dates",
                available_units_count=0,
            )

        return AvailabilityResult(
            available=True,
            available_units_count=units_count,
        )

    @staticmethod
    def get_first_available_unit(
        accommodation_type: AccommodationType,
        date_range: DateRange,
        exclude_reservation_id: Optional[int] = None,
    ) -> Optional[AccommodationUnity]:
        """Get first available unit for auto-assignment."""
        available_units = AvailabilityService.get_all_available_units(
            accommodation_type=accommodation_type,
            date_range=date_range,
            exclude_reservation_id=exclude_reservation_id,
        )
        return available_units.first()

    @staticmethod
    def get_all_available_units(
        accommodation_type: AccommodationType,
        date_range: DateRange,
        exclude_reservation_id: Optional[int] = None,
    ) -> QuerySet:
        """Get all available units using COUNT annotation."""
        units = AccommodationUnity.objects.filter(
            accommodation_type=accommodation_type,
            is_active=True,
        )

        # Build reservation filter
        reservation_filter = Q(
            reservations__check_in__lt=date_range.check_out,
            reservations__check_out__gt=date_range.check_in,
            reservations__status__in=AvailabilityService.BLOCKING_STATUSES,
        )

        if exclude_reservation_id:
            reservation_filter &= ~Q(reservations__id=exclude_reservation_id)

        # Annotate with count of overlapping reservations
        units = units.annotate(
            reservation_count=Count("reservations", filter=reservation_filter)
        )

        # Filter to only available units
        return units.filter(reservation_count=0).order_by("order", "name")
```

**When to use:** Logic that spans multiple entities/aggregates.

---

## 4. Builder Pattern

**Pattern:** Step-by-step object construction with fluent interface.

### ReservationBuilder Implementation

**File:** `src/services/reservation_builder.py`

```python
class ReservationBuilder:
    """
    Builder for creating reservations with fluent interface.

    Features:
    - Step-by-step construction with validation at each step
    - Value object integration (GuestInfo, DateRange)
    - Automatic pricing calculation
    - Automatic unit assignment
    - XSS sanitization
    """

    def __init__(self, site: Site, accommodation_type: AccommodationType) -> None:
        """Initialize with required dependencies."""
        # Validate site ownership (multi-tenant isolation)
        if accommodation_type.site_id != site.id:
            raise ValueError("AccommodationType doesn't belong to site")

        self._site = site
        self._accommodation_type = accommodation_type
        self._guest_info: Optional[GuestInfo] = None
        self._date_range: Optional[DateRange] = None
        self._num_guests: Optional[int] = None
        self._guest_notes: str = ""
        self._user: Optional[User] = None
        self._accommodation_unity: Optional[AccommodationUnity] = None

    def with_guest_info(
        self,
        name_or_guest_info: Union[str, GuestInfo],
        email: Optional[str] = None,
        phone: Optional[str] = None
    ) -> "ReservationBuilder":
        """Set guest info (accepts GuestInfo or individual fields)."""
        if isinstance(name_or_guest_info, GuestInfo):
            self._guest_info = name_or_guest_info.sanitize()
        else:
            guest_info = GuestInfo(
                name=name_or_guest_info,
                email=email,
                phone=phone or ""
            )
            self._guest_info = guest_info.sanitize()
        return self

    def with_dates(
        self,
        check_in_or_range: Union[date, DateRange],
        check_out: Optional[date] = None
    ) -> "ReservationBuilder":
        """Set dates (accepts DateRange or individual dates)."""
        if isinstance(check_in_or_range, DateRange):
            self._date_range = check_in_or_range
        else:
            # Validate not in past
            if check_in_or_range < date.today():
                raise ValueError("check_in cannot be in the past")
            self._date_range = DateRange(
                check_in=check_in_or_range,
                check_out=check_out
            )
        return self

    def with_guests(self, num_guests: int) -> "ReservationBuilder":
        """Set number of guests with capacity validation."""
        if num_guests < self._accommodation_type.capacity_min:
            raise ValueError(
                f"num_guests ({num_guests}) is below minimum "
                f"capacity ({self._accommodation_type.capacity_min})"
            )
        if num_guests > self._accommodation_type.capacity_max:
            raise ValueError(
                f"num_guests ({num_guests}) exceeds maximum "
                f"capacity ({self._accommodation_type.capacity_max})"
            )
        self._num_guests = num_guests
        return self

    def with_notes(self, notes: str) -> "ReservationBuilder":
        """Set guest notes with XSS sanitization."""
        self._guest_notes = sanitize_text(notes)
        return self

    def with_user(self, user: User) -> "ReservationBuilder":
        """Set authenticated user."""
        self._user = user
        return self

    def auto_assign_unit(self) -> "ReservationBuilder":
        """Automatically assign first available unit."""
        if not self._date_range:
            raise ValueError("Dates must be set before auto_assign_unit()")

        unit = AvailabilityService.get_first_available_unit(
            accommodation_type=self._accommodation_type,
            date_range=self._date_range
        )

        if not unit:
            raise ValueError(
                f"No units available for {self._accommodation_type.name} "
                f"from {self._date_range.check_in} to {self._date_range.check_out}"
            )

        self._accommodation_unity = unit
        return self

    def build(self) -> Reservation:
        """Build and persist the reservation atomically."""
        # Validate all required fields
        if not self._guest_info:
            raise ValueError("Guest info must be set before build()")
        if not self._date_range:
            raise ValueError("Dates must be set before build()")
        if not self._num_guests:
            raise ValueError("Number of guests must be set before build()")
        if not self._accommodation_unity:
            raise ValueError("Unit must be assigned before build()")

        # Calculate pricing
        nights = self._date_range.nights
        price_per_night = self._accommodation_type.price_per_night
        total_price = price_per_night * nights

        with transaction.atomic():
            return Reservation.objects.create(
                site=self._site,
                accommodation_type=self._accommodation_type,
                accommodation_unity=self._accommodation_unity,
                user=self._user,
                guest_name=self._guest_info.name,
                guest_email=self._guest_info.email,
                guest_phone=self._guest_info.phone,
                check_in=self._date_range.check_in,
                check_out=self._date_range.check_out,
                num_guests=self._num_guests,
                num_nights=nights,
                price_per_night=price_per_night,
                total_price=total_price,
                guest_notes=self._guest_notes,
                status="initialized"
            )
```

### Usage Examples

```python
# Full fluent construction
reservation = (
    ReservationBuilder(site, accommodation_type)
    .with_guest_info("John Doe", "john@example.com", "123456")
    .with_dates(date(2025, 12, 1), date(2025, 12, 5))
    .with_guests(4)
    .with_notes("Late check-in requested")
    .auto_assign_unit()
    .build()
)

# Using Value Objects directly
date_range = DateRange(check_in=date(2025, 12, 1), check_out=date(2025, 12, 5))
guest_info = GuestInfo(name="John Doe", email="john@example.com", phone="123456")

reservation = (
    ReservationBuilder(site, accommodation_type)
    .with_guest_info(guest_info)
    .with_dates(date_range)
    .with_guests(4)
    .auto_assign_unit()
    .build()
)
```

---

## 5. Specification Pattern

**Pattern:** Composable business rules as objects.

### Base Implementation

**File:** `apps/reservations/specifications.py`

```python
from abc import ABC, abstractmethod
from typing import Union, Optional

class Specification(ABC):
    """Abstract base for composable business rules."""

    @abstractmethod
    def is_satisfied_by(self, data: Union[Reservation, ReservationData]) -> bool:
        """Check if entity satisfies this specification."""
        pass

    @abstractmethod
    def get_error_message(self) -> str:
        """Get human-readable error message (in target language)."""
        pass

    def and_(self, other: "Specification") -> "AndSpecification":
        """Combine with AND logic."""
        return AndSpecification(self, other)

    def or_(self, other: "Specification") -> "OrSpecification":
        """Combine with OR logic."""
        return OrSpecification(self, other)


class AndSpecification(Specification):
    """Composite specification with AND logic."""

    def __init__(self, spec1: Specification, spec2: Specification):
        self._spec1 = spec1
        self._spec2 = spec2
        self._failed_spec: Optional[Specification] = None

    def is_satisfied_by(self, data) -> bool:
        if not self._spec1.is_satisfied_by(data):
            self._failed_spec = self._spec1
            return False
        if not self._spec2.is_satisfied_by(data):
            self._failed_spec = self._spec2
            return False
        return True

    def get_error_message(self) -> str:
        if self._failed_spec:
            return self._failed_spec.get_error_message()
        return f"{self._spec1.get_error_message()} y {self._spec2.get_error_message()}"
```

### Concrete Specifications

```python
class DateRangeValidSpecification(Specification):
    """Specification: check_out > check_in."""

    def is_satisfied_by(self, data) -> bool:
        if isinstance(data, ReservationData):
            return True  # Already validated in DateRange constructor
        try:
            DateRange(check_in=data.check_in, check_out=data.check_out)
            return True
        except ValueError:
            return False

    def get_error_message(self) -> str:
        return "La fecha de salida debe ser posterior a la fecha de entrada."


class NoOverlapSpecification(Specification):
    """Specification: No overlapping reservations for accommodation type."""

    def __init__(self, exclude_reservation_id: Optional[int] = None):
        self.exclude_reservation_id = exclude_reservation_id

    def is_satisfied_by(self, data) -> bool:
        # Extract fields based on type (Reservation or ReservationData)
        if isinstance(data, ReservationData):
            accommodation_type_id = data.accommodation_type_id
            date_range = data.date_range
            exclude_id = data.exclude_reservation_id or self.exclude_reservation_id
        else:
            date_range = DateRange(check_in=data.check_in, check_out=data.check_out)
            accommodation_type_id = data.accommodation_type_id
            exclude_id = self.exclude_reservation_id

        accommodation_type = AccommodationType.objects.get(id=accommodation_type_id)
        result = AvailabilityService.check_type_availability(
            accommodation_type=accommodation_type,
            date_range=date_range,
            exclude_reservation_id=exclude_id,
        )
        return result.available

    def get_error_message(self) -> str:
        return (
            "No hay unidades disponibles para las fechas seleccionadas. "
            "Ya existe una reserva confirmada para este periodo."
        )


class CapacityValidSpecification(Specification):
    """Specification: capacity_min <= num_guests <= capacity_max."""

    def is_satisfied_by(self, data) -> bool:
        if isinstance(data, ReservationData):
            accommodation_type_id = data.accommodation_type_id
            num_guests = data.num_guests
        else:
            accommodation_type_id = data.accommodation_type_id
            num_guests = data.num_guests

        accommodation = AccommodationType.objects.get(id=accommodation_type_id)

        if num_guests < accommodation.capacity_min:
            return False
        if num_guests > accommodation.capacity_max:
            return False
        return True

    def get_error_message(self) -> str:
        return "El numero de huespedes no esta dentro de la capacidad permitida."
```

### Factory Method for Standard Validation

```python
class ReservationValidationSpecification:
    """Factory for creating standard validation specifications."""

    @staticmethod
    def create_standard_validation(
        exclude_reservation_id: Optional[int] = None
    ) -> Specification:
        """
        Create standard validation specification.

        Combines:
        - DateRangeValidSpecification
        - CapacityValidSpecification
        - NoOverlapSpecification

        Args:
            exclude_reservation_id: ID to exclude from overlap check (for updates)

        Returns:
            Composite Specification with AND logic
        """
        date_valid = DateRangeValidSpecification()
        capacity_valid = CapacityValidSpecification()
        no_overlap = NoOverlapSpecification(exclude_reservation_id)

        return date_valid.and_(capacity_valid).and_(no_overlap)
```

### Usage Examples

```python
# Single specification
spec = NoOverlapSpecification()
if not spec.is_satisfied_by(reservation):
    raise ValidationError(spec.get_error_message())

# Factory method (recommended)
spec = ReservationValidationSpecification.create_standard_validation()
if not spec.is_satisfied_by(reservation_data):
    raise ValidationError(spec.get_error_message())

# For updates (exclude self from overlap check)
spec = ReservationValidationSpecification.create_standard_validation(
    exclude_reservation_id=existing_reservation.id
)

# Manual composition
date_spec = DateRangeValidSpecification()
capacity_spec = CapacityValidSpecification()
combined = date_spec.and_(capacity_spec)

if not combined.is_satisfied_by(data):
    print(combined.get_error_message())
```

---

## 6. Command Pattern

**Pattern:** Encapsulated operations with execute/can_execute.

**File:** `apps/reservations/commands.py`

```python
from abc import ABC, abstractmethod

class ReservationCommand(ABC):
    """Base command for reservation operations."""

    def __init__(self, reservation: Reservation, user: User):
        self.reservation = reservation
        self.user = user
        self._previous_state = {}

    @abstractmethod
    def execute(self) -> bool:
        pass

    @abstractmethod
    def can_execute(self) -> bool:
        pass


class ApproveReservationCommand(ReservationCommand):
    """Command to approve a pending reservation."""

    def can_execute(self) -> bool:
        return self.reservation.status == "pending"

    def execute(self) -> bool:
        if not self.can_execute():
            return False
        self._previous_state["status"] = self.reservation.status
        self.reservation.status = "confirmed"
        self.reservation.validated_by = self.user
        self.reservation.validated_at = timezone.now()
        self.reservation.save()
        return True


class CancelReservationCommand(ReservationCommand):
    """Command to cancel a reservation."""

    def __init__(self, reservation, user, reason: str = ""):
        super().__init__(reservation, user)
        self.reason = reason

    def can_execute(self) -> bool:
        return self.reservation.status in ["pending", "confirmed"]

    def execute(self) -> bool:
        if not self.can_execute():
            return False
        self._previous_state["status"] = self.reservation.status
        self.reservation.status = "cancelled"
        self.reservation.admin_notes = f"Cancelled by {self.user.email}: {self.reason}"
        self.reservation.save()
        return True
```

### Usage in Views

```python
class ReservationApproveView(SiteAdminRequiredMixin, View):
    def post(self, request, pk):
        reservation = get_object_or_404(Reservation, pk=pk, site=request.site)

        command = ApproveReservationCommand(reservation, request.user)
        if command.can_execute():
            command.execute()
            messages.success(request, "Reserva aprobada.")
        else:
            messages.error(request, "No se puede aprobar esta reserva.")

        return redirect("reservations:detail", pk=pk)
```

---

## 7. Integration in Views and Forms

### Form Validation with Specifications (SINGLE SOURCE OF TRUTH)

**File:** `apps/reservations/forms.py`

```python
class ReservationCheckoutForm(forms.Form):
    """Form with DDD validation integration."""

    guest_name = forms.CharField(max_length=200)
    guest_email = forms.EmailField()
    guest_phone = forms.CharField(max_length=20, required=False)
    guest_notes = forms.CharField(widget=forms.Textarea, required=False)

    def __init__(self, site, accommodation_type, check_in, check_out, num_guests, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.site = site
        self.accommodation_type = accommodation_type
        self.check_in = check_in
        self.check_out = check_out
        self.num_guests = num_guests

    def clean(self) -> dict:
        """
        Perform cross-field validation using Specification Pattern (DDD).

        This is the SINGLE SOURCE OF TRUTH for reservation validation.
        All validation rules are centralized in Specifications.
        """
        cleaned_data = super().clean()

        # Skip if prerequisites not available
        if not all([self.site, self.accommodation_type, self.check_in, self.check_out, self.num_guests]):
            return cleaned_data

        # Create DateRange (validates dates)
        try:
            date_range = DateRange(check_in=self.check_in, check_out=self.check_out)
        except ValueError as e:
            raise ValidationError({"check_out": str(e)})

        # Create GuestInfo for validation
        guest_info = GuestInfo(
            name=cleaned_data.get("guest_name", ""),
            email=cleaned_data.get("guest_email", ""),
            phone=cleaned_data.get("guest_phone", ""),
        )

        # Create DTO for Specification validation
        reservation_data = ReservationData(
            site_id=self.site.id,
            accommodation_type_id=self.accommodation_type.id,
            date_range=date_range,
            num_guests=self.num_guests,
            guest_info=guest_info,
        )

        # Validate with Specifications (SINGLE SOURCE OF TRUTH)
        spec = ReservationValidationSpecification.create_standard_validation()
        if not spec.is_satisfied_by(reservation_data):
            raise ValidationError(spec.get_error_message())

        return cleaned_data
```

### View Using Builder Pattern

**File:** `apps/reservations/views.py`

```python
class ReservationCheckoutView(View):
    """Checkout view using Builder pattern for reservation creation."""

    def _create_reservation(self, request, form):
        """
        Create reservation using ReservationBuilder.

        Builder ensures:
        - XSS sanitization
        - Value object integration
        - Automatic unit assignment
        - Atomic creation
        """
        cleaned_data = form.cleaned_data

        builder = (
            ReservationBuilder(self.site, self.accommodation)
            .with_guest_info(
                cleaned_data["guest_name"],
                cleaned_data["guest_email"],
                cleaned_data.get("guest_phone", "")
            )
            .with_dates(self.check_in, self.check_out)
            .with_guests(self.num_guests)
        )

        # Optional fields
        if cleaned_data.get("guest_notes"):
            builder = builder.with_notes(cleaned_data["guest_notes"])

        if request.user.is_authenticated:
            builder = builder.with_user(request.user)

        # CRITICAL: Auto-assign unit ensures unit is ALWAYS set
        return builder.auto_assign_unit().build()
```

---

## 8. Project Structure

```
src/
  domain/
    value_objects/
      __init__.py              # Exports: DateRange, GuestInfo, Money, ReservationData
      date_range.py            # DateRange value object
      guest_info.py            # GuestInfo value object
      money.py                 # Money value object
      reservation_data.py      # ReservationData DTO
  services/
    availability_service.py    # Domain service for availability
    reservation_builder.py     # Builder pattern
  utils/
    sanitizers.py              # XSS sanitization utilities

apps/reservations/
  specifications.py            # Specification pattern implementations
  commands.py                  # Command pattern implementations
  forms.py                     # Forms with DDD integration
  views.py                     # Views using Builder and Commands
  managers.py                  # Repository-style manager

tests/
  domain/
    test_date_range.py         # Value object tests
    test_guest_info.py
    test_reservation_data.py   # DTO tests
  services/
    test_reservation_builder.py
    test_availability_service.py
  reservations/
    test_specifications.py     # Specification tests
    test_commands.py           # Command tests
```

---

## 9. Testing Considerations

### Testing Value Objects

```python
def test_date_range_rejects_invalid_dates(self):
    with self.assertRaises(ValueError):
        DateRange(check_in=date(2025, 12, 5), check_out=date(2025, 12, 1))

def test_date_range_calculates_nights(self):
    dr = DateRange(check_in=date(2025, 12, 1), check_out=date(2025, 12, 5))
    self.assertEqual(dr.nights, 4)

def test_date_range_overlap_detection(self):
    range1 = DateRange(check_in=date(2025, 12, 1), check_out=date(2025, 12, 5))
    range2 = DateRange(check_in=date(2025, 12, 3), check_out=date(2025, 12, 7))
    self.assertTrue(range1.overlaps_with(range2))
```

### Testing Specifications

```python
class NoOverlapSpecificationTest(TestCase):
    def setUp(self):
        # Create accommodation type with 2 units
        # Delete auto-created unit from signal
        AccommodationUnity.objects.filter(
            accommodation_type=self.accommodation_type
        ).delete()
        # Create controlled units
        self.unit1 = AccommodationUnity.objects.create(...)
        self.unit2 = AccommodationUnity.objects.create(...)

    def test_overlap_returns_false_when_no_units_available(self):
        # Book both units
        Reservation.objects.create(accommodation_unity=self.unit1, ...)
        Reservation.objects.create(accommodation_unity=self.unit2, ...)

        spec = NoOverlapSpecification()
        result = spec.is_satisfied_by(new_overlapping_reservation)

        self.assertFalse(result)
```

### Testing Builder

```python
def test_builder_creates_reservation_with_all_fields(self):
    reservation = (
        ReservationBuilder(self.site, self.accommodation)
        .with_guest_info("Test", "test@example.com", "123")
        .with_dates(future_date(5), future_date(10))
        .with_guests(2)
        .auto_assign_unit()
        .build()
    )
    self.assertIsNotNone(reservation.pk)
    self.assertEqual(reservation.guest_name, "Test")
    self.assertIsNotNone(reservation.accommodation_unity)

def test_auto_assign_raises_when_no_units(self):
    # Book all units
    ...
    builder = ReservationBuilder(self.site, self.accommodation)
    builder.with_dates(future_date(5), future_date(10))

    with self.assertRaises(ValueError) as ctx:
        builder.auto_assign_unit()

    self.assertIn("no units available", str(ctx.exception).lower())
```

---

## Best Practices Summary

### 1. Value Objects
- Always use `frozen=True` for immutability
- Validate invariants in `__post_init__`
- Keep them pure (no I/O, no database)
- Provide meaningful comparison and business methods

### 2. DTOs
- Make DTOs immutable (`frozen=True`)
- Provide factory methods for different sources (`from_form_data`, `from_reservation`, `from_builder`)
- Validate in `__post_init__`
- Use for validation BEFORE model creation

### 3. Builder Pattern
- Validate at each step when possible
- Use fluent interface (return `self`)
- Integrate with Value Objects
- Make `build()` atomic (transaction)
- Provide `reset()` for reuse

### 4. Specification Pattern
- Keep specifications small and focused (Single Responsibility)
- Use composition via `.and_()` and `.or_()`
- Provide clear error messages in target language
- Create factory methods for common combinations
- Support both Reservation and ReservationData types

### 5. Commands
- Use `can_execute()` for permission checks
- Store previous state for potential rollback
- Keep execute() focused and atomic

### 6. Integration
- Form.clean() uses Specifications (Single Source of Truth)
- Views use Builder for complex object creation
- Domain Services coordinate cross-entity logic
- Tests verify each pattern independently

---

## References

- Evans, Eric. "Domain-Driven Design: Tackling Complexity in the Heart of Software"
- Martin, Robert C. "Clean Architecture"
- Fowler, Martin. "Patterns of Enterprise Application Architecture"
- Django Documentation: https://docs.djangoproject.com/
