# Python Professional Standards - NASA + Wemake Styleguide

---

## Overview

Este documento establece los estandares profesionales de Python para proyectos Django, basado en:

1. **NASA JPL Power of 10** - Reglas para código crítico de seguridad
2. **Wemake Python Styleguide** - Estándares estrictos de calidad (1095 rules)
3. **Google Python Style Guide** - Docstrings y convenciones
4. **PEP 8** - Estilo base de Python

**Filosofía:** "Your computer, RAM, drive, and brain have limits. Cut problems, code, and data into small boxes that fit together within those constraints."

---

## NASA Power of 10 Rules (Adapted to Python)

### Rule 1: Avoid Complex Flow Constructs

**Principle:** Keep control flow simple and predictable.

**Python Rules:**
- ✅ Use if/elif/else chains (clear flow)
- ✅ Use early returns (avoid nesting)
- ❌ NO recursion (use iteration or explicit stack)
- ❌ NO numbered break/continue in nested loops
- ❌ NO complex list comprehensions (max 2 for loops)

**Examples:**

```python
# ✅ GOOD: Clear control flow with early returns
def check_reservation_availability(
    accommodation_type: AccommodationType,
    check_in: date,
    check_out: date,
) -> bool:
    """Check if accommodation is available."""
    # Early validation
    if check_in >= check_out:
        raise ValidationError("Check-in must be before check-out")

    if not accommodation_type.is_active:
        return False

    # Main logic
    overlapping = Reservation.objects.filter(
        accommodation_type=accommodation_type,
        status__in=["pending", "confirmed"],
        check_in__lt=check_out,
        check_out__gt=check_in,
    )

    return not overlapping.exists()


# ❌ BAD: Nested conditions, no early returns
def check_reservation_availability(accommodation_type, check_in, check_out):
    if accommodation_type:
        if check_in and check_out:
            if check_in < check_out:
                if accommodation_type.is_active:
                    overlapping = Reservation.objects.filter(...)
                    if overlapping.exists():
                        return False
                    return True
    return None  # ❌ Also violates Rule 4 (no None returns)


# ✅ GOOD: Simple list comprehension
blocked_dates = [r.check_in for r in reservations if r.status == "confirmed"]

# ❌ BAD: Complex nested comprehension
ast_nodes = [
    target
    for assignment in top_level_assigns
    for target in assignment.targets
    for _ in range(10)  # Too many nested for loops
]
```

---

### Rule 2: Fixed Loop Bounds

**Principle:** All loops must have statically provable upper bounds.

**Python Rules:**
- ✅ Use pagination with explicit limits
- ✅ Use assertions to enforce loop bounds
- ✅ Process data in bounded chunks
- ❌ NO unbounded while loops without iteration counter

**Examples:**

```python
# ✅ GOOD: Bounded iteration with assertion
def iter_max(iterable, max_iter: int):
    """Iterate with maximum iteration limit."""
    count = 0
    for item in iterable:
        assert count < max_iter, f"Exceeded max iterations: {max_iter}"
        yield item
        count += 1


# ✅ GOOD: Pagination with fixed page size
def get_reservations_paginated(site: Site, page: int = 1, page_size: int = 50):
    """Get reservations with bounded query."""
    assert 1 <= page_size <= 100, "Page size must be between 1 and 100"

    offset = (page - 1) * page_size
    return Reservation.objects.filter(site=site)[offset:offset + page_size]


# ✅ GOOD: Process in bounded chunks
def process_large_dataset(queryset: QuerySet, chunk_size: int = 1000):
    """Process large dataset in bounded chunks."""
    assert chunk_size > 0, "Chunk size must be positive"

    total = queryset.count()
    for offset in range(0, total, chunk_size):
        chunk = queryset[offset:offset + chunk_size]
        for item in chunk:
            process_item(item)


# ❌ BAD: Unbounded while loop
def wait_for_completion(task_id):
    while True:  # No upper bound!
        task = Task.objects.get(id=task_id)
        if task.is_complete:
            return task
        time.sleep(1)


# ✅ GOOD: While loop with explicit bound
def wait_for_completion(task_id, max_attempts: int = 60):
    """Wait for task completion with timeout."""
    attempts = 0
    while attempts < max_attempts:
        task = Task.objects.get(id=task_id)
        if task.is_complete:
            return task
        time.sleep(1)
        attempts += 1

    raise TimeoutError(f"Task {task_id} did not complete in {max_attempts}s")
```

