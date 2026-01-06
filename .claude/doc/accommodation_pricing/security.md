# accommodation_pricing/security.md - Dynamic Pricing Security Analysis

## 1. Threat Model

### Attack Surface

**High Risk Vectors:**
- **Price Manipulation**: Attackers attempt to manipulate `num_guests` or discount values to obtain incorrect (lower) prices
- **Multi-tenant Data Leakage**: User A attempts to access/modify User B's pricing configurations
- **Discount Bypass**: Attackers try to create bookings without applying configured discounts
- **Admin Privilege Escalation**: Regular users attempt to access admin pricing configuration

**Medium Risk Vectors:**
- **Input Validation Bypass**: Malformed input to price calculation API (negative values, SQL injection attempts)
- **Race Conditions**: Concurrent modifications to pricing records
- **Decimal Precision Exploitation**: Rounding errors in price calculations

**Low Risk Vectors:**
- **Information Disclosure**: Exposure of pricing strategy through API responses
- **DoS via Price Calculation**: High-frequency requests to price preview endpoint

---

## 2. Multi-tenant Isolation

### Database-Level Isolation

**FK Chain:**
```
AccommodationPricesPerPerson → AccommodationType → Site
```

**CRITICAL: All queries MUST filter by site:**

```python
# CORRECT
def get_queryset(self, request):
    qs = super().get_queryset(request)
    if request.user.is_superuser:
        return qs

    return qs.filter(
        Q(accommodation__site__owner=request.user) |
        Q(accommodation__site__admins=request.user)
    ).distinct()

# INCORRECT
def get_queryset(self, request):
    return AccommodationPricesPerPerson.objects.all()  # VULNERABLE
```

### Admin Interface Isolation

**Required Implementations:**

```python
@admin.register(AccommodationPricesPerPerson)
class AccommodationPricesPerPersonAdmin(admin.ModelAdmin):

    # 1. Filter queryset by site ownership
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if request.user.is_superuser:
            return qs
        return qs.filter(
            Q(accommodation__site__owner=request.user) |
            Q(accommodation__site__admins=request.user)
        ).distinct()

    # 2. Filter accommodation choices in forms
    def formfield_for_foreignkey(self, db_field, request, **kwargs):
        if db_field.name == "accommodation":
            if not request.user.is_superuser:
                kwargs["queryset"] = AccommodationType.objects.filter(
                    Q(site__owner=request.user) |
                    Q(site__admins=request.user)
                ).distinct()
        return super().formfield_for_foreignkey(db_field, request, **kwargs)

    # 3. Check change permission
    def has_change_permission(self, request, obj=None):
        if not super().has_change_permission(request, obj):
            return False
        if obj is None:
            return True
        if request.user.is_superuser:
            return True
        site = obj.accommodation.site
        return site.owner == request.user or request.user in site.admins.all()

    # 4. Check delete permission
    def has_delete_permission(self, request, obj=None):
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

## 3. Input Validation

### Field-Level Validation

**num_guests (User Input)**

```python
def validate_num_guests(value: int, accommodation: AccommodationType) -> None:
    """Validate num_guests input"""

    if not isinstance(value, int):
        raise ValidationError("num_guests must be an integer")

    if value < 1:
        raise ValidationError("num_guests must be at least 1")

    if value > accommodation.capacity_max:
        raise ValidationError(
            f"num_guests cannot exceed accommodation capacity ({accommodation.capacity_max})"
        )
```

**discount (Admin Input)**

```python
class AccommodationPricesPerPerson(models.Model):
    discount = models.PositiveIntegerField(
        default=0,
        validators=[MinValueValidator(0)]  # Cannot be negative
    )

    def clean(self):
        """Model-level validation"""
        super().clean()

        # Discount cannot exceed 100 for percentage mode
        if self.mode == '%' and self.discount > 100:
            raise ValidationError({
                'discount': 'Percentage discount cannot exceed 100%'
            })

        # Active records must have valid discount
        if self.is_active and self.discount is None:
            raise ValidationError({
                'is_active': 'Cannot activate record without discount value'
            })
```

---

## 4. Price Calculation Security

### Negative Price Prevention

```python
def calculate_discounted_price(self, base_price: Decimal) -> Decimal:
    """Calculate discounted price with negative value protection"""

    if base_price < 0:
        raise ValidationError("Base price cannot be negative")

    if self.discount == 0:
        return base_price

    if self.mode == '%':
        discount_amount = base_price * Decimal(self.discount) / Decimal(100)
        final_price = base_price - discount_amount
    else:
        final_price = base_price - Decimal(self.discount)

    # CRITICAL: Ensure price never goes negative
    if final_price < 0:
        logger.warning(
            f"Price calculation resulted in negative value. Clamping to 0."
        )
        return Decimal('0.00')

    return final_price.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
```

### Decimal Precision Handling

```python
from decimal import Decimal, ROUND_HALF_UP

# CORRECT - Use Decimal for currency
price_per_night = Decimal('150.00')
discount = Decimal('20')
discounted_price = price_per_night * (Decimal('100') - discount) / Decimal('100')
discounted_price = discounted_price.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)

