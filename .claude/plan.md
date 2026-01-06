# Plan: Arquitectura Escalable para Apps de Seguros ("Productos Enlatados")

## Resumen del Analisis

Analisis de las apps `hogar/`, `vehicles/` y `laborales/` identificando patrones comunes y oportunidades de mejora arquitectonica.

---

## 1. Estado Actual: Patrones Identificados

### 1.1 Estructura Comun por App

```
branchs/{categoria}/{producto}/
    models.py
    domain/entities.py
    services.py
    views.py
    forms.py
    managers.py
    api/v1/
```

### 1.2 Modelos Base Compartidos (`commercial_core/models.py`)

| Modelo Base | Proposito |
|-------------|-----------|
| `BaseQuotation` | Cotizacion abstracta (product, person, status) |
| `BaseQuotationCover` | Cobertura con precio (fee, extra_data, company) |
| `BasePreSale` | Emision/venta (policy_holder, policy_number, status) |

### 1.3 DTOs y Entities (`core/domain/entities.py`)

```python
@dataclass
class QuoteBase(DataclassBase):
    pk: int | None = None
    intermediaries: Intermediaries | None = None
    person: Person | None = None
    zip_code: int | None = None
    product: str | None = None
    locality: Locality | None = None
```

### 1.4 Services Pattern Actual

```python
def process_vehicle_coverage(company_slug: str, quote_id: int, cover: Cover, coverage: dict) -> None:
    company = Company.objects.get(slug=company_slug)
    quote = QuotationVehicle.objects.get(pk=quote_id)
    validity_type = Validity.objects.get(slug="mensual")
    QuotationVehicleCover.objects.create(...)

def process_home_coverage(company_slug: str, quote_id: int, cover: Cover, coverage: dict) -> None:
    company = Company.objects.get(slug=company_slug)
    quote = QuotationHome.objects.get(pk=quote_id)
    validity_type = Validity.objects.get(slug="mensual")
    QuotationHomeCover.objects.create(...)
```

### 1.5 Views Pattern Actual

Todas las apps usan:
- `BaseQuotationCreateView` para crear cotizaciones
- `BaseQuoteDetailView` para ver precios
- `ImprovedDynamicTableHeaderMixin` para listados

---

## 2. Problemas Identificados

| Problema | Impacto | Apps Afectadas |
|----------|---------|----------------|
| Duplicacion de services | Mantenimiento, bugs | vehicles, hogar |
| build_data_class() repetido | Inconsistencia en DTOs | Todas |
| Managers/QuerySets inconsistentes | Solo hogar tiene managers | vehicles, laborales |
| Validacion dispersa | Reglas de negocio en forms y models | Todas |
| Templates acoplados | Logica de precio por company hardcodeada | vehicles |
| Sin patron claro para nuevas apps | Dificultad para escalar | Futuras apps |

---

## 3. Propuesta de Arquitectura

### 3.1 Patron Strategy para Services

**Objetivo**: Eliminar duplicacion en `process_{product}_coverage()`.

```python
from abc import ABC, abstractmethod
from typing import Any, TypeVar, Generic

TQuotation = TypeVar('TQuotation', bound='BaseQuotation')
TCover = TypeVar('TCover', bound='BaseQuotationCover')

class CoverageProcessorStrategy(ABC, Generic[TQuotation, TCover]):

    @abstractmethod
    def get_quotation_model(self) -> type[TQuotation]:
        pass

    @abstractmethod
    def get_cover_model(self) -> type[TCover]:
        pass

    @abstractmethod
    def extract_cover_data(self, coverage: dict[str, Any]) -> dict[str, Any]:
        pass

    def process(self, company_slug: str, quote_id: int, cover: Cover, coverage: dict[str, Any]) -> TCover:
        company = Company.objects.get(slug=company_slug)
        quote = self.get_quotation_model().objects.get(pk=quote_id)
        validity_type = Validity.objects.get(slug="mensual")

        cover_data = {
            'cover': cover,
            'quote': quote,
            'company': company,
            'validity_type': validity_type,
            'fee': coverage.get('premium'),
            'extra_data': json.dumps(coverage),
            'creation_user': quote.creation_user,
            **self.extract_cover_data(coverage)
        }

        return self.get_cover_model().objects.create(**cover_data)


class VehicleCoverageProcessor(CoverageProcessorStrategy[QuotationVehicle, QuotationVehicleCover]):

    def get_quotation_model(self):
        return QuotationVehicle

    def get_cover_model(self):
        return QuotationVehicleCover

    def extract_cover_data(self, coverage: dict[str, Any]) -> dict[str, Any]:
        return {
            'insured_amount': coverage.get('insured_amount'),
        }


class HomeCoverageProcessor(CoverageProcessorStrategy[QuotationHome, QuotationHomeCover]):

    def get_quotation_model(self):
        return QuotationHome

    def get_cover_model(self):
        return QuotationHomeCover

    def extract_cover_data(self, coverage: dict[str, Any]) -> dict[str, Any]:
        return {
            'capital': coverage.get('capital'),
            'description': coverage.get('description'),
            'coverage_code': coverage.get('coverage_code'),
            'detail': coverage.get('detail'),
            'premium_to_validity': coverage.get('premium_to_validity'),
            'detail_type': coverage.get('detail_type'),
        }
```