---

### Rule 3: Avoid Heap Memory Allocation (Control Memory Bounds)

**Principle:** Memory consumption must be predictable and bounded.

**Python Rules:**
- ✅ Use pagination to limit memory footprint
- ✅ Cap field lengths (CharField max_length)
- ✅ Use iterators/generators for large datasets
- ❌ NO loading entire querysets into memory

**Examples:**

```python
# ✅ GOOD: Generator for memory efficiency
def iter_reservations(site: Site):
    """Iterate reservations without loading all into memory."""
    return Reservation.objects.filter(site=site).iterator(chunk_size=100)


# ✅ GOOD: Field length limits enforce memory bounds
class ContactSubmission(models.Model):
    name = models.CharField(max_length=200)  # Bounded
    email = models.EmailField(max_length=254)  # Bounded
    message = models.TextField(max_length=5000)  # Bounded, not infinite


# ✅ GOOD: Paginated API response
def get_accommodations_api(request):
    """Return paginated accommodations (bounded memory)."""
    page_size = min(int(request.GET.get("page_size", 20)), 100)  # Cap at 100
    accommodations = AccommodationType.objects.filter(
        site=request.site
    )[:page_size]

    return JsonResponse({"results": list(accommodations.values())})


# ❌ BAD: Load everything into memory
def export_all_reservations(site: Site):
    reservations = list(Reservation.objects.filter(site=site))  # All in memory!
    return generate_csv(reservations)


# ✅ GOOD: Stream CSV without loading all
def export_reservations_streaming(site: Site):
    """Stream CSV without loading all reservations."""
    queryset = Reservation.objects.filter(site=site).iterator(chunk_size=500)

    def csv_generator():
        yield "ID,Guest,Check-in,Check-out\n"
        for reservation in queryset:
            yield f"{reservation.id},{reservation.guest_name},...\n"

    return StreamingHttpResponse(csv_generator(), content_type="text/csv")
```

---

### Rule 4: Restrict Functions to Single Page (60 lines max)

**Principle:** Functions must fit on one screen for cognitive manageability.

**Python Rules:**
- ✅ Maximum 60 lines per function (including docstring)
- ✅ Maximum 40 lines of logic (excluding docstring/blank lines)
- ✅ Extract helper functions if exceeding limit
- ✅ Use descriptive function names for decomposition

**Enforcement:** Wemake WPS213 (too many expressions)

**Examples:**

```python
# ✅ GOOD: Small, focused function (25 lines)
def create_reservation(
    site: Site,
    accommodation_type: AccommodationType,
    guest_data: dict,
    dates: dict,
) -> Reservation:
    """
    Create a new reservation with validation.

    Args:
        site: Site for multi-tenant isolation
        accommodation_type: Type of accommodation
        guest_data: Guest information (name, email, phone)
        dates: Check-in and check-out dates

    Returns:
        Created Reservation instance

    Raises:
        ValidationError: If dates invalid or accommodation unavailable
    """
    check_in = dates["check_in"]
    check_out = dates["check_out"]

    # Validate dates
    if check_in >= check_out:
        raise ValidationError("Check-in must be before check-out")

    # Check availability
    if not Reservation.check_availability(accommodation_type, check_in, check_out):
        raise ValidationError("Accommodation not available for selected dates")

    # Calculate pricing
    num_nights = (check_out - check_in).days
    total_price = accommodation_type.price_per_night * num_nights

    # Create reservation
    reservation = Reservation.objects.create(
        site=site,
        accommodation_type=accommodation_type,
        guest_name=guest_data["name"],
        guest_email=guest_data["email"],
        guest_phone=guest_data["phone"],
        check_in=check_in,
        check_out=check_out,
        num_nights=num_nights,
        price_per_night=accommodation_type.price_per_night,
        total_price=total_price,
        status="pending",
    )

    return reservation


# ❌ BAD: Function too long (exceeds 60 lines with all validation, business logic, and side effects mixed together)
# Instead, decompose into smaller functions:
#   - validate_reservation_dates()
#   - check_accommodation_availability()
#   - calculate_reservation_price()
#   - send_reservation_emails()
```

---

### Rule 5: Minimum Two Assertions Per Function

**Principle:** Verify pre-conditions, post-conditions, and invariants.

