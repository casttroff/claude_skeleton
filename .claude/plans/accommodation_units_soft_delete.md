# Plan: AccommodationUnity Auto-Creation & Soft Delete

**Feature:** Gestion de AccommodationUnity con auto-creacion y soft delete
**Fecha:** 2025-12-01
**Estado:** Pendiente Aprobacion

---

## Resumen Ejecutivo

Implementar gestion completa de AccommodationUnity con:
1. Auto-creacion de primera unidad via Signal
2. UI de formset para agregar/editar unidades
3. Soft delete con validacion de reservaciones

---

## Decisiones de Arquitectura (Confirmadas por Usuario)

| Decision | Opcion Elegida |
|----------|----------------|
| Soft Delete Pattern | `deleted_at` timestamp (DDD riguroso) |
| Bloqueo de Eliminacion | Modal con mensaje de error |
| Formset UI | Boton + inline (patron Django estandar) |
| Nombre Predeterminado | "Unidad 1" (simple y claro) |

---

## Componentes a Implementar

### 1. SoftDeleteMixin (Framework Layer)

**Archivo:** `framework/models.py`

Agregar mixin reutilizable con:
- Campo `deleted_at` (DateTimeField, null=True, db_index=True)
- Metodo `soft_delete()` - marca como eliminado
- Metodo `restore()` - restaura registro
- Property `is_deleted` - verifica estado

### 2. SoftDeleteQuerySet (Framework Layer)

**Archivo:** `framework/managers.py`

Agregar queryset base con:
- `.active()` - filtra deleted_at=NULL
- `.deleted()` - solo eliminados
- `.with_deleted()` - todos

### 3. Specification Pattern (Domain Layer)

**Archivo:** `apps/accommodations/specifications.py` (NUEVO)

Implementar:
- `Specification` base class (ABC)
- `AndSpecification`, `OrSpecification` para composicion
- `AccommodationUnityDeletableSpec` - valida que no tenga reservas activas
- `AccommodationTypeDeletableSpec` - valida que todas sus unidades sean eliminables

**Regla de Negocio:**
- Se puede eliminar si NO tiene reservaciones con status: `initialized`, `pending`, `confirmed`
- Reservaciones `cancelled` NO bloquean eliminacion

### 4. Cambios en Modelos

**Archivo:** `apps/accommodations/models.py`

#### AccommodationType:
- Heredar de `SoftDeleteMixin`
- Agregar metodo `can_soft_delete()` - usa AccommodationTypeDeletableSpec
- Override `soft_delete()` - elimina todas las unidades y luego el tipo

#### AccommodationUnity:
- Heredar de `SoftDeleteMixin`
- Agregar metodo `can_soft_delete()` - usa AccommodationUnityDeletableSpec
- Agregar metodo `get_blocking_reservations()` - para mostrar en modal de error

### 5. Cambios en Managers

**Archivo:** `apps/accommodations/managers.py`

#### AccommodationUnityQuerySet:
- Heredar de `SoftDeleteQuerySet`
- Actualizar `get_available_units()` para excluir soft-deleted

#### AccommodationUnityManager:
- Agregar metodos Repository-style: `find_by_id()`, `find_for_accommodation_type()`, `count_active_units()`

### 6. Signal para Auto-Creacion

**Archivo:** `apps/accommodations/signals.py`

Agregar signal `auto_create_first_accommodation_unit`:
- Trigger: `post_save` de AccommodationType
- Condicion: Solo si `created=True`
- Accion: Crear AccommodationUnity con name="Unidad 1", order=1

### 7. Forms y FormSet

**Archivo:** `apps/accommodations/forms.py`

Agregar:
- `AccommodationUnityForm` - ModelForm para AccommodationUnity
- `AccommodationUnityFormSet` - inlineformset_factory con extra=1, can_delete=True

### 8. Vista de Gestion de Unidades

**Archivo:** `apps/accommodations/views.py`

Agregar `AccommodationUnityManageView`:
- Hereda de `SiteAdminRequiredMixin`, `UpdateView`
- Template: `accommodations/admin/manage_units.html`
- Maneja formset con transaccion atomica
- Valida soft-delete antes de eliminar
- Muestra error modal si hay reservas bloqueantes

### 9. URL Pattern

**Archivo:** `apps/accommodations/urls.py`

Agregar:
```python
path('admin/<int:pk>/units/', views.AccommodationUnityManageView.as_view(), name='manage_units')
```

### 10. Templates

#### manage_units.html (NUEVO)
- Formset con cards para cada unidad
- Boton "+ Agregar Unidad" con JS para agregar forms dinamicamente
- Checkbox de eliminar con validacion visual (opacity)
- Nombre auto-generado: "Unidad {count+1}"