### 3.2 Patron Factory para DTOs

**Objetivo**: Centralizar la construccion de DTOs con validacion.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import TypeVar, Generic

TQuotation = TypeVar('TQuotation', bound='BaseQuotation')
TDto = TypeVar('TDto', bound='QuoteBase')

class QuoteDTOFactory(ABC, Generic[TQuotation, TDto]):

    def __init__(self, quotation: TQuotation):
        self.quotation = quotation

    def build_person_dto(self) -> Person | None:
        if not self.quotation.person:
            return None

        p = self.quotation.person
        return Person(
            first_name=p.first_name,
            last_name=p.last_name,
            email=p.email,
            tax_type=getattr(p.tax_type, "slug", None),
            person_type=getattr(p.person_type, "slug", None),
            birthdate=p.birthdate.isoformat() if p.birthdate else None,
            sex=getattr(p.sex, "slug", None),
        )

    @abstractmethod
    def build_product_specific_data(self) -> dict:
        pass

    def build(self) -> TDto:
        base_data = {
            'pk': self.quotation.pk,
            'person': self.build_person_dto(),
            'product': getattr(self.quotation.product, 'slug', None),
            'intermediaries': None,
        }
        base_data.update(self.build_product_specific_data())
        return self.get_dto_class()(**base_data)

    @abstractmethod
    def get_dto_class(self) -> type[TDto]:
        pass


class VehicleQuoteDTOFactory(QuoteDTOFactory[QuotationVehicle, QuoteVehicle]):

    def get_dto_class(self):
        return QuoteVehicle

    def build_product_specific_data(self) -> dict:
        q = self.quotation
        vehicle_data = Vehicle(
            ia_version=str(q.ia_version) if q.ia_version else None,
            year=q.year,
            vehicle_use=getattr(q.vehicle_use, "slug", None),
            plate=q.plate.patente if q.plate else None,
            zero_kilometers=q.zero_kilometers,
            fuel_type=getattr(q.fuel_type, "slug", None),
            insured_amount_fuel=q.insured_amount_fuel,
        )
        return {
            'vehicle': vehicle_data,
            'zip_code': q.zip_code,
        }
```

### 3.3 Patron Specification para Validaciones

**Objetivo**: Centralizar reglas de negocio fuera de forms/models.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Generic, TypeVar, Union

T = TypeVar('T')


class Specification(ABC, Generic[T]):

    @abstractmethod
    def is_satisfied_by(self, candidate: T) -> bool:
        pass

    @abstractmethod
    def get_error_message(self) -> str:
        pass

    def and_(self, other: 'Specification[T]') -> 'AndSpecification[T]':
        return AndSpecification(self, other)

    def or_(self, other: 'Specification[T]') -> 'OrSpecification[T]':
        return OrSpecification(self, other)


class AndSpecification(Specification[T]):

    def __init__(self, spec1: Specification[T], spec2: Specification[T]):
        self.spec1 = spec1
        self.spec2 = spec2

    def is_satisfied_by(self, candidate: T) -> bool:
        return (
            self.spec1.is_satisfied_by(candidate)
            and self.spec2.is_satisfied_by(candidate)
        )

    def get_error_message(self) -> str:
        messages: list[str] = []
        messages.append(self.spec1.get_error_message())
        messages.append(self.spec2.get_error_message())
        return " ".join(messages) if messages else ""


class OrSpecification(Specification[T]):

    def __init__(self, spec1: Specification[T], spec2: Specification[T]):
        self.spec1 = spec1
        self.spec2 = spec2

    def is_satisfied_by(self, candidate: T) -> bool:
        return (
            self.spec1.is_satisfied_by(candidate)
            or self.spec2.is_satisfied_by(candidate)
        )

    def get_error_message(self) -> str:
        return (
            f"{self.spec1.get_error_message()} o "
            f"{self.spec2.get_error_message()}"
        )
```