**Python Rules:**
- ✅ Minimum 2 assertions per function (or raise explicit exceptions)
- ✅ Validate inputs at function entry
- ✅ Verify critical assumptions
- ✅ Let exceptions bubble up (fail fast)
- ❌ NO silent failures or swallowed exceptions

**Examples:**

```python
# ✅ GOOD: Assertions for pre/post-conditions
def calculate_total_price(
    price_per_night: Decimal,
    num_nights: int,
) -> Decimal:
    """
    Calculate total reservation price.

    Args:
        price_per_night: Price per night (must be positive)
        num_nights: Number of nights (must be positive)

    Returns:
        Total price

    Raises:
        AssertionError: If inputs invalid
    """
    # Pre-condition assertions
    assert price_per_night > 0, f"Price must be positive: {price_per_night}"
    assert num_nights > 0, f"Nights must be positive: {num_nights}"
    assert num_nights <= 365, f"Nights cannot exceed 365: {num_nights}"

    total = price_per_night * num_nights

    # Post-condition assertion
    assert total > 0, f"Total price must be positive: {total}"

    return total


# ✅ GOOD: Explicit validation with exceptions (production)
def create_reservation_validated(data: dict) -> Reservation:
    """Create reservation with explicit validation."""
    # Pre-condition validations (explicit exceptions for production)
    if data["check_in"] >= data["check_out"]:
        raise ValidationError("Check-in must be before check-out")

    if data["num_guests"] < 1:
        raise ValidationError("Must have at least 1 guest")

    if data["num_guests"] > data["accommodation_type"].capacity_max:
        raise ValidationError(
            f"Guests ({data['num_guests']}) exceed capacity "
            f"({data['accommodation_type'].capacity_max})"
        )

    # Create reservation
    reservation = Reservation.objects.create(**data)

    # Post-condition validation
    if reservation.total_price <= 0:
        raise ValueError(f"Invalid total price: {reservation.total_price}")

    return reservation


# ❌ BAD: No validation, silent failures
def calculate_price(price, nights):
    return price * nights  # What if price or nights is negative? Zero? None?


# ❌ BAD: Swallowing exceptions silently
def send_email_notification(reservation):
    try:
        send_mail(...)
    except Exception:
        pass  # Silent failure! No logging, no re-raise
```

---

### Rule 6: Restrict Data Scope (Minimize Global State)

**Principle:** Limit variable scope to smallest necessary context.

**Python Rules:**
- ✅ Use class attributes for shared state (not globals)
- ✅ Use function closures for encapsulation
- ✅ Use private naming (`_name`) for internal data
- ❌ NO module-level mutable globals
- ❌ NO more than 5 public attributes per class

**Enforcement:** Wemake WPS214 (too many public attributes)

**Examples:**

```python
# ✅ GOOD: Encapsulated state in class
class ReservationManager:
    """Manager for reservation operations."""

    def __init__(self, site: Site):
        self._site = site  # Private attribute
        self._cache: dict = {}  # Private cache

    def get_available_accommodations(self, check_in: date, check_out: date):
        """Get available accommodations for dates."""
        cache_key = f"{check_in}_{check_out}"

        if cache_key in self._cache:
            return self._cache[cache_key]

        available = self._check_availability(check_in, check_out)
        self._cache[cache_key] = available
        return available

    def _check_availability(self, check_in: date, check_out: date):
        """Private helper method."""
        return AccommodationType.objects.filter(
            site=self._site,
            is_active=True,
        ).exclude(
            reservations__check_in__lt=check_out,
            reservations__check_out__gt=check_in,
            reservations__status__in=["pending", "confirmed"],
        )


# ✅ GOOD: Function closure for encapsulation
def create_price_calculator(base_price: Decimal, tax_rate: Decimal):
    """Factory function with closure."""
    def calculate_price(num_nights: int) -> Decimal:
        subtotal = base_price * num_nights
        tax = subtotal * tax_rate
        return subtotal + tax

    return calculate_price


# ❌ BAD: Module-level mutable global
GLOBAL_RESERVATION_CACHE = {}  # ❌ Mutable global state

def get_reservation(reservation_id):
    if reservation_id in GLOBAL_RESERVATION_CACHE:
        return GLOBAL_RESERVATION_CACHE[reservation_id]
    # ...


# ❌ BAD: Too many public attributes (exceeds 5)
class Site(models.Model):
    name = models.CharField(max_length=200)
    slug = models.SlugField()
    domain = models.CharField(max_length=100)
    theme = models.CharField(max_length=50)
    phone = models.CharField(max_length=50)
    email = models.EmailField()
    address = models.TextField()
    whatsapp = models.CharField(max_length=50)
    facebook_url = models.URLField()
    instagram_url = models.URLField()
    # ... 15+ more fields  # ❌ Too many! Group into related models
```

