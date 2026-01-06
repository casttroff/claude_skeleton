# OAuth Signal Debugging Guide

## 🔍 Problema: `social_account_added` no se dispara

### ✅ Estado Actual: Signal Correctamente Configurado

Basado en la investigación con context7 y revisión del código:

- ✅ `apps.py` importa signals en `ready()` method
- ✅ Decorador `@receiver(social_account_added)` es correcto
- ✅ Firma de función coincide con documentación django-allauth
- ✅ Signal está registrado en Django

**El signal NO tiene problemas de configuración.**

---

## 📌 Por Qué No Se Dispara

### Comportamiento del Signal `social_account_added`

Este signal **solo se dispara cuando**:

1. **Primera autenticación con Google** (usuario nuevo)
2. **Reconexión después de eliminar SocialAccount** del admin
3. **Account linking** (usuario Django existente conecta Google)

**NO se dispara cuando**:
- ❌ Login subsecuente con cuenta Google ya conectada
- ❌ Refresh de token
- ❌ Re-autenticación en la misma sesión

### Signal Alternativo: `pre_social_login`

Si necesitas ejecutar lógica en **cada login** (no solo primera vez):

```python
@receiver(pre_social_login)
def update_profile_on_every_login(sender, request, sociallogin, **kwargs):
    """Se dispara en CADA login, no solo primera vez"""

    # Solo ejecutar si es login exitoso (no signup)
    if sociallogin.is_existing:
        user = sociallogin.user
        google_data = sociallogin.account.extra_data

        # Actualizar profile en cada login
        profile = user.profile
        if "picture" in google_data:
            profile.picture = google_data["picture"]
        profile.save()
```

---

## 🧪 Cómo Verificar Que Signals Funcionan

### Paso 1: Revisar Console Output

Con los debug prints agregados, deberías ver:

```
================================================================================
PRE_SOCIAL_LOGIN SIGNAL FIRED
  Provider: google
  Is Existing: True
  Email: tu@email.com
  User Authenticated: False
================================================================================
```

**Si ves "Is Existing: True"** → Cuenta ya existe, `social_account_added` NO se disparará.

### Paso 2: Probar Con Nueva Cuenta

Para que `social_account_added` se dispare, debes:

**Opción A: Eliminar Social Account (Recomendado para testing)**

1. Ve a Django admin: `http://localhost:8011/admin/`
2. Busca "Social accounts" (django-allauth)
3. Elimina tu SocialAccount de Google
4. Vuelve a autenticarte con Google
5. Ahora verás:

```
================================================================================
SOCIAL_ACCOUNT_ADDED SIGNAL FIRED ✓
================================================================================
Provider: google
UID: 1234567890
User Email: tu@email.com
User ID: 5
User Exists in DB: True

Extra Data (all fields):
  email: tu@email.com
  name: Tu Nombre
  given_name: Tu
  family_name: Nombre
  picture: https://lh3.googleusercontent.com/...
  locale: es
================================================================================

✓ Profile created for tu@email.com
```

**Opción B: Usar Otra Cuenta Google**

1. Usa otra cuenta Google que nunca hayas usado en la app
2. Autentica por primera vez
3. `social_account_added` se disparará

**Opción C: Django Flush (Borra TODA la DB)**

⚠️ **PELIGRO: Borra todos los datos**

```bash
make shell
python manage.py flush --no-input
```

---

## 🔧 Testing Checklist

### Test 1: Nuevo Usuario (social_account_added SE DISPARA)

1. Elimina SocialAccount del admin
2. Logout de Google en el browser
3. Ve a `/accounts/google/login/`
4. Autentica con Google
5. Verifica console output:
   - ✓ `PRE_SOCIAL_LOGIN SIGNAL FIRED` con `Is Existing: False`
   - ✓ `SOCIAL_ACCOUNT_ADDED SIGNAL FIRED ✓`
6. Verifica en DB:
   - Profile tiene `picture` de Google
   - User tiene `first_name` y `last_name`
   - User tiene `phone` si Google lo proporcionó

### Test 2: Usuario Existente (social_account_added NO SE DISPARA)

1. Login con la misma cuenta Google
2. Verifica console output:
   - ✓ `PRE_SOCIAL_LOGIN SIGNAL FIRED` con `Is Existing: True`
   - ❌ `SOCIAL_ACCOUNT_ADDED SIGNAL FIRED` **NO aparece** (esperado)
3. Usuario es autenticado, pero signal no se ejecuta

### Test 3: Account Linking (social_account_added SE DISPARA)

1. Crea usuario Django manual: `user@example.com`
2. No conectes Google todavía
3. Logout
4. Ve a `/accounts/google/login/`
5. Autentica con `user@example.com` en Google
6. Signal `pre_social_login` redirige a login Django
7. Login con password Django
8. Account linking se ejecuta
9. `social_account_added` se dispara

---

## 🐛 Troubleshooting

### Signal No Se Dispara Nunca

**Check 1: Apps.py Ready Method**