#### 3.3.1 Ejemplo Extendido: Validacion de Cotizacion Vehicular

```python
class VehicleYearSpecification(Specification['QuotationVehicle']):

    def __init__(self, max_age: int = 20):
        self.max_age = max_age

    def is_satisfied_by(self, quote: 'QuotationVehicle') -> bool:
        from django.utils import timezone
        current_year = timezone.now().year
        if not quote.year:
            return False
        return quote.year >= (current_year - self.max_age)

    def get_error_message(self) -> str:
        return f"El vehiculo no puede tener mas de {self.max_age} anios."


class GNCRequiresAmountSpecification(Specification['QuotationVehicle']):

    def is_satisfied_by(self, quote: 'QuotationVehicle') -> bool:
        if quote.fuel_type and quote.fuel_type.slug == "gnc":
            return bool(quote.insured_amount_fuel)
        return True

    def get_error_message(self) -> str:
        return "Debe indicar el monto asegurado del equipo de GNC."


class ZeroKmOrUsedWithPlateSpecification(Specification['QuotationVehicle']):

    def is_satisfied_by(self, quote: 'QuotationVehicle') -> bool:
        if quote.zero_kilometers:
            return True
        return bool(quote.plate)

    def get_error_message(self) -> str:
        return "Vehiculos usados deben tener patente."


class InsuredAmountValidSpecification(Specification['QuotationVehicle']):

    def __init__(self, min_amount: int = 1000000, max_amount: int = 500000000):
        self.min_amount = min_amount
        self.max_amount = max_amount

    def is_satisfied_by(self, quote: 'QuotationVehicle') -> bool:
        if not quote.insured_amount:
            return False
        return self.min_amount <= quote.insured_amount <= self.max_amount

    def get_error_message(self) -> str:
        return f"Suma asegurada debe estar entre ${self.min_amount:,} y ${self.max_amount:,}."


class VehicleQuotationValidationSpecification:

    @staticmethod
    def create_standard_validation() -> Specification:
        year_valid = VehicleYearSpecification(max_age=20)
        gnc_valid = GNCRequiresAmountSpecification()
        plate_valid = ZeroKmOrUsedWithPlateSpecification()
        amount_valid = InsuredAmountValidSpecification()

        return (
            year_valid
            .and_(gnc_valid)
            .and_(plate_valid)
            .and_(amount_valid)
        )

    @staticmethod
    def create_basic_validation() -> Specification:
        year_valid = VehicleYearSpecification(max_age=20)
        amount_valid = InsuredAmountValidSpecification()
        return year_valid.and_(amount_valid)

    @staticmethod
    def create_flexible_year_validation() -> Specification:
        standard_year = VehicleYearSpecification(max_age=20)
        classic_year = VehicleYearSpecification(max_age=50)
        return standard_year.or_(classic_year)
```

#### 3.3.2 Uso en Modelo y Form

```python
class QuotationVehicle(BaseQuotation):

    def clean(self):
        spec = VehicleQuotationValidationSpecification.create_standard_validation()
        if not spec.is_satisfied_by(self):
            raise ValidationError(spec.get_error_message())


class QuotationVehicleForm(forms.ModelForm):

    def clean(self):
        cleaned_data = super().clean()

        temp_quote = QuotationVehicle(**cleaned_data)
        spec = VehicleQuotationValidationSpecification.create_standard_validation()

        if not spec.is_satisfied_by(temp_quote):
            raise forms.ValidationError(spec.get_error_message())

        return cleaned_data
```

### 3.4 Patron Builder para PreSale

**Objetivo**: Simplificar la creacion de PreSale que involucra Person, Document, Address.