---

### Rule 7: Check All Non-Void Function Returns

**Principle:** Always validate return values and handle errors explicitly.

**Python Rules:**
- ✅ Check return values from external calls
- ✅ Let exceptions propagate (don't catch too early)
- ✅ Log exceptions before re-raising
- ❌ NO ignoring return values
- ❌ NO bare except clauses

**Examples:**

```python
# ✅ GOOD: Check return values and propagate exceptions
def send_reservation_confirmation(reservation: Reservation) -> None:
    """Send confirmation email and log result."""
    try:
        result = send_mail(
            subject=f"Reservation Confirmed - {reservation.accommodation_type.name}",
            message=render_to_string("emails/confirmation.html", {"reservation": reservation}),
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[reservation.guest_email],
            fail_silently=False,
        )

        # Check return value (send_mail returns number of emails sent)
        if result != 1:
            logger.error(
                f"Email send returned unexpected value: {result}",
                extra={"reservation_id": reservation.id}
            )

        # Log success
        logger.info(
            f"Confirmation email sent to {reservation.guest_email}",
            extra={"reservation_id": reservation.id}
        )

    except SMTPException as e:
        logger.error(
            f"Failed to send confirmation email: {e}",
            extra={"reservation_id": reservation.id},
            exc_info=True
        )
        raise  # Re-raise to caller


# ✅ GOOD: Validate API response
def fetch_external_data(api_url: str) -> dict:
    """Fetch data from external API with validation."""
    response = requests.get(api_url, timeout=10)

    # Check status code
    if not response.ok:
        raise ValueError(f"API request failed: {response.status_code}")

    # Validate response content
    try:
        data = response.json()
    except ValueError as e:
        raise ValueError(f"Invalid JSON response: {e}")

    # Validate data structure
    if not isinstance(data, dict):
        raise ValueError(f"Expected dict, got {type(data)}")

    if "results" not in data:
        raise ValueError("Missing 'results' key in response")

    return data


# ❌ BAD: Ignoring return value
def process_reservation(reservation):
    send_email(reservation.guest_email)  # ❌ Not checking if email sent
    reservation.status = "confirmed"
    reservation.save()  # ❌ Not checking if save succeeded


# ❌ BAD: Bare except clause
def unsafe_operation():
    try:
        do_something_risky()
    except:  # ❌ Catches everything, even KeyboardInterrupt, SystemExit
        pass
```

---

### Rule 8: Use Preprocessor Sparingly (Minimize Toolchain Complexity)

**Principle:** Keep build and deployment pipeline simple. Avoid custom meta-frameworks.

**Django Rules:**
- ✅ Use Django's built-in patterns (CBVs) for Django view standards
- ✅ Always use CBVs for consistency and Rule 4 compliance (methods <60 lines) for Django view standards
- ✅ Minimal frontend toolchain (DaisyUI + HTMX, no webpack)
- ✅ Document every tool in the pipeline
- ❌ NO custom meta-frameworks on top of Django
- ❌ NO custom template languages or code generators
- ❌ NO unnecessary transpilation layers

**IMPORTANT:** Django CBVs are **built-in framework patterns**, NOT a custom abstraction layer.
This rule does NOT prohibit CBVs.

**Examples:**

```python
# ✅ GOOD: Django CBV (built-in pattern, no abstraction layer)
class AccommodationListView(ListView):
    """List accommodations for site."""
    model = AccommodationType
    template_name = "accommodations/list.html"
    context_object_name = "accommodations"

    def get_queryset(self):
        """Filter by site."""
        site = get_object_or_404(Site, slug=self.kwargs["site_slug"], is_active=True)
        return AccommodationType.objects.filter(site=site, is_active=True)

    def get_context_data(self, **kwargs):
        """Add site to context."""
        context = super().get_context_data(**kwargs)
        context["site"] = get_object_or_404(
            Site, slug=self.kwargs["site_slug"], is_active=True
        )
        return context


# ❌ BAD: Custom meta-framework abstraction layer
class CustomViewFramework:
    """DON'T create custom frameworks on top of Django."""
    def __init__(self):
        self.registry = {}
        self.custom_routing = {}

    def register_view(self, name, template, model):
        """Custom routing/rendering system (unnecessary complexity)."""
        pass

    def auto_generate_crud(self, model):
        """Code generation (adds complexity without clear benefit)."""
        pass


# ❌ BAD: Custom template language
class CustomTemplateEngine:
    """DON'T create custom template languages."""
    def render(self, custom_syntax):
        # Parsing custom syntax, converting to Django templates
        pass
```

**See Also:**
- For Django view standards (when to use CBV), see `CLAUDE.md` section "Django View Standards"

---

### Rule 9: Limit Pointer Dereferencing (Avoid Deep Attribute Access)

**Principle:** Limit indirection and mutation depth.

**Python Rules:**
- ✅ Maximum 3 levels of attribute chaining
- ✅ Return new objects (immutability)
- ✅ Use type hints to catch errors early
- ❌ NO mutating function arguments
- ❌ NO deep attribute chains (a.b.c.d.e)

**Enforcement:** Wemake WPS219 (deep access), WPS220 (deep nesting)

**Examples:**

```python
# ✅ GOOD: Shallow attribute access (max 3 levels)
accommodation_name = reservation.accommodation_type.name  # 2 levels OK
site_email = reservation.site.email  # 2 levels OK


# ✅ GOOD: Extract intermediate variables for clarity
def get_site_owner_email(reservation: Reservation) -> str:
    """Get owner email with shallow access."""
    site = reservation.site
    owner = site.owner
    return owner.email  # Clear, each step visible


# ✅ GOOD: Immutable updates (return new object)
def add_guest_to_reservation(reservation: Reservation, guest_name: str) -> Reservation:
    """Create new reservation with additional guest (immutable)."""
    # Don't mutate - create new instance
    new_reservation = Reservation(
        site=reservation.site,
        accommodation_type=reservation.accommodation_type,
        guest_name=f"{reservation.guest_name}, {guest_name}",
        # ... copy other fields ...
    )
    return new_reservation


# ✅ GOOD: Django update (explicit, not mutation)
def confirm_reservation(reservation: Reservation) -> None:
    """Confirm reservation (explicit update)."""
    Reservation.objects.filter(id=reservation.id).update(
        status="confirmed",
        updated_at=timezone.now()
    )


# ❌ BAD: Deep attribute chaining (5 levels)
email = manager.filter().exclude().annotate().values().first()  # ❌ Too deep

# ❌ BAD: Deep access
owner_city = reservation.site.owner.profile.address.city  # ❌ 5 levels


# ❌ BAD: Mutating function arguments
def add_nights(reservation, extra_nights):
    """Mutate reservation in place (bad pattern)."""
    reservation.num_nights += extra_nights  # ❌ Mutation
    reservation.check_out += timedelta(days=extra_nights)  # ❌ Mutation
    reservation.save()  # ❌ Side effect


# ✅ GOOD: Return new value (functional approach)
def extend_reservation(reservation: Reservation, extra_nights: int) -> Reservation:
    """Extend reservation (return new instance)."""
    new_check_out = reservation.check_out + timedelta(days=extra_nights)
    new_num_nights = reservation.num_nights + extra_nights

    return Reservation.objects.create(
        site=reservation.site,
        accommodation_type=reservation.accommodation_type,
        guest_name=reservation.guest_name,
        guest_email=reservation.guest_email,
        check_in=reservation.check_in,
        check_out=new_check_out,
        num_nights=new_num_nights,
        # ... other fields ...
    )
```

---

### Rule 10: Compile with All Warnings Enabled

**Principle:** Treat all warnings as errors. Use multiple static analysis tools.

**Python Rules:**
- ✅ black (code formatting)
- ✅ flake8 (linting)
- ✅ mypy (type checking, strict mode)
- ✅ radon (complexity analysis)
- ✅ xenon (cyclomatic complexity)
- ✅ bandit (security scanning)
- ❌ NO ignoring warnings without documented reason
- ❌ NO disabling checks globally

**Configuration:**

```bash
# Pre-commit checks (MANDATORY)
# 1. Check for debug statements
grep -r "print(" apps/ src/
grep -r "console.log" static/js/

# 2. Format code
black apps/ src/ tests/ --line-length 88

# 3. Lint
flake8 apps/ src/ tests/ --max-line-length=88 --max-complexity=10

# 4. Type check
mypy apps/ src/ --strict --disallow-untyped-defs --disallow-any-generics

# 5. Complexity analysis
radon cc apps/ src/ -nc -a  # Cyclomatic complexity (average)
radon mi apps/ src/ -nb  # Maintainability index

# 6. Complexity thresholds (fail if exceeded)
xenon --max-absolute B --max-modules A --max-average A apps/ src/

# 7. Security scan
bandit -r apps/ src/ -ll

# 8. Run tests
python manage.py test

# 9. System check
python manage.py check --deploy
```

**pyproject.toml configuration:**

```toml
[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_any_generics = true
check_untyped_defs = true

[tool.flake8]
max-line-length = 88
max-complexity = 10
exclude = .git,__pycache__,migrations
ignore = E203,W503  # Black compatibility

[tool.radon]
cc_min = "B"  # Minimum complexity grade
```

---

## Wemake Python Styleguide - Numeric Limits

### Complexity Limits

| Metric | Limit | Wemake Code | Description |
|--------|-------|-------------|-------------|
| **Function Length** | 60 lines | WPS213 | Max lines per function (including docstring) |
| **Function Logic** | 40 expressions | WPS213 | Max expressions per function |
| **Line Complexity** | 14 AST nodes | WPS221 | Max AST nodes per line (Jones Complexity) |
| **Cognitive Complexity** | 12 | WPS231 | Max cognitive complexity score per function |
| **Cyclomatic Complexity** | 10 | - | Max McCabe complexity (flake8) |
| **Conditions per If** | 4 | WPS222 | Max logical operators (and/or) per condition |
| **Function Parameters** | 5 | WPS211 | Max parameters per function signature |
| **Class Attributes** | 6 public | WPS214 | Max public attributes per class |
| **Base Classes** | 3 | WPS215 | Max inheritance (base classes) |
| **Module Members** | 7 public | WPS202 | Max public members in module |
| **Imports per Module** | 12 | WPS235 | Max import statements per file |
| **Try Body Length** | 1 expression | WPS229 | Only one operation in try block |
| **Attribute Access Depth** | 3 levels | WPS219 | Max attribute chaining (a.b.c) |
| **Nested Functions** | 3 levels | WPS220 | Max nesting level for functions |
| **For Loops in Comprehension** | 2 | WPS224 | Max nested for in comprehension |
| **Type Annotation Complexity** | 3 | WPS234 | Max nesting in type hints |

### Enforcement Commands

```bash
# Install wemake-python-styleguide
pip install wemake-python-styleguide

# Run all checks (includes flake8 + wemake rules)
flake8 apps/ src/ tests/

# Specific complexity checks
radon cc apps/ -nc  # Cyclomatic complexity
radon mi apps/ -nb  # Maintainability index
xenon --max-absolute B --max-modules A apps/

# Complexity report
radon cc apps/ -s -a  # Show average and summary
```

---

## Naming Conventions

### Python (PEP 8 + Django)

```python
# Variables and functions: snake_case
accommodation_type = AccommodationType.objects.get(id=1)
def check_availability(check_in, check_out): ...

# Classes: PascalCase
class ReservationManager: ...
class AccommodationType(models.Model): ...

# Constants: UPPER_SNAKE_CASE
MAX_GUESTS = 10
DEFAULT_PAGE_SIZE = 20

# Private attributes/methods: _leading_underscore
def _validate_dates(self): ...
self._cache = {}

# Django models: PascalCase with singular names
class Reservation(models.Model): ...  # Not "Reservations"

# Django views: snake_case verbs
def create_reservation(request): ...
def list_accommodations(request): ...

# URL patterns: kebab-case
path('check-availability/', check_availability_view)
path('reservations/new/', create_reservation)

# Template files: snake_case.html
accommodation_list.html
reservation_form.html

# Type hints: Use built-in generics (Python 3.9+)
from collections.abc import Sequence
def process_items(items: Sequence[int]) -> list[str]: ...
```

---

## Google-Style Docstrings (Required)

### Function Docstring Template

```python
def create_reservation(
    site: Site,
    accommodation_type: AccommodationType,
    guest_data: dict,
    check_in: date,
    check_out: date,
) -> Reservation:
    """
    Create a new reservation with validation.

    This function validates dates, checks availability, calculates pricing,
    and creates a pending reservation in the database. Multi-tenant isolation
    is enforced via the site parameter.

    Args:
        site: Site instance for multi-tenant isolation (required)
        accommodation_type: Type of accommodation to reserve
        guest_data: Dictionary containing guest information:
            - name (str): Full name of guest
            - email (str): Valid email address
            - phone (str): Contact phone number
        check_in: Check-in date (must be before check_out)
        check_out: Check-out date (must be after check_in)

    Returns:
        Reservation: Created reservation instance with status="pending"

    Raises:
        ValidationError: If check_in >= check_out
        ValidationError: If accommodation not available for dates
        ValidationError: If guest_data is invalid
        IntegrityError: If database constraints violated

    Examples:
        >>> site = Site.objects.get(slug="ribera-calma")
        >>> accommodation = AccommodationType.objects.get(slug="cabana-2")
        >>> guest = {"name": "John Doe", "email": "john@example.com", "phone": "+54..."}
        >>> reservation = create_reservation(
        ...     site=site,
        ...     accommodation_type=accommodation,
        ...     guest_data=guest,
        ...     check_in=date(2025, 12, 1),
        ...     check_out=date(2025, 12, 5)
        ... )
        >>> reservation.status
        'pending'

    Note:
        This function does NOT send confirmation emails. Call
        send_reservation_confirmation() separately after creation.

    See Also:
        - check_availability(): Check if dates are available
        - calculate_reservation_price(): Calculate total price
        - send_reservation_confirmation(): Send confirmation email
    """
    # Implementation...
```

### Class Docstring Template

```python
class ReservationManager:
    """
    Manager for reservation-related operations.

    This class encapsulates business logic for creating, validating, and
    managing reservations. It handles availability checking, pricing
    calculation, and multi-tenant filtering.

    Attributes:
        site: Site instance for multi-tenant filtering
        _cache: Private cache for availability queries (dict)

    Examples:
        >>> manager = ReservationManager(site=my_site)
        >>> available = manager.get_available_accommodations(
        ...     check_in=date(2025, 12, 1),
        ...     check_out=date(2025, 12, 5)
        ... )
    """

    def __init__(self, site: Site):
        """
        Initialize reservation manager.

        Args:
            site: Site instance for multi-tenant operations
        """
        self.site = site
        self._cache = {}
```

---

## Type Hints (Strict Mode Required)

### Basic Type Hints

```python
from typing import Optional, Union, Any
from collections.abc import Sequence, Mapping
from decimal import Decimal
from datetime import date, datetime

# Primitives
def calculate_price(price: Decimal, nights: int) -> Decimal: ...
def format_name(name: str) -> str: ...
def is_available(check_in: date) -> bool: ...

# Optional (use | None in Python 3.10+)
def get_reservation(id: int) -> Reservation | None: ...

# Collections (use built-in generics Python 3.9+)
def process_items(items: list[int]) -> list[str]: ...
def get_mapping(data: dict[str, int]) -> dict[str, str]: ...

# Django Models
def get_site(slug: str) -> Site: ...
def filter_reservations(site: Site) -> QuerySet[Reservation]: ...

# Complex types (use type aliases for readability)
GuestData = dict[str, str]  # Type alias
def create_guest(data: GuestData) -> Guest: ...

# Callables
from collections.abc import Callable
def apply_transform(data: list[int], fn: Callable[[int], str]) -> list[str]: ...
```

### Type Aliases (Wemake WPS234 - Simplify Complex Annotations)

```python
# ✅ GOOD: Use type aliases for complex types
from typing import TypeAlias

ReservationData: TypeAlias = dict[str, Union[str, int, date, Decimal]]
AccommodationList: TypeAlias = list[AccommodationType]
ValidationResult: TypeAlias = tuple[bool, Optional[str]]

def validate_reservation(data: ReservationData) -> ValidationResult:
    """Validate reservation data."""
    if "check_in" not in data:
        return False, "Missing check_in"
    return True, None


# ❌ BAD: Complex nested annotations without aliases
def process_data(
    data: dict[str, Union[list[tuple[int, str]], dict[str, Optional[Decimal]]]]
) -> list[tuple[str, Union[int, None]]]:  # Too complex!
    ...
```

---

## Error Handling Best Practices

### Fail Fast Philosophy

```python
# ✅ GOOD: Explicit validation, fail fast
def create_reservation(data: dict) -> Reservation:
    """Create reservation with explicit validation."""
    # Validate early - fail fast
    if not data.get("guest_email"):
        raise ValidationError("Email is required")

    if not isinstance(data.get("check_in"), date):
        raise TypeError("check_in must be a date object")

    if data["check_in"] >= data["check_out"]:
        raise ValidationError("Check-in must be before check-out")

    # Main logic only runs if validation passed
    return Reservation.objects.create(**data)


# ❌ BAD: Silent failures, late validation
def create_reservation(data):
    try:
        reservation = Reservation.objects.create(**data)
        if reservation:
            return reservation
    except Exception:
        pass  # Silent failure!
    return None  # Ambiguous: success or failure?
```

### Exception Hierarchy

```python
# ✅ GOOD: Specific exception types
from django.core.exceptions import ValidationError

def check_availability(accommodation, check_in, check_out):
    """Check availability with specific exceptions."""
    if check_in >= check_out:
        raise ValidationError("Invalid date range")

    if not accommodation.is_active:
        raise ValueError("Accommodation is not active")

    if Reservation.objects.filter(...).exists():
        raise ValidationError("Dates not available")

    return True


# Use try/except only when you can handle the error
def safe_create_reservation(data):
    """Create reservation with error handling."""
    try:
        return create_reservation(data)
    except ValidationError as e:
        logger.error(f"Validation failed: {e}")
        raise  # Re-raise to caller
    except IntegrityError as e:
        logger.error(f"Database integrity error: {e}", exc_info=True)
        raise
```

---

## Pre-Commit Hooks Configuration

Create `.pre-commit-config.yaml`:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.1.1
    hooks:
      - id: black
        language_version: python3.11

  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
        args: [--max-line-length=88, --max-complexity=10]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        args: [--strict, --disallow-untyped-defs]
        additional_dependencies: [django-stubs]

  - repo: local
    hooks:
      - id: check-debug-statements
        name: Check for debug statements
        entry: bash -c 'grep -r "print(" apps/ src/ && exit 1 || exit 0'
        language: system
        pass_filenames: false

      - id: radon-complexity
        name: Check cyclomatic complexity
        entry: radon cc apps/ src/ -nc -a
        language: system
        pass_filenames: false

      - id: xenon-complexity
        name: Check complexity thresholds
        entry: xenon --max-absolute B --max-modules A apps/ src/
        language: system
        pass_filenames: false

      - id: bandit-security
        name: Security scan with bandit
        entry: bandit -r apps/ src/ -ll
        language: system
        pass_filenames: false

      - id: django-tests
        name: Run Django tests
        entry: python manage.py test
        language: system
        pass_filenames: false
```

Install pre-commit:
```bash
pip install pre-commit
pre-commit install
```

---

## Summary Checklist

Before committing code, verify:

- [ ] **Rule 1:** No complex flow (recursion, nested loops)
- [ ] **Rule 2:** All loops have fixed bounds with assertions
- [ ] **Rule 3:** Memory bounded (pagination, field limits, iterators)
- [ ] **Rule 4:** Functions ≤ 60 lines
- [ ] **Rule 5:** Minimum 2 assertions per function
- [ ] **Rule 6:** No mutable globals, max 5 public attributes per class
- [ ] **Rule 7:** All return values checked, exceptions logged
- [ ] **Rule 8:** Simple toolchain (no unnecessary transpilers)
- [ ] **Rule 9:** Max 3 levels attribute access, no argument mutation
- [ ] **Rule 10:** All tools pass (black, flake8, mypy, radon, xenon, bandit)

**Complexity Limits:**
- [ ] Line complexity ≤ 14 AST nodes
- [ ] Cognitive complexity ≤ 12
- [ ] Cyclomatic complexity ≤ 10
- [ ] Max 4 conditions per if statement
- [ ] Max 5 parameters per function
- [ ] Max 3 base classes per class

**Quality:**
- [ ] Type hints on all functions
- [ ] Google-style docstrings
- [ ] No debug statements (print, console.log)
- [ ] Tests passing (70% coverage for Standard Mode)

---

## References

1. **NASA JPL Power of 10**: https://spinroot.com/gerard/pdf/P10.pdf
2. **Wemake Python Styleguide**: https://wemake-python-styleguide.readthedocs.io
3. **Google Python Style Guide**: https://google.github.io/styleguide/pyguide.html
4. **PEP 8**: https://peps.python.org/pep-0008/
5. **NASA applied to Python**: https://dev.to/xowap/10-rules-to-code-like-nasa-applied-to-interpreted-languages-40dd

---

**Philosophy:** "Simplicity is the ultimate sophistication. Code that fits in your head is code you can maintain."