# INCORRECT - Float introduces rounding errors
price_per_night = 150.00  # ❌
```

---

## 5. Admin Interface Security

### Capacity_max Record Protection

```python
def save(self, *args, **kwargs):
    """Override save to enforce business rules"""

    is_capacity_max = (self.capacity == self.accommodation.capacity_max)

    if is_capacity_max and self.pk:
        try:
            old_instance = AccommodationPricesPerPerson.objects.get(pk=self.pk)
            if old_instance.discount != self.discount:
                raise ValidationError(
                    f"Cannot modify discount for capacity_max ({self.capacity}). "
                    f"This is the base price."
                )
        except AccommodationPricesPerPerson.DoesNotExist:
            pass

    super().save(*args, **kwargs)
```

---

## 6. API Endpoint Security

### Rate Limiting

```python
def rate_limit(key: str, limit: int, period: int) -> bool:
    """Simple rate limiting using cache"""
    cache_key = f'rate_limit:{key}'
    requests = cache.get(cache_key, 0)

    if requests >= limit:
        return False

    cache.set(cache_key, requests + 1, period)
    return True

@api_view(['GET'])
def price_preview(request, site_slug):
    """Price preview with rate limiting"""

    # Rate limiting: 100 requests per minute per IP
    client_ip = request.META.get('REMOTE_ADDR')
    if not rate_limit(f'price_preview_{client_ip}', limit=100, period=60):
        return HttpResponseTooManyRequests("Rate limit exceeded")

    # ... calculation logic ...
```

---

## 7. Signal/Auto-creation Security

### Atomic Transactions

```python
@receiver(post_save, sender=AccommodationType)
@transaction.atomic
def auto_create_pricing_records(sender, instance, created, **kwargs):
    """Auto-create pricing records with atomic transaction"""

    if not created:
        return

    try:
        capacity_range = range(instance.capacity_min, instance.capacity_max + 1)
        records = [
            AccommodationPricesPerPerson(
                accommodation=instance,
                capacity=capacity,
                discount=0,
                is_active=True
            )
            for capacity in capacity_range
        ]

        AccommodationPricesPerPerson.objects.bulk_create(records)

    except Exception as e:
        logger.error(f"Failed to auto-create pricing records: {e}", exc_info=True)
        raise  # Transaction automatically rolled back
```

---

## 8. OWASP Top 10 Analysis

### A01: Broken Access Control

**Mitigations:**
- ✅ Admin get_queryset() filters by site ownership
- ✅ Admin has_change_permission() validates site access
- ✅ API endpoint validates site ownership

### A03: Injection

**Mitigations:**
- ✅ ORM queries only (no raw SQL)
- ✅ Parameterized queries via Django ORM
- ✅ Type validation before queries

### A04: Insecure Design

**Mitigations:**
- ✅ Model clean() validates all fields
- ✅ save() enforces business rules
- ✅ calculate_discounted_price() clamps to 0

---

## 9. Test Cases for Security

### Multi-tenant Isolation Tests

```python
def test_user_cannot_access_other_site_pricing_admin(self):
    """User A cannot access User B's pricing in admin"""
    self.client.login(username='user_a', password='pass')

    response = self.client.get(
        f'/admin/accommodations/accommodationpricesperperson/{self.pricing_b.pk}/change/'
    )

    self.assertIn(response.status_code, [403, 404])
```

### Input Validation Tests

```python
def test_negative_discount_rejected(self):
    """Cannot create pricing with negative discount"""
    with self.assertRaises(ValidationError):
        pricing = AccommodationPricesPerPerson(
            accommodation=self.accommodation,
            capacity=2,
            discount=-10,
            mode='%'
        )
        pricing.full_clean()
```

### Price Calculation Tests

```python
def test_price_never_negative(self):
    """Price calculation never returns negative value"""
    pricing = AccommodationPricesPerPerson.objects.create(
        accommodation=self.accommodation,
        capacity=2,
        discount=300,  # Exceeds base price
        mode='Fijo'
    )

    base_price = Decimal('200.00')
    final_price = pricing.calculate_discounted_price(base_price)

    self.assertEqual(final_price, Decimal('0.00'))  # Clamped to 0
```

---

## Security Checklist

**Before Deployment:**

- [ ] All queries filter by site (multi-tenant isolation)
- [ ] Admin get_queryset() uses Q objects
- [ ] Admin has_change_permission() validates site ownership
- [ ] API endpoint validates site ownership
- [ ] Input validation: num_guests, discount, capacity, mode
- [ ] Price calculation clamps negative values to 0
- [ ] Decimal precision maintained (2 decimal places)
- [ ] capacity_max discount modification blocked
- [ ] Signal auto-creation uses atomic transactions
- [ ] Rate limiting on API endpoint (100 req/min)
- [ ] CSRF protection on all forms
- [ ] Test coverage: multi-tenant isolation ≥80%
- [ ] Test coverage: input validation ≥90%
- [ ] Test coverage: price calculation 100%

---

## Conclusion

The dynamic pricing system introduces **moderate security risk** through:
1. Multi-tenant data leakage (HIGH RISK)
2. Price manipulation through input validation bypass (MEDIUM RISK)
3. Admin privilege escalation (MEDIUM RISK)

**All risks are mitigated through:**
- Comprehensive multi-tenant filtering
- Multiple validation layers
- Business logic enforcement
- Secure price calculation
- Rate limiting and logging

**Test coverage requirements:**
- Multi-tenant isolation: ≥80%
- Input validation: ≥90%
- Price calculation: 100%