```python
from typing import TypeVar, Generic

TPreSale = TypeVar('TPreSale', bound='BasePreSale')


class PreSaleBuilder(Generic[TPreSale]):

    def __init__(self, presale_model: type[TPreSale], user):
        self.presale_model = presale_model
        self.user = user
        self._person_data = {}
        self._document_data = {}
        self._address_data = {}
        self._presale_data = {}

    def with_person(self, first_name: str, last_name: str,
                    birthdate=None, sex=None, email=None) -> 'PreSaleBuilder':
        self._person_data = {
            'first_name': first_name,
            'last_name': last_name,
            'birthdate': birthdate,
            'sex': sex,
            'email': email,
        }
        return self

    def with_document(self, document_number: str, document_type_slug: str = 'dni') -> 'PreSaleBuilder':
        self._document_data = {
            'document_number': document_number,
            'document_type_slug': document_type_slug,
        }
        return self

    def with_address(self, street: str, street_number: str,
                     zip_code: str = None, **kwargs) -> 'PreSaleBuilder':
        self._address_data = {
            'street': street,
            'street_number': street_number,
            'zip_code': zip_code,
            **kwargs
        }
        return self

    def with_quote_cover(self, quote_cover) -> 'PreSaleBuilder':
        self._presale_data['quote_cover'] = quote_cover
        return self

    def with_fields(self, **fields) -> 'PreSaleBuilder':
        self._presale_data.update(fields)
        return self

    def build(self) -> TPreSale:
        person = self._get_or_create_person()

        presale = self.presale_model.objects.create(
            policy_holder=person,
            creation_user=self.user,
            **self._presale_data
        )

        return presale

    def _get_or_create_person(self):
        from person.models import Document, DocumentType, Person, PersonType

        doc_number = self._document_data.get('document_number')
        doc_type_slug = self._document_data.get('document_type_slug', 'dni')

        try:
            existing_doc = Document.objects.get(document_number=doc_number)
            person = existing_doc.person
            for key, value in self._person_data.items():
                if value and getattr(person, key) != value:
                    setattr(person, key, value)
            person.save()
        except Document.DoesNotExist:
            self._person_data['person_type'] = PersonType.objects.get(slug="fisica")
            self._person_data['creation_user'] = self.user
            person = Person.objects.create(**self._person_data)

            Document.objects.create(
                person=person,
                document_number=doc_number,
                document_type=DocumentType.objects.get(slug=doc_type_slug),
                creation_user=self.user,
            )

        return person
```

#### 3.4.1 Ejemplo de Uso: PreSale de Vehiculo

```python
class VehiclePreSaleBuilder(PreSaleBuilder['PreSaleVehicle']):

    def __init__(self, user):
        super().__init__(PreSaleVehicle, user)
        self._vehicle_data = {}

    def with_vehicle(self, plate: str, chassis: str, engine: str) -> 'VehiclePreSaleBuilder':
        self._vehicle_data = {
            'plate': plate,
            'chassis': chassis,
            'engine': engine,
        }
        return self

    def with_payment(self, payment_method: str, cbu: str = None,
                     card_number: str = None) -> 'VehiclePreSaleBuilder':
        self._presale_data['payment_method'] = payment_method
        if cbu:
            self._presale_data['cbu'] = cbu
        if card_number:
            self._presale_data['card_number'] = card_number
        return self

    def build(self) -> 'PreSaleVehicle':
        presale = super().build()

        if self._vehicle_data:
            presale.plate = self._vehicle_data.get('plate')
            presale.chassis = self._vehicle_data.get('chassis')
            presale.engine = self._vehicle_data.get('engine')
            presale.save()

        return presale
```

#### 3.4.2 Uso en View

