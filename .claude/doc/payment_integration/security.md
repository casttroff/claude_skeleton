# Security Research: Payment Integration (MercadoPago)

## Overview

Complete security threat analysis for dual-revenue payment system with MercadoPago integration. This document covers OWASP Top 10 threats, payment-specific attack vectors, multi-tenant isolation, and PCI DSS compliance requirements.

**System Components**:
1. **Subscription Payments** - Recurring monthly billing (to platform)
2. **Reservation Payments** - One-time split payments (95% to site owner, 5% to platform)

**Security Level**: CRITICAL - Handles sensitive financial transactions, must comply with PCI DSS SAQ A

---

## OWASP Top 10 Coverage

### A01: Broken Access Control

**Threat**: Multi-tenant data leakage allowing users to access/modify other users' subscriptions or site reservations

#### Subscription-Level Threats

**Threat 1.1: Unauthorized Subscription Access**
```python
# ❌ VULNERABLE - No user check
def subscription_view(request):
    subscription = Subscription.objects.get(id=request.GET['id'])
    return render(request, 'subscription.html', {'subscription': subscription})

# ✅ SECURE - User ownership check
def subscription_view(request):
    subscription = get_object_or_404(Subscription, id=request.GET['id'], user=request.user)
    return render(request, 'subscription.html', {'subscription': subscription})
```