Verifica que `apps/users/apps.py` tenga:

```python
def ready(self):
    """Import signals when Django starts"""
    import apps.users.signals
```

**Check 2: INSTALLED_APPS Order**

Verifica que en `settings.py`:

```python
LOCAL_APPS = [
    "apps.core",
    "apps.web",
    "apps.users",  # ✓ Debe estar aquí
    # ...
]
```

**Check 3: Signal Import**

En `apps/users/signals.py`:

```python
from allauth.socialaccount.signals import social_account_added

@receiver(social_account_added)  # ✓ Correcto
def populate_profile_from_google(sender, request, sociallogin, **kwargs):
    # ...
```

**Check 4: Restart Docker**

```bash
make down
make up
```

---

## 📋 Estrategias de Sincronización

### Estrategia 1: Solo Primera Vez (Actual)

```python
@receiver(social_account_added)
def populate_profile_from_google(sender, request, sociallogin, **kwargs):
    """Se ejecuta solo cuando se crea nueva SocialAccount"""
    # Profile se crea/actualiza una vez
```

**Pros:**
- No sobrescribe cambios manuales del usuario
- Performance (no ejecuta en cada login)

**Contras:**
- Si usuario cambia foto en Google, no se actualiza en la app

---

### Estrategia 2: Sincronización en Cada Login

Si quieres actualizar profile en **cada login**:

```python
from allauth.account.signals import user_logged_in

@receiver(user_logged_in)
def sync_profile_on_login(sender, request, user, **kwargs):
    """
    Sincroniza profile con Google en cada login.

    IMPORTANTE: Solo ejecutar si usuario tiene SocialAccount de Google.
    """
    from allauth.socialaccount.models import SocialAccount

    try:
        social_account = SocialAccount.objects.get(
            user=user,
            provider='google'
        )

        google_data = social_account.extra_data
        profile = user.profile

        # Actualizar picture en cada login
        if "picture" in google_data:
            profile.picture = google_data["picture"]
            profile.save()
            logger.info(f"Profile picture synced for {user.email}")

    except SocialAccount.DoesNotExist:
        # Usuario no tiene cuenta Google conectada
        pass
```

**Pros:**
- Profile siempre sincronizado con Google
- Si usuario cambia foto en Google, se actualiza

**Contras:**
- Sobrescribe cambios manuales del usuario
- Ejecuta query en cada login

---

### Estrategia 3: Sincronización Manual

Botón en dashboard para "Sync with Google":

```python
# views.py
from allauth.socialaccount.models import SocialAccount

class SyncGoogleProfileView(LoginRequiredMixin, View):
    """Permite al usuario sincronizar su profile manualmente"""

    def post(self, request):
        try:
            social_account = SocialAccount.objects.get(
                user=request.user,
                provider='google'
            )

            google_data = social_account.extra_data
            profile = request.user.profile

            if "picture" in google_data:
                profile.picture = google_data["picture"]

            if "given_name" in google_data:
                request.user.first_name = google_data["given_name"]

            if "family_name" in google_data:
                request.user.last_name = google_data["family_name"]

            profile.save()
            request.user.save()

            messages.success(request, "Profile synced with Google!")

        except SocialAccount.DoesNotExist:
            messages.error(request, "No Google account connected.")

        return redirect('core:dashboard')
```

---

## 🎯 Recomendación

**Para tu caso (Ribera Calma):**

1. **Mantén `social_account_added`** para primera sincronización ✓
2. **Agrega `user_logged_in`** si quieres actualizar picture en cada login
3. **No uses sync manual** (overkill para MVP)

**Código sugerido:**

```python
from allauth.account.signals import user_logged_in

@receiver(user_logged_in)
def sync_google_picture_on_login(sender, request, user, **kwargs):
    """
    Actualiza picture de Google en cada login.

    Solo ejecuta si usuario tiene cuenta Google conectada.
    NO sobrescribe first_name/last_name (usuario puede cambiarlos).
    """
    from allauth.socialaccount.models import SocialAccount

    try:
        social_account = SocialAccount.objects.get(
            user=user,
            provider='google'
        )

        google_data = social_account.extra_data

        # Solo actualizar picture (no nombre)
        if "picture" in google_data:
            profile = user.profile
            profile.picture = google_data["picture"]
            profile.save()
            logger.info(f"Profile picture synced for {user.email}")

    except SocialAccount.DoesNotExist:
        # Usuario Django sin OAuth - skip
        pass
```

---

## 📝 Summary

| Signal | Cuándo Se Dispara | Usar Para |
|--------|-------------------|-----------|
| `social_account_added` | Solo primera conexión | Crear profile inicial |
| `pre_social_login` | Cada login (antes de autenticar) | Validaciones, account linking |
| `user_logged_in` | Cada login (después de autenticar) | Sincronización periódica |

**Tu problema:** Estabas testeando con cuenta existente → Signal correcto, comportamiento esperado.

**Solución:** Elimina SocialAccount del admin y vuelve a autenticar para ver el signal dispararse.