```python
class PreSaleVehicleCreateView(BasePreSaleCreateView):
    model = PreSaleVehicle
    form_class = PreSaleVehicleForm

    def forms_valid(self, form, person_form):
        quote_cover = QuotationVehicleCover.objects.get(pk=self.kwargs['cover_pk'])

        builder = VehiclePreSaleBuilder(user=self.request.user)

        presale = (
            builder
            .with_person(
                first_name=person_form.cleaned_data['first_name'],
                last_name=person_form.cleaned_data['last_name'],
                birthdate=person_form.cleaned_data['birthdate'],
                email=person_form.cleaned_data['email'],
            )
            .with_document(
                document_number=person_form.cleaned_data['document_number'],
                document_type_slug='dni',
            )
            .with_address(
                street=form.cleaned_data['street'],
                street_number=form.cleaned_data['street_number'],
                zip_code=form.cleaned_data['zip_code'],
            )
            .with_vehicle(
                plate=form.cleaned_data['plate'],
                chassis=form.cleaned_data['chassis'],
                engine=form.cleaned_data['engine'],
            )
            .with_payment(
                payment_method=form.cleaned_data['payment_method'],
                cbu=form.cleaned_data.get('cbu'),
            )
            .with_quote_cover(quote_cover)
            .build()
        )

        return redirect('presale_detail', pk=presale.pk)
```

#### 3.4.3 Uso en Service

```python
class PreSaleService:

    @staticmethod
    def create_from_api_response(user, quote_cover, api_data: dict) -> PreSaleVehicle:
        builder = VehiclePreSaleBuilder(user=user)

        return (
            builder
            .with_person(
                first_name=api_data['holder']['first_name'],
                last_name=api_data['holder']['last_name'],
                birthdate=api_data['holder'].get('birthdate'),
                email=api_data['holder'].get('email'),
            )
            .with_document(
                document_number=api_data['holder']['document_number'],
            )
            .with_vehicle(
                plate=api_data['vehicle']['plate'],
                chassis=api_data['vehicle']['chassis'],
                engine=api_data['vehicle']['engine'],
            )
            .with_quote_cover(quote_cover)
            .with_fields(
                proposal_number=api_data.get('proposal_number'),
                policy_number=api_data.get('policy_number'),
            )
            .build()
        )
```

### 3.5 Base Managers y QuerySets

**Objetivo**: Estandarizar queries comunes para todas las apps.

```python
from django.db import models
from django.db.models import QuerySet
from typing import TypeVar, Self

T = TypeVar('T', bound=models.Model)


class BaseQuotationQuerySet(QuerySet[T]):

    def for_user(self, user) -> Self:
        from core.general_helpers import HierarchyManager
        user_ids = HierarchyManager.get_hierarchical_user_ids(user_id=user.id)
        user_ids.append(user.id)
        return self.filter(creation_user_id__in=user_ids)

    def active(self) -> Self:
        return self.filter(date_deleted__isnull=True)

    def with_product(self, product_slug: str) -> Self:
        return self.filter(product__slug=product_slug)

    def with_status(self, status_slug: str) -> Self:
        return self.filter(status__slug=status_slug)

    def recent(self, days: int = 30) -> Self:
        from django.utils import timezone
        from datetime import timedelta
        cutoff = timezone.now() - timedelta(days=days)
        return self.filter(date_created__gte=cutoff)


class BaseQuotationManager(models.Manager[T]):

    def get_queryset(self) -> BaseQuotationQuerySet[T]:
        return BaseQuotationQuerySet(model=self.model, using=self._db)

    def for_user(self, user):
        return self.get_queryset().for_user(user)

    def active(self):
        return self.get_queryset().active()


class QuotationVehicleQuerySet(BaseQuotationQuerySet['QuotationVehicle']):

    def with_gnc(self) -> Self:
        return self.filter(fuel_type__slug='gnc')

    def zero_km(self) -> Self:
        return self.filter(zero_kilometers=True)


class QuotationVehicleManager(BaseQuotationManager['QuotationVehicle']):

    def get_queryset(self) -> QuotationVehicleQuerySet:
        return QuotationVehicleQuerySet(model=self.model, using=self._db)
```

---

## 4. Estructura Propuesta para Nueva App

```
branchs/{categoria}/{producto}/
    __init__.py
    apps.py
    admin.py
    urls.py

    domain/
        __init__.py
        entities.py
        specifications.py
        dto_factory.py

    models.py
    managers.py

    services/
        __init__.py
        coverage_processor.py
        presale_builder.py

    views.py
    forms.py

    api/
        v1/
            urls.py
            views.py
            serializers.py

    tests/
        test_models.py
        test_services.py
        test_specifications.py
        test_views.py
```

---

## 5. Checklist para Nueva App

### 5.1 Modelos

- [ ] Extender `BaseQuotation` con campos especificos
- [ ] Extender `BaseQuotationCover` con campos especificos
- [ ] Extender `BasePreSale` si requiere emision
- [ ] Implementar `build_data_class()` o usar `QuoteDTOFactory`
- [ ] Crear Manager y QuerySet heredando de base