**Attack Vector**:
- User A manipulates URL parameter: `?id=123` → `?id=456` (User B's subscription)
- User A can view/modify User B's payment information

**Mitigation**:
```python
# apps/subscriptions/mixins.py

class SubscriptionOwnerRequiredMixin(LoginRequiredMixin):
    """Ensure user can only access their own subscription."""

    def dispatch(self, request, *args, **kwargs):
        subscription = self.get_object()

        if subscription.user != request.user:
            raise PermissionDenied("You cannot access this subscription")

        return super().dispatch(request, *args, **kwargs)
```

**Test Case**:
```python
def test_user_cannot_access_other_subscription(self):
    """User A cannot view User B's subscription"""
    user_a = User.objects.create_user('user_a', 'a@test.com', 'pass')
    user_b = User.objects.create_user('user_b', 'b@test.com', 'pass')

    subscription_b = Subscription.objects.create(user=user_b, plan=plan)

    self.client.login(username='user_a', password='pass')
    response = self.client.get(f'/subscriptions/{subscription_b.id}/')

    self.assertEqual(response.status_code, 403)
```

---

**Threat 1.2: Site-Level Reservation Access (Multi-tenant)**
```python
# ❌ VULNERABLE - No site ownership check
def reservation_view(request, reservation_id):
    reservation = Reservation.objects.get(id=reservation_id)
    return render(request, 'reservation.html', {'reservation': reservation})

# ✅ SECURE - Site ownership check
def reservation_view(request, site_slug, reservation_id):
    site = get_object_or_404(Site, slug=site_slug)

    # Check user owns or manages this site
    if not (site.owner == request.user or request.user in site.admins.all()):
        raise PermissionDenied

    # Filter by site
    reservation = get_object_or_404(Reservation, id=reservation_id, site=site)
    return render(request, 'reservation.html', {'reservation': reservation})
```

**Attack Vector**:
- Admin A manages Site X
- Admin A manipulates URL to access reservations from Site Y
- Admin A can view payment details, guest information from unauthorized site

**Mitigation Pattern**:
```python
# ALWAYS filter by site in reservation queries
reservations = Reservation.objects.filter(site=request.site)

# ALWAYS check site ownership before operations
def check_site_permission(user, site):
    """Check if user can manage this site."""
    return site.owner == user or user in site.admins.all()
```

---

**Threat 1.3: SubscriptionRequiredMixin Bypass**
```python
# ❌ VULNERABLE - Missing subscription check
class SiteCreateView(LoginRequiredMixin, CreateView):
    model = Site
    # User can create site without active subscription

# ✅ SECURE - Subscription check enforced
class SiteCreateView(SubscriptionRequiredMixin, CreateView):
    model = Site
    # Redirects to /subscriptions/plans/ if no active subscription
```

**Attack Vector**:
- User signs up (trial starts)
- User cancels trial immediately
- User continues using platform without payment (subscription status = "cancelled")
- User creates sites, accommodations, reservations without valid subscription

**Mitigation**:
```python
# apps/subscriptions/mixins.py

class SubscriptionRequiredMixin(LoginRequiredMixin):
    """
    Block access if subscription inactive.

    Apply to:
    - SiteCreateView, SiteUpdateView, SiteDeleteView
    - AccommodationCreateView, AccommodationUpdateView
    - All Site-related operations
    """

    def dispatch(self, request, *args, **kwargs):
        # Check if user has subscription
        if not hasattr(request.user, "subscription"):
            messages.warning(request, "Necesitas una suscripción para acceder.")
            return redirect("subscriptions:select-plan")

        subscription = request.user.subscription

        # Check if subscription is active (includes grace period)
        if not subscription.is_active:
            if subscription.status == "suspended":
                messages.error(request, "Tu suscripción está suspendida.")
                return redirect("subscriptions:suspended")
            elif subscription.status == "cancelled":
                messages.warning(request, "Tu suscripción ha sido cancelada.")
                return redirect("subscriptions:reactivate")

        # Check plan limits (before creating new resources)
        if self.request.method == "POST":
            if not self._check_plan_limits(request.user, subscription.plan):
                messages.error(request, f"Alcanzaste el límite de tu plan.")
                return redirect("subscriptions:upgrade")

        return super().dispatch(request, *args, **kwargs)
```

**Test Cases**:
```python
def test_suspended_user_cannot_create_site(self):
    """User with suspended subscription cannot create site"""
    user = User.objects.create_user('user', 'user@test.com', 'pass')
    subscription = Subscription.objects.create(
        user=user,
        plan=plan,
        status="suspended"
    )

    self.client.login(username='user', password='pass')
    response = self.client.post('/sites/create/', {...})

    self.assertEqual(response.status_code, 302)
    self.assertRedirects(response, '/subscriptions/suspended/')

def test_plan_limit_enforced(self):
    """User cannot exceed plan limits"""
    user = User.objects.create_user('user', 'user@test.com', 'pass')
    plan_basic = Plan.objects.create(tier="basic", max_sites=1, ...)
    subscription = Subscription.objects.create(user=user, plan=plan_basic, status="active")

    # Create first site (allowed)
    site1 = Site.objects.create(owner=user, name="Site 1")

    # Attempt second site (blocked)
    self.client.login(username='user', password='pass')
    response = self.client.post('/sites/create/', {'name': 'Site 2', ...})

    self.assertEqual(response.status_code, 302)
    self.assertRedirects(response, '/subscriptions/upgrade/')
    self.assertEqual(Site.objects.filter(owner=user).count(), 1)
```

---

### A02: Cryptographic Failures

**Threat**: Exposure of card data, payment tokens, or credentials

#### PCI DSS Compliance Level

**Classification**: SAQ A (Merchant handles NO card data)

**Requirements**:
- ✅ NO card numbers stored locally (MercadoPago tokenization)
- ✅ NO CVV stored (MercadoPago handles)
- ✅ NO card expiry stored
- ✅ Payment tokens encrypted at rest (if stored)
- ✅ HTTPS enforced for all transactions
- ✅ Webhook secrets stored securely

---

**Threat 2.1: MercadoPago Token Storage**
```python
# ❌ VULNERABLE - Token stored in plaintext
class Subscription(models.Model):
    mercadopago_payer_id = models.CharField(max_length=100)  # Plaintext

# ✅ SECURE - Token encrypted with Fernet
from cryptography.fernet import Fernet
from django.conf import settings

class Subscription(models.Model):
    _mercadopago_payer_id_encrypted = models.BinaryField(blank=True)

    @property
    def mercadopago_payer_id(self) -> str:
        """Decrypt payer ID when accessed."""
        if not self._mercadopago_payer_id_encrypted:
            return ""

        cipher = Fernet(settings.MERCADOPAGO_ENCRYPTION_KEY.encode())
        return cipher.decrypt(self._mercadopago_payer_id_encrypted).decode()

    @mercadopago_payer_id.setter
    def mercadopago_payer_id(self, value: str):
        """Encrypt payer ID when saved."""
        if not value:
            self._mercadopago_payer_id_encrypted = b""
            return

        cipher = Fernet(settings.MERCADOPAGO_ENCRYPTION_KEY.encode())
        self._mercadopago_payer_id_encrypted = cipher.encrypt(value.encode())
```

**Environment Variable**:
```bash
# .env
MERCADOPAGO_ENCRYPTION_KEY=your_32_byte_key_here  # Generate with Fernet.generate_key()
```

**Test Case**:
```python
def test_mercadopago_payer_id_encrypted_at_rest(self):
    """MercadoPago payer ID encrypted in database"""
    subscription = Subscription.objects.create(
        user=user,
        plan=plan,
        mercadopago_payer_id="123456789"
    )

    # Check database value is encrypted (not plaintext)
    subscription.refresh_from_db()
    raw_value = subscription._mercadopago_payer_id_encrypted
    assert raw_value != b"123456789"
    assert b"123456789" not in raw_value

    # Check decryption works
    assert subscription.mercadopago_payer_id == "123456789"
```

---

**Threat 2.2: Webhook Secret Exposure**
```python
# ❌ VULNERABLE - Hardcoded secret
MERCADOPAGO_WEBHOOK_SECRET = "my_secret_123"

# ✅ SECURE - Environment variable
MERCADOPAGO_WEBHOOK_SECRET = os.getenv("MERCADOPAGO_WEBHOOK_SECRET")

# Verify secret is loaded
if not MERCADOPAGO_WEBHOOK_SECRET:
    raise ImproperlyConfigured("MERCADOPAGO_WEBHOOK_SECRET not set")
```

**Attack Vector**:
- Attacker finds webhook secret in version control
- Attacker crafts valid HMAC signatures
- Attacker sends fake webhook events (payment approved without actual payment)

**Mitigation**:
- Store in environment variables
- Rotate webhook secret every 90 days
- Log all webhook events with signature validation result

---

### A03: Injection (SQL Injection)

**Threat**: SQL injection via payment-related queries

#### SQL Injection Prevention

**Threat 3.1: Payment ID Injection**
```python
# ❌ VULNERABLE - Raw SQL with string formatting
def get_payment_log(payment_id):
    query = f"SELECT * FROM payment_log WHERE mercadopago_payment_id = '{payment_id}'"
    cursor.execute(query)
    return cursor.fetchone()

# ✅ SECURE - Django ORM (parameterized queries)
def get_payment_log(payment_id):
    return PaymentLog.objects.get(mercadopago_payment_id=payment_id)
```

**Attack Vector**:
- Webhook sends malicious payment_id: `123' OR '1'='1`
- Raw SQL executes: `SELECT * FROM payment_log WHERE mercadopago_payment_id = '123' OR '1'='1'`
- Returns all payment logs from all sites

**Mitigation**:
```python
# RULE: NEVER use raw SQL in payment code

# ✅ Always use Django ORM
PaymentLog.objects.filter(mercadopago_payment_id=payment_id)

# ✅ If raw SQL required, use parameterized queries
cursor.execute(
    "SELECT * FROM payment_log WHERE mercadopago_payment_id = %s",
    [payment_id]
)
```

---

**Threat 3.2: Webhook Data Injection**
```python
# ❌ VULNERABLE - Unvalidated webhook data used in query
def process_webhook(request):
    data = json.loads(request.body)
    external_ref = data['external_reference']  # No validation

    # Direct use in query (vulnerable if raw SQL)
    subscription = Subscription.objects.raw(
        f"SELECT * FROM subscription WHERE id = {external_ref}"
    )

# ✅ SECURE - Pydantic validation + ORM
from pydantic import BaseModel, validator

class WebhookPayload(BaseModel):
    external_reference: str

    @validator('external_reference')
    def validate_external_reference(cls, v):
        # Must be UUID or integer
        if not v.isdigit():
            raise ValueError("Invalid external_reference format")
        return v

def process_webhook(request):
    try:
        payload = WebhookPayload(**json.loads(request.body))
    except ValidationError as e:
        return JsonResponse({"error": str(e)}, status=400)

    # Safe to use with ORM
    subscription = Subscription.objects.get(id=payload.external_reference)
```

---

### A04: Insecure Design

**Threat**: Design flaws in payment flow allowing race conditions, double-spending, or state corruption

#### Race Condition Prevention

**Threat 4.1: Double-Spending (Concurrent Reservations)**
```python
# ❌ VULNERABLE - No lock, race condition possible
def create_reservation(accommodation, check_in, check_out):
    # Thread 1 and Thread 2 both check availability
    available = accommodation.get_available_units(check_in, check_out)

    if available >= 1:
        # Both threads see available=1, both proceed
        reservation = Reservation.objects.create(...)  # Overbooking!

# ✅ SECURE - select_for_update() prevents race condition
from django.db import transaction

@transaction.atomic
def create_reservation_with_lock(accommodation, check_in, check_out, **kwargs):
    """
    Create reservation with database lock to prevent overbooking.

    Uses select_for_update() to lock accommodation row until transaction commits.
    """
    # Lock accommodation row (blocks other transactions)
    accommodation = AccommodationType.objects.select_for_update().get(
        id=accommodation.id
    )

    # Check availability (within lock)
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

**Test Case**:
```python
def test_concurrent_bookings_prevented(self):
    """Test race condition prevention with select_for_update"""
    import threading
    from django.db import transaction

    accommodation = AccommodationType.objects.create(
        site=site,
        quantity=1,  # Only 1 unit
        name="Cabaña",
        price_per_night=100
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
                    guest_email="test@example.com",
                    guest_phone="123456",
                    num_guests=2,
                    total_price=400,
                    site=site
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
    assert len(successful) == 1  # Only 1 reservation created
```

---

**Threat 4.2: Grace Period Retry Logic Failure**
```python
# ❌ VULNERABLE - Infinite retry loop possible
@shared_task(max_retries=None)  # No limit!
def retry_failed_subscription_payment(subscription_id):
    subscription = Subscription.objects.get(id=subscription_id)

    if subscription.status == "payment_failed":
        success = SubscriptionService.retry_payment(subscription)

        if not success:
            raise self.retry()  # Retry forever

# ✅ SECURE - Exponential backoff with max retries
@shared_task(
    bind=True,
    max_retries=5,
    retry_backoff=True,
    default_retry_delay=21600  # 6 hours base
)
def retry_failed_subscription_payment(self, subscription_id):
    subscription = Subscription.objects.get(id=subscription_id)

    # Check if already resolved
    if subscription.status != "payment_failed":
        return

    # Check grace period (7 days)
    if timezone.now() > subscription.grace_period_ends_at:
        # Suspend after grace period
        subscription.status = "suspended"
        subscription.save()
        send_suspension_email.delay(subscription.id)
        return

    # Retry payment
    success = SubscriptionService.retry_payment(subscription)

    if not success:
        subscription.payment_retry_count += 1
        subscription.save()

        # Retry with exponential backoff (up to 5 attempts)
        if subscription.payment_retry_count < 5:
            raise self.retry(countdown=21600)  # 6h, 12h, 24h, 48h, 96h
```

---

### A05: Security Misconfiguration

**Threat**: Misconfigured settings exposing sensitive data or bypassing security

#### Webhook Security

**Threat 5.1: Missing HMAC Signature Verification**
```python
# ❌ VULNERABLE - No signature verification
@csrf_exempt
def subscription_webhook(request):
    data = json.loads(request.body)
    # Process webhook without verification
    # Attacker can send fake "payment approved" webhooks

# ✅ SECURE - HMAC SHA256 signature verification
import hmac
import hashlib

def verify_mercadopago_webhook(request) -> bool:
    """Verify MercadoPago webhook HMAC signature."""
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

@csrf_exempt
def subscription_webhook(request):
    # Verify signature first (CRITICAL)
    if not verify_mercadopago_webhook(request):
        logger.warning("Invalid webhook signature")
        return JsonResponse({"error": "Invalid signature"}, status=401)

    # Now safe to process
    data = json.loads(request.body)
    process_payment_webhook(data)
```

**Test Case**:
```python
def test_webhook_without_signature_rejected(self):
    """Webhook without valid signature is rejected"""
    webhook_data = {
        "action": "payment.created",
        "data": {"id": "123456"}
    }

    response = self.client.post(
        '/subscriptions/webhook/',
        data=json.dumps(webhook_data),
        content_type='application/json'
        # No X-Signature header
    )

    self.assertEqual(response.status_code, 401)
    self.assertEqual(PaymentLog.objects.count(), 0)

def test_webhook_with_invalid_signature_rejected(self):
    """Webhook with invalid signature is rejected"""
    webhook_data = {
        "action": "payment.created",
        "data": {"id": "123456"}
    }

    response = self.client.post(
        '/subscriptions/webhook/',
        data=json.dumps(webhook_data),
        content_type='application/json',
        HTTP_X_SIGNATURE="invalid_signature_12345"
    )

    self.assertEqual(response.status_code, 401)
```

---

**Threat 5.2: MercadoPago API URL Validation (SSRF)**
```python
# ❌ VULNERABLE - No URL validation
def get_payment_info(payment_id):
    api_url = f"https://{settings.MERCADOPAGO_API_DOMAIN}/v1/payments/{payment_id}"
    response = requests.get(api_url)
    # If attacker controls API_DOMAIN, can trigger SSRF

# ✅ SECURE - Whitelist allowed domains
ALLOWED_MERCADOPAGO_DOMAINS = [
    'api.mercadopago.com',
    'sandbox.mercadopago.com',
]

def get_payment_info(payment_id):
    # Validate domain
    if settings.MERCADOPAGO_API_DOMAIN not in ALLOWED_MERCADOPAGO_DOMAINS:
        raise ValueError(f"Invalid MercadoPago API domain: {settings.MERCADOPAGO_API_DOMAIN}")

    # Only allow HTTPS
    api_url = f"https://{settings.MERCADOPAGO_API_DOMAIN}/v1/payments/{payment_id}"

    # Validate URL doesn't contain IP addresses (prevent bypass)
    parsed_url = urlparse(api_url)
    if parsed_url.hostname and parsed_url.hostname[0].isdigit():
        raise ValueError("IP addresses not allowed in API URL")

    response = requests.get(api_url, timeout=10)
    return response.json()
```

---

### A06: Vulnerable and Outdated Components

**Threat**: Outdated MercadoPago SDK with known vulnerabilities

#### Dependency Management

**Requirements**:
```python
# requirements.txt
mercadopago==2.2.0  # Pin to specific version
celery==5.3.4
redis==5.0.1
bleach==6.1.0

# Audit dependencies monthly
# pip-audit (recommended tool)
```

**Security Audit Schedule**:
```bash
# Monthly dependency audit
pip-audit --desc

# Update dependencies quarterly
pip install --upgrade mercadopago
```

---

### A07: Identification and Authentication Failures

**Threat**: Authentication bypass in payment operations

#### Session Security

**Threat 7.1: Session Hijacking**
```python
# settings.py

# ❌ VULNERABLE - Insecure session settings
SESSION_COOKIE_HTTPONLY = False  # JavaScript can access
SESSION_COOKIE_SECURE = False  # Sent over HTTP
SESSION_COOKIE_SAMESITE = None  # CSRF vulnerable

# ✅ SECURE - Secure session settings
SESSION_COOKIE_HTTPONLY = True  # Prevent XSS access
SESSION_COOKIE_SECURE = True if not DEBUG else False  # HTTPS only
SESSION_COOKIE_SAMESITE = 'Lax'  # CSRF protection
SESSION_COOKIE_AGE = 3600  # 1 hour timeout
```

---

**Threat 7.2: Unauthenticated Checkout**
```python
# ❌ VULNERABLE - Guest checkout without validation
def reservation_checkout(request, accommodation_id):
    # Anyone can create reservation without authentication
    reservation = Reservation.objects.create(...)

# ✅ SECURE - Require authentication
class ReservationCheckoutView(LoginRequiredMixin, FormView):
    """Checkout requires authentication."""

    def form_valid(self, form):
        # User authenticated, safe to create reservation
        reservation = Reservation.objects.create(
            user=self.request.user,
            ...
        )
```

---

### A08: Software and Data Integrity Failures

**Threat**: Idempotency failures causing duplicate payments

#### Idempotency Patterns

**Threat 8.1: Duplicate Webhook Processing**
```python
# ❌ VULNERABLE - No idempotency check
def subscription_webhook(request):
    payment_id = request.data['data']['id']

    # Process payment (no check for duplicate)
    subscription.status = "active"
    subscription.save()

    # Email sent twice if webhook retried!
    send_payment_success_email(subscription.id)

# ✅ SECURE - Idempotency via database constraint
def subscription_webhook(request):
    payment_id = request.data['data']['id']

    # Check if already processed
    if PaymentLog.objects.filter(mercadopago_payment_id=payment_id).exists():
        logger.info(f"Payment {payment_id} already processed (idempotent)")
        return JsonResponse({"status": "already_processed"}, status=200)

    # Create payment log (unique constraint prevents duplicates)
    PaymentLog.objects.create(
        mercadopago_payment_id=payment_id,  # Unique constraint
        subscription=subscription,
        status="approved",
        ...
    )

    # Safe to send email (only sent once)
    send_payment_success_email(subscription.id)
```

**Migration**:
```python
# apps/subscriptions/migrations/XXXX_add_payment_id_unique_constraint.py

operations = [
    migrations.AlterField(
        model_name='paymentlog',
        name='mercadopago_payment_id',
        field=models.CharField(max_length=100, unique=True, db_index=True),
    ),
]
```

---

**Threat 8.2: Duplicate Reservations (Idempotency Key)**
```python
# ❌ VULNERABLE - Same user can create duplicate reservations
def create_reservation(accommodation, check_in, check_out, guest_email):
    # No check for existing reservation
    reservation = Reservation.objects.create(...)

# ✅ SECURE - SHA256 idempotency key
import hashlib

def generate_reservation_idempotency_key(
    accommodation_id: int,
    check_in: date,
    check_out: date,
    guest_email: str
) -> str:
    """
    Generate SHA256 idempotency key for reservation.

    Prevents duplicate reservations for same accommodation, dates, guest.
    """
    data = f"{accommodation_id}:{check_in}:{check_out}:{guest_email}"
    return hashlib.sha256(data.encode()).hexdigest()

def create_reservation_idempotent(accommodation, check_in, check_out, guest_email):
    # Generate idempotency key
    idempotency_key = generate_reservation_idempotency_key(
        accommodation.id, check_in, check_out, guest_email
    )

    # Check if already exists
    existing = Reservation.objects.filter(
        idempotency_key=idempotency_key,
        status__in=["pending", "confirmed"]
    ).first()

    if existing:
        logger.info(f"Reservation {existing.id} already exists (idempotent)")
        return existing

    # Create new reservation
    reservation = Reservation.objects.create(
        accommodation_type=accommodation,
        check_in=check_in,
        check_out=check_out,
        guest_email=guest_email,
        idempotency_key=idempotency_key,
        ...
    )

    return reservation
```

**Model Update**:
```python
# apps/reservations/models.py

class Reservation(models.Model):
    # ... existing fields ...

    idempotency_key = models.CharField(
        max_length=64,
        unique=True,
        db_index=True,
        help_text="SHA256 idempotency key to prevent duplicates"
    )
```

---

### A09: Security Logging and Monitoring Failures

**Threat**: Lack of audit trail for payment operations

#### Logging Requirements

**What to Log**:
```python
# apps/subscriptions/services.py

import logging

logger = logging.getLogger(__name__)

class SubscriptionService:
    @staticmethod
    def create_customer(email: str, payment_token: str) -> str:
        """Create MercadoPago customer with logging."""
        logger.info(f"Creating MercadoPago customer for {email}")

        sdk = mercadopago.SDK(settings.MERCADOPAGO_ACCESS_TOKEN)

        try:
            response = sdk.customer().create({
                "email": email,
                "card_token": payment_token,
            })

            if response["status"] != 201:
                logger.error(f"Failed to create customer for {email}: {response}")
                raise ValueError(f"Failed to create customer: {response}")

            payer_id = response["response"]["id"]
            logger.info(f"Created MercadoPago customer {payer_id} for {email}")
            return payer_id

        except Exception as e:
            logger.exception(f"Exception creating customer for {email}: {e}")
            raise
```

**Log Events to Track**:
1. **Subscription events**:
   - Trial start/end
   - Payment success/failure
   - Retry attempts
   - Suspension/cancellation

2. **Reservation events**:
   - Reservation created
   - Payment preference created
   - Payment approved/rejected
   - Timeout/cancellation

3. **Webhook events**:
   - Signature verification (success/failure)
   - Idempotency checks
   - Processing errors

4. **Security events**:
   - Invalid webhook signatures
   - Unauthorized access attempts
   - Multi-tenant isolation violations

**Log Format**:
```python
# settings.py

LOGGING = {
    'version': 1,
    'handlers': {
        'payment_file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/calma/payment.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'apps.subscriptions': {
            'handlers': ['payment_file'],
            'level': 'INFO',
            'propagate': False,
        },
        'apps.reservations': {
            'handlers': ['payment_file'],
            'level': 'INFO',
            'propagate': False,
        },
    },
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
}
```

---

### A10: Server-Side Request Forgery (SSRF)

**Threat**: SSRF via MercadoPago API URL manipulation

**Covered in A05: Security Misconfiguration (Threat 5.2)**

---

## Payment-Specific Threats

### Threat: Price Manipulation

**Attack Vector**:
- User modifies price in browser DevTools
- User submits modified form with lower price
- Reservation created with manipulated price

**Vulnerable Code**:
```python
# ❌ VULNERABLE - Price from user input
def create_reservation(request):
    total_price = request.POST['total_price']  # User-controlled!
    reservation = Reservation.objects.create(total_price=total_price, ...)
```

**Secure Code**:
```python
# ✅ SECURE - Server-side price calculation
def create_reservation(request, accommodation_id):
    accommodation = get_object_or_404(AccommodationType, id=accommodation_id)

    check_in = form.cleaned_data['check_in']
    check_out = form.cleaned_data['check_out']

    # Calculate price on server (NEVER trust client)
    num_nights = (check_out - check_in).days
    total_price = accommodation.price_per_night * num_nights

    reservation = Reservation.objects.create(
        total_price=total_price,  # Server-calculated
        ...
    )
```

**Test Case**:
```python
def test_price_manipulation_blocked(self):
    """User cannot manipulate price via POST data"""
    accommodation = AccommodationType.objects.create(
        site=site,
        price_per_night=Decimal("100.00")
    )

    self.client.login(username='user', password='pass')

    # Attempt to submit lower price
    response = self.client.post('/reservas/checkout/1/', {
        'check_in': '2025-12-01',
        'check_out': '2025-12-05',
        'total_price': '50.00',  # Manipulated (should be 400.00)
        ...
    })

    # Verify server-calculated price used
    reservation = Reservation.objects.last()
    self.assertEqual(reservation.total_price, Decimal("400.00"))  # 4 nights × 100
    self.assertNotEqual(reservation.total_price, Decimal("50.00"))
```

---

### Threat: Replay Attacks (Webhook)

**Attack Vector**:
- Attacker captures valid webhook request
- Attacker replays webhook multiple times
- System processes same payment multiple times (grants access without new payment)

**Mitigation**:
```python
# Redis-based idempotency cache (24h TTL)
from django.core.cache import cache

def process_webhook_idempotent(webhook_data):
    event_id = webhook_data.get('id')  # MercadoPago webhook event ID

    cache_key = f"webhook_processed:{event_id}"

    # Check if already processed (Redis cache)
    if cache.get(cache_key):
        logger.info(f"Webhook {event_id} already processed (idempotent)")
        return {"status": "already_processed"}

    # Process webhook
    process_payment_webhook(webhook_data)

    # Mark as processed (24h TTL)
    cache.set(cache_key, True, timeout=86400)

    return {"status": "ok"}
```

**Test Case**:
```python
def test_webhook_replay_prevented(self):
    """Duplicate webhook requests are ignored"""
    webhook_data = {
        "id": "webhook_12345",
        "action": "payment.created",
        "data": {"id": "payment_123"}
    }

    # First request (processes normally)
    response1 = self.client.post('/subscriptions/webhook/', webhook_data)
    self.assertEqual(response1.status_code, 200)
    self.assertEqual(PaymentLog.objects.count(), 1)

    # Second request (idempotent, ignored)
    response2 = self.client.post('/subscriptions/webhook/', webhook_data)
    self.assertEqual(response2.status_code, 200)
    self.assertEqual(PaymentLog.objects.count(), 1)  # No duplicate
```

---

### Threat: Man-in-the-Middle (MITM)

**Attack Vector**:
- Attacker intercepts payment request/response
- Attacker modifies payment details
- Guest pays for wrong accommodation or modified price

**Mitigation**:
```python
# settings.py (production)

# Enforce HTTPS for all requests
SECURE_SSL_REDIRECT = True

# HSTS (HTTP Strict Transport Security)
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Secure cookies
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# Certificate pinning (optional, for MercadoPago API)
MERCADOPAGO_CERT_BUNDLE = '/path/to/mercadopago-certs.pem'
```

**Nginx Configuration**:
```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    # SSL certificates
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Strong SSL protocols
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # HSTS header
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

### Threat: Subscription Hijacking

**Attack Vector**:
- User A creates subscription
- User B modifies subscription_id in URL
- User B cancels or modifies User A's subscription

**Mitigation**:
```python
# apps/subscriptions/views.py

class SubscriptionCancelView(LoginRequiredMixin, FormView):
    """Cancel subscription with ownership check."""

    def dispatch(self, request, *args, **kwargs):
        # Get subscription from user (not from URL parameter)
        try:
            subscription = request.user.subscription
        except Subscription.DoesNotExist:
            raise Http404("No subscription found")

        # Store for use in form_valid
        self.subscription = subscription

        return super().dispatch(request, *args, **kwargs)

    def form_valid(self, form):
        # Safe to cancel (we validated ownership in dispatch)
        SubscriptionService.cancel_subscription(self.subscription)
        self.subscription.cancelled_at = timezone.now()
        self.subscription.save()

        return redirect("subscriptions:cancelled")
```

**URL Pattern**:
```python
# ❌ VULNERABLE - Subscription ID in URL
path('subscriptions/<int:subscription_id>/cancel/', SubscriptionCancelView.as_view())

# ✅ SECURE - No ID in URL, use request.user
path('subscriptions/cancel/', SubscriptionCancelView.as_view(), name='cancel')
```

---

### Threat: Payment Spoofing

**Attack Vector**:
- Attacker sends fake webhook claiming payment approved
- System grants subscription access without payment

**Mitigation**:
- **CRITICAL**: HMAC signature verification (covered in A05: Threat 5.1)
- **Backup**: Poll MercadoPago API to verify payment status

```python
# Webhook processing with verification
def subscription_webhook(request):
    # 1. Verify HMAC signature
    if not verify_mercadopago_webhook(request):
        logger.warning("Invalid webhook signature")
        return JsonResponse({"error": "Invalid signature"}, status=401)

    # 2. Extract payment ID
    payment_id = request.data['data']['id']

    # 3. Fetch payment details from MercadoPago API (backup verification)
    payment_info = SubscriptionService.get_payment_info(payment_id)

    # 4. Verify payment actually approved on MercadoPago side
    if payment_info['status'] != 'approved':
        logger.warning(f"Payment {payment_id} not approved on MercadoPago")
        return JsonResponse({"error": "Payment not approved"}, status=400)

    # 5. Safe to process
    process_subscription_payment(payment_info)
```

---

## Testing Strategy

### Security Test Coverage

**Target**: 30+ security-focused tests

**Categories**:

1. **Multi-tenant Isolation** (10 tests):
   - User cannot access other user's subscription
   - Admin cannot access other site's reservations
   - Webhook cannot update wrong tenant's data
   - Plan limits enforced per user
   - Payment logs isolated by subscription/reservation

2. **Input Validation** (5 tests):
   - Webhook data validated with Pydantic
   - Price manipulation blocked
   - Date validation (check-in < check-out, future only)
   - Honeypot detection
   - SQL injection attempts blocked

3. **Webhook Security** (5 tests):
   - Invalid HMAC signature rejected
   - Missing signature rejected
   - Idempotency check works
   - Replay attacks prevented
   - Malformed JSON rejected

4. **Race Conditions** (3 tests):
   - Concurrent reservation attempts (select_for_update)
   - Concurrent subscription activations
   - Concurrent availability checks

5. **Authentication** (3 tests):
   - Unauthenticated user cannot checkout
   - Session hijacking prevented
   - Subscription ownership validated

6. **Idempotency** (4 tests):
   - Duplicate webhook processing
   - Duplicate reservation creation
   - Payment log unique constraint
   - Retry logic max attempts

**Example Test Suite**:
```python
# tests/security/test_payment_security.py

class PaymentSecurityTestCase(TestCase):
    """Security tests for payment integration."""

    def test_price_manipulation_blocked(self):
        """User cannot manipulate price via POST data"""
        # ... (covered above)

    def test_webhook_signature_required(self):
        """Webhook without valid HMAC signature is rejected"""
        # ... (covered above)

    def test_multi_tenant_isolation_subscription(self):
        """User A cannot access User B's subscription"""
        # ... (covered above)

    def test_concurrent_reservations_prevented(self):
        """Race condition prevention with select_for_update"""
        # ... (covered above)

    def test_sql_injection_blocked(self):
        """SQL injection in payment_id parameter blocked"""
        malicious_payment_id = "123' OR '1'='1"

        # Attempt injection via webhook
        webhook_data = {
            "action": "payment.created",
            "data": {"id": malicious_payment_id}
        }

        response = self.client.post('/subscriptions/webhook/', webhook_data)

        # Should fail safely (no data leaked)
        self.assertIn(response.status_code, [400, 404])
        self.assertEqual(PaymentLog.objects.count(), 0)

    def test_replay_attack_prevented(self):
        """Duplicate webhook requests ignored"""
        # ... (covered above)

    def test_subscription_hijacking_blocked(self):
        """User B cannot cancel User A's subscription"""
        user_a = User.objects.create_user('user_a', 'a@test.com', 'pass')
        user_b = User.objects.create_user('user_b', 'b@test.com', 'pass')

        subscription_a = Subscription.objects.create(user=user_a, plan=plan, status="active")

        # User B tries to cancel User A's subscription
        self.client.login(username='user_b', password='pass')
        response = self.client.post('/subscriptions/cancel/')

        # User A's subscription unaffected
        subscription_a.refresh_from_db()
        self.assertEqual(subscription_a.status, "active")
        self.assertIsNone(subscription_a.cancelled_at)
```

---

### Penetration Testing Checklist

**Manual Tests**:

- [ ] Attempt to access other user's subscription via URL manipulation
- [ ] Attempt to modify payment amount in browser DevTools
- [ ] Send webhook with invalid/missing HMAC signature
- [ ] Attempt concurrent reservations for same unit/date
- [ ] Attempt to create reservation without authentication
- [ ] Replay captured webhook request
- [ ] Inject SQL in payment_id parameter
- [ ] Access reservation from other site via URL manipulation
- [ ] Exceed plan limits (create more sites than allowed)
- [ ] Cancel other user's subscription via URL manipulation

**Automated Tools**:
- OWASP ZAP scan on payment endpoints
- SQLMap scan on payment-related parameters
- Burp Suite session manipulation tests
- Race condition tests with Apache Bench

---

## Security Audit Summary

### Critical Security Controls

**MUST HAVE (Before Production)**:

1. **PCI DSS Compliance**:
   - [x] NO card data stored locally
   - [x] MercadoPago tokenization
   - [x] HTTPS enforced
   - [ ] Webhook secrets encrypted (Fernet)
   - [ ] Payment tokens encrypted at rest

2. **Multi-tenant Isolation**:
   - [x] Subscription queries filter by user
   - [x] Reservation queries filter by site
   - [x] SubscriptionRequiredMixin enforced
   - [x] Plan limits checked before operations

3. **Webhook Security**:
   - [ ] HMAC SHA256 signature verification
   - [ ] Idempotency checks (database + Redis)
   - [ ] Rate limiting (100 webhooks/min per IP)
   - [x] CSRF exempt only on webhooks

4. **Race Condition Prevention**:
   - [ ] select_for_update() for availability checks
   - [ ] Atomic transactions for reservations
   - [ ] Idempotency keys for duplicate prevention

5. **Input Validation**:
   - [ ] Pydantic schemas for webhook data
   - [ ] Bleach sanitization on text fields
   - [x] Server-side price calculation
   - [x] Date validation
   - [x] Honeypot fields

6. **Logging & Monitoring**:
   - [ ] All payment operations logged
   - [ ] Webhook events logged with signature status
   - [ ] Security events logged (unauthorized access)
   - [ ] Rotating log files (10MB max)

---

### Risk Matrix

| Threat | Severity | Likelihood | Impact | Mitigation Status |
|--------|----------|------------|--------|-------------------|
| Multi-tenant data leakage | CRITICAL | MEDIUM | HIGH | ✅ Implemented |
| Price manipulation | HIGH | HIGH | MEDIUM | ✅ Implemented |
| Webhook spoofing | CRITICAL | LOW | HIGH | ⚠️ To implement |
| Race condition (overbooking) | HIGH | MEDIUM | HIGH | ⚠️ To implement |
| SQL injection | CRITICAL | LOW | HIGH | ✅ ORM prevents |
| Session hijacking | HIGH | LOW | MEDIUM | ✅ Secure cookies |
| Replay attacks | MEDIUM | MEDIUM | MEDIUM | ⚠️ To implement |
| SSRF | MEDIUM | LOW | MEDIUM | ⚠️ To implement |
| Subscription hijacking | HIGH | MEDIUM | HIGH | ✅ Implemented |
| Payment spoofing | CRITICAL | LOW | HIGH | ⚠️ To implement |

**Legend**:
- ✅ Implemented - Security control in place
- ⚠️ To implement - Requires implementation before production
- ❌ Not implemented - Security gap identified

---

## Recommendations

1. **PRIORITY 1 (Before Production)**:
   - Implement HMAC webhook signature verification
   - Add select_for_update() for availability checks
   - Encrypt payment tokens with Fernet
   - Implement Redis-based idempotency for webhooks
   - Add rate limiting on webhook endpoints

2. **PRIORITY 2 (Before Production)**:
   - SSRF protection (domain whitelist for MercadoPago API)
   - Comprehensive security logging
   - Automated security tests (30+ tests)
   - Penetration testing with OWASP ZAP

3. **PRIORITY 3 (Post-Launch)**:
   - Security monitoring dashboard
   - Quarterly dependency audits
   - Webhook secret rotation (every 90 days)
   - Bug bounty program

---

## Compliance Requirements

**PCI DSS SAQ A (Self-Assessment Questionnaire A)**:

- [x] Question 1: Do you store cardholder data? **NO** (MercadoPago tokenization)
- [x] Question 2: Do you store sensitive authentication data after authorization? **NO**
- [x] Question 3: Is cardholder data transmitted over public networks? **YES** (encrypted via HTTPS)
- [x] Question 4: Is your website protected with firewall? **YES** (AWS WAF/similar)
- [x] Question 5: Do you maintain security patches? **YES** (monthly audits)

**Additional Compliance**:
- Argentina Data Protection Law (DPA): Personal data handling
- Consumer Protection Law: Clear refund policies
- BCRA Regulations: NOT payment processor (split payment avoids licensing)

---

## Document Version

**Version**: 1.0
**Created**: 2025-11-18
**Last Updated**: 2025-11-18
**Status**: Draft - Requires implementation of Priority 1 items

---

## Next Steps

1. Implement Priority 1 security controls (webhook HMAC, select_for_update, encryption)
2. Create security test suite (30+ tests)
3. Run security-auditor after implementation
4. Penetration testing with OWASP ZAP
5. Final security review before production deployment