#### Actualizar list.html
- Agregar opcion "Agregar Unidades" en dropdown de cada accommodation

---

## Migraciones de Base de Datos

### Migracion 1: SoftDeleteMixin fields
```python
# AccommodationType
migrations.AddField(
    model_name='accommodationtype',
    name='deleted_at',
    field=models.DateTimeField(null=True, blank=True, db_index=True),
)

# AccommodationUnity
migrations.AddField(
    model_name='accommodationunity',
    name='deleted_at',
    field=models.DateTimeField(null=True, blank=True, db_index=True),
)
```

---

## Orden de Implementacion (TDD)

### Fase 1: Infrastructure (Framework Layer)
1. Crear `SoftDeleteMixin` en `framework/models.py`
2. Crear `SoftDeleteQuerySet` en `framework/managers.py`
3. Tests unitarios para mixin y queryset

### Fase 2: Specifications (Domain Layer)
4. Crear `apps/accommodations/specifications.py`
5. Implementar `AccommodationUnityDeletableSpec`
6. Implementar `AccommodationTypeDeletableSpec`
7. Tests unitarios para specifications

### Fase 3: Model Changes
8. Agregar `SoftDeleteMixin` a modelos
9. Agregar metodos `can_soft_delete()`, `get_blocking_reservations()`
10. Actualizar managers con queryset base
11. Crear migraciones
12. Tests de integracion para soft delete

### Fase 4: Signal
13. Agregar signal `auto_create_first_accommodation_unit`
14. Tests para signal

### Fase 5: Forms & Views
15. Crear `AccommodationUnityForm` y formset
16. Crear `AccommodationUnityManageView`
17. Tests para vista

### Fase 6: Templates & UI
18. Crear template `manage_units.html`
19. Actualizar template `list.html` con opcion en dropdown
20. JavaScript para formset dinamico

---

## Criterios de Aceptacion

### 1. Auto-Creacion de Primera Unidad
- [ ] Al crear AccommodationType, se crea automaticamente "Unidad 1"
- [ ] Signal no se dispara en updates, solo en creates
- [ ] Si ya existen unidades, signal no crea duplicados

### 2. UI de Gestion de Unidades
- [ ] Boton "Agregar Unidades" visible en dropdown de list
- [ ] Vista muestra formset con unidades existentes
- [ ] Boton "+" agrega form inline con nombre "Unidad {count+1}"
- [ ] Cambios se guardan con transaccion atomica

### 3. Soft Delete de AccommodationUnity
- [ ] Campo `deleted_at` se setea en lugar de eliminar registro
- [ ] No se puede eliminar si tiene reservas `initialized`, `pending`, `confirmed`
- [ ] Se puede eliminar si reservas son `cancelled`
- [ ] Modal de error muestra mensaje claro cuando no se puede eliminar
- [ ] Unidades eliminadas no aparecen en listados

### 4. Soft Delete de AccommodationType
- [ ] Solo se puede eliminar si TODAS sus unidades son eliminables
- [ ] Al eliminar, primero soft-delete de todas las unidades, luego del tipo
- [ ] Validacion recursiva respeta regla de reservas

### 5. Queries Actualizadas
- [ ] `get_available_units()` excluye unidades soft-deleted
- [ ] `get_first_available_unit()` excluye unidades soft-deleted
- [ ] Manager filtra por `deleted_at__isnull=True` por defecto

---

## Estimacion de Tiempo

| Fase | Tiempo Estimado |
|------|-----------------|
| Framework Layer | 1 hora |
| Specifications | 1 hora |
| Model Changes + Migrations | 1.5 horas |
| Signal | 30 min |
| Forms & Views | 1.5 horas |
| Templates & JS | 1 hora |
| Tests | 1.5 horas |
| **Total** | **8 horas** |

---

## Riesgos y Mitigaciones

| Riesgo | Mitigacion |
|--------|------------|
| Race condition en eliminacion | Transaccion atomica con `select_for_update()` |
| Unidades huerfanas si falla eliminacion de tipo | Rollback automatico via `@transaction.atomic` |
| Performance en queries con many units | Index en `deleted_at` |
| Formset JS compatibility | Usar patron Django estandar probado |

---

## Notas Adicionales

- Esta implementacion sigue los patrones de MENTOR.md (SOLID, DDD)
- El Specification Pattern permite agregar nuevas reglas de eliminacion sin modificar codigo existente (OCP)
- El SoftDeleteMixin es reutilizable para otros modelos que necesiten soft delete
- Compatible con el sistema de reservaciones existente (ReservationBuilder, etc.)