### 5.2 Domain

- [ ] Crear DTO en `domain/entities.py` extendiendo `QuoteBase`
- [ ] Crear `DTOFactory` para el producto
- [ ] Crear `Specifications` para reglas de negocio

### 5.3 Services

- [ ] Implementar `CoverageProcessorStrategy`
- [ ] Implementar `PreSaleBuilder` si hay emision

### 5.4 Views

- [ ] `CotizarCreateView` extendiendo `BaseQuotationCreateView`
- [ ] `CotizarDetailView` extendiendo `BaseQuoteDetailView`
- [ ] `CotizacionListView` con `ImprovedDynamicTableHeaderMixin`
- [ ] `PreSaleCreateView` y `PreSaleListView` si hay emision

### 5.5 Forms

- [ ] `QuotationForm` como `ModelForm`
- [ ] `QuotationFilter` para listado
- [ ] `PreSaleForm` si hay emision

---

## 6. Migracion Gradual

### Fase 1: Infraestructura Base

1. Crear `commercial_core/managers.py` con `BaseQuotationManager`
2. Crear `src/services/coverage_processor.py` con `CoverageProcessorStrategy`
3. Crear `src/domain/dto_factory.py` con `QuoteDTOFactory`
4. Crear `src/domain/specifications.py` con base `Specification`

### Fase 2: Migrar vehicles/

1. Implementar `VehicleCoverageProcessor`
2. Implementar `VehicleQuoteDTOFactory`
3. Agregar `QuotationVehicleManager`
4. Crear specifications para validaciones
5. Tests

### Fase 3: Migrar hogar/

1. Implementar `HomeCoverageProcessor`
2. Implementar `HomeQuoteDTOFactory`
3. Refactorizar managers existentes
4. Tests

### Fase 4: Documentar Patron

1. Crear `.claude/doc/insurance_apps/` con documentacion
2. Agregar ejemplos de uso
3. Actualizar CLAUDE.md con el patron

---

## 7. Decision Points

Antes de implementar, necesito confirmacion sobre:

1. **Ubicacion de services compartidos**:
   - Opcion A: `src/services/` (actual propuesta)
   - Opcion B: `commercial_core/services/`

2. **Nivel de abstraccion**:
   - Opcion A: Mantener flexibilidad para diferencias entre productos
   - Opcion B: Forzar consistencia estricta

3. **Prioridad de migracion**:
   - Opcion A: Migrar apps existentes primero
   - Opcion B: Crear nueva app como prueba del patron

4. **Tests**:
   - Opcion A: TDD completo (Standard Mode)
   - Opcion B: Tests basicos (Fast Track)

---

## 8. Resumen de Beneficios

| Patron | Beneficio | Ahorro Estimado |
|--------|-----------|-----------------|
| **Strategy (Services)** | Elimina duplicacion en `process_coverage()` | 50% menos codigo por app |
| **Factory (DTOs)** | Construccion consistente de DTOs | 30% menos bugs en APIs |
| **Specification** | Validaciones testeables y reutilizables | 40% menos logica en forms |
| **Builder (PreSale)** | Simplifica creacion de entidades | 60% menos codigo en views |
| **Base Managers** | Queries consistentes | 70% menos duplicacion en querysets |

---

**Siguiente paso**: Confirmar decision points para comenzar implementacion.

---

## 9. Analisis: CoverageProcessorStrategy - Template Method vs Pure Strategy

### 9.1 Estado Actual (Inconsistente)

| Implementacion | Patron Usado | Comportamiento |
|----------------|--------------|----------------|
| **Base Class** (`commercial_core/services/coverage_processor.py`) | Template Method | `process()` concreto con hooks |
| **HomeCoverageProcessor** | Template Method | Sigue el patron base (correcto) |
| **VehicleCoverageProcessor** | Override completo | Reescribe `process()` sin usar base |

**Problema**: VehicleCoverageProcessor ignora el Template Method y duplica la logica de `get_or_create()`.

### 9.2 Opciones Analizadas

#### Option A: Pure Strategy (`process()` como abstractmethod)

```python
class CoverageProcessorStrategy(ABC):
    @abstractmethod
    def process(self, company_slug, quote_id, cover, coverage, *, creation_user=None):
        """Cada producto implementa el flujo completo."""
        pass
```

**Pros:**
- Maxima flexibilidad
- Clara responsabilidad

**Cons:**
- Duplicacion de `get_or_create()` en cada producto (~80 lineas)
- Viola DRY
- Dificil garantizar idempotencia consistente

#### Option B: Template Method (`process()` concreto con hooks)

```python
class CoverageProcessorStrategy(ABC):
    @abstractmethod
    def extract_cover_data(self, coverage): pass  # Hook

    def process(self, company_slug, quote_id, cover, coverage, *, creation_user=None):
        """Algoritmo comun - usa hooks para variaciones."""
        cover_data = self.extract_cover_data(coverage)
        quote_cover, created = self.get_or_create_cover(...)
        if created:
            self.post_process(...)
        return quote_cover

    def post_process(self, quote_cover, coverage, *, creation_user=None):
        """Hook para logica post-creacion (Features, etc.)."""
        pass
```

**Pros:**
- DRY - `get_or_create()` en un solo lugar
- Idempotencia garantizada
- Sigue patron de Django CBVs
- Menos codigo total

**Cons:**
- Menos flexible para productos muy diferentes
- Requiere entender el algoritmo base para override

### 9.3 Recomendacion: Template Method con Soporte Hibrido

**Mantener Option B** con mejoras:

1. **Agregar helper `get_or_create_cover()`** para productos que overridean `process()`:

```python
def get_or_create_cover(self, quotation, cover, cover_data, *, creation_user=None):
    """Helper para mantener idempotencia en overrides."""
    return self.get_cover_model().objects.get_or_create(
        quote=quotation,
        cover=cover,
        company_id=cover.company_id,
        defaults={"creation_user": creation_user, **cover_data},
    )
```

2. **Documentar claramente** cuando usar cada approach:

```
Productos Simples (Home, ART):
- Implementar solo metodos abstractos
- Usar post_process() para Features
- NO overridear process()

Productos Complejos (Auto, Moto):
- Overridear process() si necesario
- USAR get_or_create_cover() para mantener idempotencia
- Documentar POR QUE se overridea
```

3. **Refactorizar VehicleCoverageProcessor** para usar helper:

```python
class VehicleCoverageProcessor(CoverageProcessorStrategy):
    def process(self, company_slug, quote_id, cover, coverage, *, creation_user=None):
        """Override para logica especifica de vehiculos (PaymentMethod, Validity, GNC)."""
        quote = self.get_quotation_model().objects.get(pk=quote_id)

        # Logica especifica de vehiculos
        cover_data = self._build_vehicle_cover_data(quote, coverage, company_slug)

        # Usar helper para idempotencia
        quote_cover, created = self.get_or_create_cover(
            quote, cover, cover_data, creation_user=creation_user
        )
        return quote_cover
```

### 9.4 Comparacion Final

| Criterio | Option A (Strategy) | Option B (Template Method) |
|----------|---------------------|----------------------------|
| DRY | Viola (duplica get_or_create) | Cumple |
| Idempotencia | Inconsistente | Garantizada |
| Lineas de codigo | ~430 (5 productos) | ~350 (5 productos) |
| Testabilidad | Testear proceso completo x5 | Testear base 1x + especificos |
| Patron Django | No | Si (CBV pattern) |
| Flexibilidad | Alta | Alta (con override) |

**Winner: Option B (Template Method)**

### 9.5 Tareas Propuestas

**Fase 1: Actualizar Base Class**
- [ ] Agregar `get_or_create_cover()` helper
- [ ] Mejorar docstring con ejemplos de uso

**Fase 2: Refactorizar VehicleCoverageProcessor**
- [ ] Usar `get_or_create_cover()` helper
- [ ] Documentar por que overridea `process()`

**Fase 3: Tests**
- [ ] Tests de idempotencia en base class
- [ ] Tests de post_process() hook
- [ ] Tests especificos por producto

### 9.6 Decision Point

**Pregunta**: Aprobar Template Method Pattern como estandar para CoverageProcessorStrategy?

- Si se aprueba, se implementaran las tareas de Fase 1-3
- Si no, se puede discutir alternativas

---

**Fecha de analisis**: 2025-12-18
