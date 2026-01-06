# PROJECT_SPECS.md - Sistema de Cotizacion de Seguros Multi-Producto

## Resumen Ejecutivo

Sistema de cotizacion y emision de seguros para multiples productos (vehiculos, hogar, ART, etc.) con integracion a APIs de companias aseguradoras externas (Sancor, Galicia, Experta, etc.).

---

## 1. Arquitectura General

### 1.1 Estructura de Directorios

```
visred-socios/
    branchs/                    # Apps de productos de seguros
        patrimoniales/
            vehicles/           # Autos y motos
            home/               # Hogar
        laborales/              # ART (riesgos de trabajo)
        agricultura/            # Seguros agricolas
        personas/               # Seguros de personas

    commercial_core/            # Modelos y servicios base compartidos
        models.py               # BaseQuotation, BaseQuotationCover, BasePreSale
        views.py                # BaseQuotationCreateView, BaseQuoteDetailView, BaseGetTaskView
        domain/
            specifications.py   # Specification Pattern base
        services/
            __init__.py         # Exporta todos los servicios
            quote.py            # QuoteService (orquestador de cotizaciones)
            coverage_processor.py  # Strategy Pattern para procesamiento
            dto_factory.py      # Factory Pattern para DTOs
            presale_builder.py  # Builder Pattern para PreSales
            task_response.py    # Servicio para polling de tareas

    commercial_structure/       # Estructura comercial (organizadores, productores)
        models.py               # CommercialStructure, LevelType

    companies/                  # Integraciones con companias aseguradoras
        sancor/
            handlers/           # Handlers por producto (auto.py, home.py)
            models.py           # Producer, Locality, etc.
        galicia/
        experta/
        ...

    core/                       # Utilidades y modelos generales
        domain/
            entities.py         # DTOs base (RiskBase, Person, Address, etc.)
        models.py               # BaseEntity, Company, Product, etc.
        tasks.py                # Celery tasks (cotizar, emitir)
        client.py               # ApiClient para llamadas a APIs externas

    covers/                     # Sistema de coberturas
        models.py               # Feature, Cover, CoverFeature, CoverGroup
```

### 1.2 Flujo de Cotizacion

```
1. Usuario llena formulario (QuotationCreateView)
         |
2. Se crea QuotationXXX (modelo extendido de BaseQuotation)
         |
3. Redirige a QuotationDetailView (precios.html)
         |
4. QuoteService.execute_company_tasks()
         |
5. Para cada compania activa:
    a. Obtiene producer y locality de la compania
    b. Construye risk_data_class via quote.build_data_class()
    c. Ejecuta celery task: cotizar.delay(company_slug, risk_dict)
         |
6. core/tasks.py -> cotizar()
    a. Importa dinamicamente: companies/{company}/handlers/{product}.py
    b. Ejecuta funcion cotizar(risk_dict)
    c. Llama a API externa de la compania
    d. QuoteService.store_quote_result() guarda resultados
         |
7. Frontend polling via GetTaskView obtiene resultados y renderiza precios
```

### 1.3 Patron de Handlers por Compania

```python
# companies/{company}/handlers/{product}.py

def cotizar(risk_dict: dict) -> dict:
    """
    Cotiza el riesgo con la API de la compania.

    Args:
        risk_dict: Payload normalizado con datos del riesgo

    Returns:
        {"ok": bool, "response": list[dict], "msg": str, "company": str}
    """
    # 1. Validar entrada usando Specifications
    # 2. Obtener datos de cache (locality, cover_codes)
    # 3. Construir payload usando TypedDicts
    # 4. Llamar a API externa via ApiClient
    # 5. Normalizar respuesta usando dataclasses
    # 6. Retornar resultado normalizado


def emitir(risk_dict: dict) -> dict:
    """
    Emite la poliza con la API de la compania.
    """
    pass
```

---

## 2. Patrones DDD Generalizados

### 2.1 Specification Pattern (`commercial_core/domain/specifications.py`)

Encapsula reglas de negocio como objetos testeables y combinables.

```python
from commercial_core.domain.specifications import Specification

class Specification(ABC, Generic[T]):
    @abstractmethod
    def is_satisfied_by(self, candidate: T) -> bool: ...

    @abstractmethod
    def get_error_message(self) -> str: ...

    def and_(self, other: "Specification[T]") -> "AndSpecification[T]": ...
    def or_(self, other: "Specification[T]") -> "OrSpecification[T]": ...
    def not_(self) -> "NotSpecification[T]": ...
```

**Uso en apps:**
```python
# branchs/patrimoniales/home/domain/specifications.py
from commercial_core.domain.specifications import Specification

class ZipCodeValidSpec(Specification[str]):
    def is_satisfied_by(self, zip_code: str) -> bool:
        return zip_code.isdigit() and len(zip_code) == 4

    def get_error_message(self) -> str:
        return "El codigo postal debe tener 4 digitos"

# Uso en handler:
spec = ZipCodeValidSpec()
if not spec.is_satisfied_by(zip_code):
    return QuoteResponse.error(company=company_slug, msg=spec.get_error_message()).to_dict()
```

### 2.2 Strategy Pattern - Coverage Processor (`commercial_core/services/coverage_processor.py`)

Procesa coberturas de APIs externas de manera uniforme.

```python
from commercial_core.services import CoverageProcessorStrategy, CoverageProcessorRegistry

class CoverageProcessorStrategy(ABC, Generic[TQuotation, TCover]):
    @abstractmethod
    def get_quotation_model(self) -> type[TQuotation]: ...

    @abstractmethod
    def get_cover_model(self) -> type[TCover]: ...

    @abstractmethod
    def extract_cover_data(self, coverage: dict[str, Any]) -> dict[str, Any]: ...

    def process(self, company_slug, quote_id, cover, coverage, *, creation_user) -> TCover:
        # Logica comun de procesamiento
        ...

    def post_process(self, quote_cover: TCover, coverage: dict) -> None:
        # Hook para procesamiento adicional (Features, etc.)
        ...
```

**Implementacion por producto:**
```python
# branchs/patrimoniales/home/services/coverage_processor.py
from commercial_core.services import CoverageProcessorStrategy, CoverageProcessorRegistry

class HomeCoverageProcessor(CoverageProcessorStrategy[QuotationHome, QuotationHomeCover]):
    def get_quotation_model(self) -> type[QuotationHome]:
        return QuotationHome

    def get_cover_model(self) -> type[QuotationHomeCover]:
        return QuotationHomeCover

    def extract_cover_data(self, coverage: dict) -> dict:
        return {
            "fee": coverage.get("fee"),
            "coverage_code": coverage.get("cover_code"),
            # ... campos especificos
        }

# Registrar el procesador
CoverageProcessorRegistry.register("home", HomeCoverageProcessor)
```

### 2.3 Factory Pattern - DTO Factory (`commercial_core/services/dto_factory.py`)

Centraliza construccion de DTOs para cotizaciones.

```python
from commercial_core.services import QuoteDTOFactory

class QuoteDTOFactory(ABC, Generic[TQuotation, TDto]):
    def __init__(self, quotation: TQuotation, *, producer=None, locality=None): ...

    def build_person_dto(self) -> Person | None: ...
    def build_intermediaries_dto(self) -> Intermediaries | None: ...
    def build_address_dto(self) -> Address | None: ...

    @abstractmethod
    def build_product_specific_data(self) -> dict[str, Any]: ...

    @abstractmethod
    def get_dto_class(self) -> type[TDto]: ...

    def build(self) -> TDto: ...
```

**Implementacion por producto:**
```python
# branchs/patrimoniales/home/services/dto_factory.py
class HomeQuoteDTOFactory(QuoteDTOFactory[QuotationHome, QuoteHomeDTO]):
    def get_dto_class(self) -> type[QuoteHomeDTO]:
        return QuoteHomeDTO

    def build_product_specific_data(self) -> dict[str, Any]:
        return {
            "property_info": PropertyInfo(...),
            "aditionals": get_default_additionals(),
            "cover_codes": get_default_cover_codes(),
            "locality": Locality(zip_code=self.quotation.zip_code),
        }
```

### 2.4 Builder Pattern - PreSale Builder (`commercial_core/services/presale_builder.py`)

Construccion fluida de PreSales con muchos campos.

```python
from commercial_core.services import PreSaleBuilder

presale = (
    PreSaleBuilder(PreSaleHome)
    .with_policy_holder(person)
    .with_quote_cover(quote_cover)
    .with_address(street, number, floor, apartment)
    .with_contact(email, phone_prefix, phone_number)
    .with_locality(id_localidad)
    .with_creation_user(user)
    .build_and_save()
)
```

---

## 3. Sistema de Polling de Tareas

### 3.1 TaskResponseConfig y build_task_response (`commercial_core/services/task_response.py`)

Servicio generalizado para obtener resultados de tareas Celery via AJAX.

```python
from commercial_core.services import TaskResponseConfig, build_task_response

@dataclass
class TaskResponseConfig:
    price_tag_template: str           # Template para tarjetas de precio
    price_modal_template: str | None  # Template para modales (opcional)
    include_modals: bool = True       # Si renderiza modales

# Configuraciones predefinidas
VEHICLE_TASK_CONFIG = TaskResponseConfig(
    price_tag_template="inline/price_vehicle_tag.html",
    price_modal_template="inline/price_modal.html",
)

HOME_TASK_CONFIG = TaskResponseConfig(
    price_tag_template="inline/price_home_tag.html",
    price_modal_template="inline/price_home_modal.html",
)

ART_TASK_CONFIG = TaskResponseConfig(
    price_tag_template="inline/price_tag.html",
    price_modal_template=None,
    include_modals=False,
)
```

### 3.2 BaseGetTaskView (`commercial_core/views.py`)

Vista base para endpoints de polling de tareas.

```python
from commercial_core.views import BaseGetTaskView
from commercial_core.services import HOME_TASK_CONFIG

class GetTaskView(BaseGetTaskView):
    """Vista para obtener resultados de tareas via AJAX."""
    task_config = HOME_TASK_CONFIG
```

**URL pattern:**
```python
# urls.py de cada app
path("get-task/<str:task_id>/", GetTaskView.as_view(), name="get-task"),
```

**Uso en template (precios.html):**
```javascript
// Usar Django URL tag para generar URL dinamica
const getTaskBaseUrl = "{% url 'home:get-task' task_id='TASK_ID_PLACEHOLDER' %}"

allTasks.forEach((taskId) => {
    const taskUrl = getTaskBaseUrl.replace('TASK_ID_PLACEHOLDER', taskId)
    getTask(taskUrl)
})
```

---

## 4. Domain Layer por Producto

### 4.1 Estructura Recomendada

```
branchs/{branch}/{product}/domain/
    __init__.py              # Exporta todas las entidades
    entities.py              # DTOs, Value Objects, TypedDicts, Response dataclasses
    specifications.py        # Reglas de validacion del producto
```

### 4.2 TypedDicts para Payloads de API

Usar TypedDict para payloads que se envian a APIs externas (sin comportamiento).

```python
# branchs/patrimoniales/home/domain/entities.py
from typing import TypedDict

class LocalityPayload(TypedDict):
    locality_code: int | None
    zip_code: str | None

class AdditionalPayload(TypedDict):
    code: str
    option: str

class QuotePayload(TypedDict):
    cover_codes: list[int]
    additionals: list[AdditionalPayload]
    locality: LocalityPayload
    organizer_code: int | None
    producer_code: int | None
    validity_from: str
```

### 4.3 Dataclasses para Respuestas Normalizadas

Usar dataclasses para normalizar respuestas de APIs (con comportamiento).

```python
from dataclasses import dataclass, field
from decimal import Decimal

@dataclass
class CoverageDetail:
    """Detalle de una cobertura individual."""
    coverage_code: int
    description: str
    capital: Decimal | None = None
    fee: Decimal | None = None

    @classmethod
    def from_api_response(cls, data: dict) -> "CoverageDetail":
        return cls(
            coverage_code=data.get("coverage_code", 0),
            description=data.get("description", ""),
            capital=Decimal(str(data["capital"])) if data.get("capital") else None,
            fee=Decimal(str(data["fee"])) if data.get("fee") else None,
        )


@dataclass
class CoverageResult:
    """Resultado normalizado de una cobertura cotizada."""
    code: str
    name: str
    fee: Decimal | None
    coverages: list[CoverageDetail] = field(default_factory=list)

    @classmethod
    def from_api_response(cls, data: dict) -> "CoverageResult":
        # Normaliza cover_code -> code
        return cls(
            code=str(data.get("cover_code", "")),
            name=f"Cobertura {data.get('cover_code')}",
            fee=Decimal(str(data["fee"])) if data.get("fee") else None,
            coverages=[CoverageDetail.from_api_response(c) for c in data.get("coverages", [])],
        )

    def to_dict(self) -> dict:
        # Serializa para response JSON
        ...


@dataclass
class QuoteResponse:
    """Respuesta completa de cotizacion del handler."""
    ok: bool
    company: str
    msg: str = ""
    response: list[CoverageResult] = field(default_factory=list)
    count: int = 0

    @classmethod
    def success(cls, company: str, results: list[CoverageResult]) -> "QuoteResponse":
        return cls(ok=True, company=company, response=results, count=len(results))

    @classmethod
    def error(cls, company: str, msg: str) -> "QuoteResponse":
        return cls(ok=False, company=company, msg=msg)

    def to_dict(self) -> dict:
        # Serializa para retorno del handler
        ...
```

### 4.4 Value Objects (Inmutables con Validacion)

```python
@dataclass(frozen=True)
class PropertyInfo:
    """Value Object para informacion de la propiedad."""
    property_type: str | None = None
    amount: float | None = None
    zip_code: str | None = None
```

### 4.5 DTOs para Cotizacion (Extiende RiskBase)

```python
from core.domain.entities import RiskBase

@dataclass
class QuoteHomeDTO(RiskBase):
    """DTO para cotizacion de hogar."""
    property_info: PropertyInfo | None = None
    aditionals: list[dict[str, str]] | None = None
    cover_codes: list[int] | None = None
    locality: Locality | None = None

    def to_api_payload(self) -> dict:
        # Convierte al formato esperado por la API
        ...
```

---

## 5. Services Layer por Producto

### 5.1 Estructura Recomendada

```
branchs/{branch}/{product}/services/
    __init__.py              # Exporta todos los servicios
    coverage_processor.py    # Strategy para procesar coberturas
    dto_factory.py           # Factory para construir DTOs
    locality_cache.py        # Cache para datos de API (localidades)
    cover_codes_cache.py     # Cache para cover_codes de API
```

### 5.2 Cache Services para APIs Externas

Evitar llamadas repetitivas a APIs externas usando cache con TTL.

```python
# branchs/patrimoniales/home/services/locality_cache.py
from dataclasses import dataclass
from django.core.cache import cache

CACHE_KEY = "home_locality_{company_slug}_{zip_code}"
CACHE_TTL = 86400  # 24 horas

@dataclass
class LocalityInfo:
    locality_code: int
    zip_code: str
    name: str = ""
    province: str = ""

    @classmethod
    def from_api_response(cls, data: dict, zip_code: str) -> "LocalityInfo":
        return cls(
            locality_code=data.get("id_locality") or data.get("locality_code") or 0,
            zip_code=data.get("zip_code") or zip_code,
            name=data.get("name") or "",
            province=data.get("province") or "",
        )


class LocalityCache:
    def __init__(self, company_slug: str = "sancor"):
        self.company_slug = company_slug

    def get_locality(self, zip_code: str, force_refresh: bool = False) -> LocalityInfo | None:
        cache_key = CACHE_KEY.format(company_slug=self.company_slug, zip_code=zip_code)

        if not force_refresh:
            cached = cache.get(cache_key)
            if cached:
                return cached

        locality = self._fetch_from_api(zip_code)
        if locality:
            cache.set(cache_key, locality, CACHE_TTL)
        return locality

    def _fetch_from_api(self, zip_code: str) -> LocalityInfo | None:
        # Llama a API /params/localities/zip-code/{zip_code}/
        ...


# Funcion de conveniencia
def get_locality_by_zip_code(zip_code: str, company_slug: str = "sancor") -> LocalityInfo | None:
    return LocalityCache(company_slug).get_locality(zip_code)
```

---

## 6. Modelos Base (commercial_core)

### 6.1 BaseQuotation

```python
class BaseQuotation(BaseEntity):
    product = ForeignKey(Product)
    person = ForeignKey(Person)
    status = ForeignKey(QuoteStatus)
    company_quotes = GenericRelation("QuoteCompanyQuote")

    @abstractmethod
    def build_data_class(self, company=None, *, producer=None, locality=None) -> RiskBase:
        """Construye el DTO usando la Factory del producto."""
        pass
```

### 6.2 BaseQuotationCover

```python
class BaseQuotationCover(BaseEntity):
    content_type = ForeignKey(ContentType, limit_choices_to={
        "model__in": ["quotationart", "quotationvehicle", "quotationhome", ...]
    })
    object_id = PositiveIntegerField()
    quote = GenericForeignKey()
    fee = FloatField()
    extra_data = JSONField()
    company = ForeignKey(Company)
```

### 6.3 BasePreSale

```python
class BasePreSale(BaseEntity):
    policy_holder = ForeignKey(Person)
    policy_number = CharField()
    proposal_number = CharField()
    status = ForeignKey(SaleStatus)
    content_type = ForeignKey(ContentType, limit_choices_to={
        "model__in": ["quotationartcover", "quotationvehiclecover", "quotationhomecover", ...]
    })
    object_id = PositiveIntegerField()
    quote_cover = GenericForeignKey()
```

---

## 7. Views Base (commercial_core)

### 7.1 BaseQuotationCreateView

```python
class BaseQuotationCreateView(HtmxMixin, CreateView):
    person_form_class = PersonForm

    def process_person_initial(self) -> dict: ...      # Hook para inicializar person form
    def process_person_form(self, form) -> Form: ...   # Hook para modificar person form
    def process_quotation(self, quotation) -> Any: ... # Hook pre-save
    def process_post_quotation(self, quotation) -> Any: ... # Hook post-save
```

### 7.2 BaseQuoteDetailView

```python
class BaseQuoteDetailView(HtmxMixin, DetailView):
    template_name = "precios.html"

    def get_product_slug(self, quote) -> str: ...
    def pre_process_companies(self, quote) -> bool: ...  # Hook opcional
```

### 7.3 BaseGetTaskView

```python
class BaseGetTaskView(JSONResponseMixin, TemplateView):
    task_config: TaskResponseConfig | None = None

    def get_task_config(self) -> TaskResponseConfig | None:
        return self.task_config

    def get_data(self, context) -> dict:
        return build_task_response(self.kwargs["task_id"], config=self.get_task_config())
```

---

## 8. Templates por Producto

### 8.1 Estructura de Templates

```
branchs/{branch}/{product}/templates/
    cotizar.html              # Formulario de cotizacion
    precios.html              # Vista de resultados con polling
    inline/
        price_{product}_tag.html    # Tarjeta de precio
        price_{product}_modal.html  # Modal con detalles
```

### 8.2 Template precios.html - Polling Dinamico

```html
{% extends 'base.html' %}

{% block content %}
<div class="container-xl">
    <!-- Datagrid con datos especificos del producto -->
    <div class="card">
        <div class="datagrid">
            <div class="datagrid-item">
                <div class="datagrid-title">Campo Especifico</div>
                <div class="datagrid-content">{{ object.campo_especifico }}</div>
            </div>
            <!-- ... mas campos -->
        </div>
    </div>

    <!-- Tabs de coberturas -->
    <div class="tab-content">
        {% for cover_group in cover_groups %}
        <div class="tab-pane" id="{{ cover_group.slug }}">
            <div class="row row-cards" id="item-group-company-{{ cover_group.id }}"></div>
        </div>
        {% endfor %}
    </div>

    <div id="quotes-modals-container"></div>
</div>

<script>
document.addEventListener('DOMContentLoaded', function() {
    let allTasks = {{ object.tasks_list | safe }}
    // URL dinamica usando Django URL tag
    const getTaskBaseUrl = "{% url 'producto:get-task' task_id='TASK_ID_PLACEHOLDER' %}"

    allTasks.forEach((taskId) => {
        const taskUrl = getTaskBaseUrl.replace('TASK_ID_PLACEHOLDER', taskId)
        getTask(taskUrl)
    })

    function getTask(taskUrl) {
        fetch(taskUrl)
            .then(response => response.json())
            .then(data => {
                if (data.status === 'SUCCESS') {
                    // Renderizar quotes y modals
                    Object.keys(data.quotes).forEach(groupId => {
                        let container = document.getElementById('item-group-company-' + groupId)
                        data.quotes[groupId].forEach(html => {
                            container.innerHTML += html
                        })
                    })
                    data.modals.forEach(html => {
                        document.getElementById('quotes-modals-container').innerHTML += html
                    })
                } else if (data.status !== 'FAILURE') {
                    setTimeout(() => getTask(taskUrl), 200)
                }
            })
    }
})
</script>
{% endblock %}
```

---

## 9. Checklist para Nueva App de Seguro

### 9.1 Modelos

- [ ] Extender `BaseQuotation` con campos especificos del producto
- [ ] Extender `BaseQuotationCover` con campos especificos
- [ ] Extender `BasePreSale` si requiere emision
- [ ] Implementar `build_data_class()` usando Factory del producto
- [ ] Agregar modelo a `limit_choices_to` en:
  - `QuoteCompanyQuote.content_type`
  - `BaseQuotationCover.content_type`
  - `BasePreSale.content_type`

### 9.2 Domain Layer

- [ ] Crear `domain/__init__.py` exportando todas las entidades
- [ ] Crear `domain/entities.py` con:
  - TypedDicts para payloads de API
  - Dataclasses para respuestas normalizadas (CoverageResult, QuoteResponse)
  - DTO del producto extendiendo RiskBase
  - Value Objects si aplica
- [ ] Crear `domain/specifications.py` con reglas de validacion

### 9.3 Services Layer

- [ ] Crear `services/__init__.py` exportando todos los servicios
- [ ] Crear `services/coverage_processor.py`:
  - Implementar `{Product}CoverageProcessor`
  - Registrar en `CoverageProcessorRegistry`
- [ ] Crear `services/dto_factory.py`:
  - Implementar `{Product}QuoteDTOFactory`
- [ ] Crear cache services si usa datos de API externa:
  - `services/locality_cache.py`
  - `services/cover_codes_cache.py`

### 9.4 Handlers

- [ ] Crear `companies/{company}/handlers/{product}.py`
- [ ] Implementar `cotizar(risk_dict) -> dict`:
  - Validar con Specifications
  - Usar cache services para datos de API
  - Construir payload con TypedDicts
  - Normalizar respuesta con dataclasses
- [ ] Implementar `emitir(risk_dict) -> dict` (opcional)

### 9.5 Views

- [ ] `QuotationCreateView` extendiendo `BaseQuotationCreateView`
- [ ] `QuotationDetailView` extendiendo `BaseQuoteDetailView`
- [ ] `GetTaskView` extendiendo `BaseGetTaskView` con config del producto
- [ ] `QuotationListView` con `ImprovedDynamicTableHeaderMixin`

### 9.6 URLs

- [ ] Registrar URLs en `urls.py` de la app:
  ```python
  urlpatterns = [
      path("cotizar/", QuotationCreateView.as_view(), name="cotizar"),
      path("ver-cotizacion/<int:pk>/", QuotationDetailView.as_view(), name="ver"),
      path("get-task/<str:task_id>/", GetTaskView.as_view(), name="get-task"),
      # ... mas URLs
  ]
  ```
- [ ] Incluir en el URLconf principal

### 9.7 Templates

- [ ] `cotizar.html` - Formulario de cotizacion
- [ ] `precios.html` - Vista de resultados con:
  - Datagrid con campos especificos del producto
  - URL dinamica para get-task
- [ ] `inline/price_{product}_tag.html` - Tarjeta de precio
- [ ] `inline/price_{product}_modal.html` - Modal con detalles

### 9.8 Configuracion

- [ ] Agregar `TaskResponseConfig` para el producto en `task_response.py`
- [ ] Crear management command para setup inicial:
  - Branch (si no existe)
  - Product
  - CompanyProduct
  - Group y Permission `product_{product}`

### 9.9 Tests

- [ ] Tests de DTOs y Value Objects
- [ ] Tests de Specifications
- [ ] Tests de Services (CoverageProcessor, DTOFactory, Cache)
- [ ] Tests de Handler
- [ ] Tests de Views

### 9.10 Admin

- [ ] Registrar modelos en Django Admin

---

## 10. Convencion de Nombres

### 10.1 Modelos

```
Quotation{Product}          # QuotationHome, QuotationVehicle
Quotation{Product}Cover     # QuotationHomeCover, QuotationVehicleCover
PreSale{Product}            # PreSaleHome, PreSaleVehicle
```

### 10.2 DTOs y Entidades

```
Quote{Product}DTO           # QuoteHomeDTO (extiende RiskBase)
{Field}Payload              # LocalityPayload, QuotePayload (TypedDicts)
{Field}Info                 # PropertyInfo, LocalityInfo (Value Objects)
CoverageResult              # Respuesta normalizada de cobertura
QuoteResponse               # Respuesta completa del handler
```

### 10.3 Services

```
{Product}CoverageProcessor  # HomeCoverageProcessor
{Product}QuoteDTOFactory    # HomeQuoteDTOFactory
{Entity}Cache               # LocalityCache, CoverCodesCache
get_{entity}_by_{field}()   # get_locality_by_zip_code()
```

### 10.4 Specifications

```
{Field}ValidSpec            # ZipCodeValidSpec
{Field}PositiveSpec         # AmountPositiveSpec
{Entity}ExistsSpec          # LocalityExistsSpec
```

### 10.5 Views

```
QuotationCreateView         # Vista crear cotizacion
QuotationDetailView         # Vista ver precios
GetTaskView                 # Vista polling de tareas
QuotationListView           # Vista lista de cotizaciones
```

### 10.6 Templates

```
cotizar.html                # Formulario
precios.html                # Resultados
inline/price_{product}_tag.html   # Tarjeta
inline/price_{product}_modal.html # Modal
```

---

## 11. Variables de Entorno

```bash
ENVIRONMENT=dev|test|prod       # Controla comportamiento de tasks
CELERY_BROKER_URL=redis://...   # Broker para Celery
DATABASE_URL=postgres://...     # Base de datos
```

---

## 12. Implementacion de Referencia: Home (Hogar)

### 12.1 Modelos Implementados

**QuotationHome** (`branchs/patrimoniales/home/models.py`):
```python
class QuotationHome(BaseQuotation):
    """Cotizacion de seguro de hogar."""
    property = ForeignKey(PropertyType, on_delete=CASCADE)
    amount = DecimalField(max_digits=12, decimal_places=2)
    zip_code = CharField(max_length=10)

    # GenericRelations
    covers = GenericRelation("QuotationHomeCover")
    company_quotes = GenericRelation("QuoteCompanyQuote")

    def build_data_class(self, company=None, *, producer=None, locality=None) -> QuoteHomeDTO:
        """Usa HomeQuoteDTOFactory para construir el DTO."""
        return HomeQuoteDTOFactory(self, producer=producer, locality=locality).build()
```

**QuotationHomeCover** (`branchs/patrimoniales/home/models.py`):
```python
class QuotationHomeCover(BaseQuotationCover):
    """Cobertura cotizada para hogar."""
    cover = ForeignKey(Cover, on_delete=CASCADE)
    coverage_code = CharField(max_length=20)
    quotation_number = CharField(max_length=50, blank=True)  # Para emision
    description = TextField(blank=True)
    detail = TextField(blank=True)
    detail_type = CharField(max_length=50, blank=True)
    validity_type = ForeignKey(Validity, null=True)
    hired = BooleanField(default=False)  # Si fue contratado

    # GenericRelations
    pre_sales = GenericRelation("PreSaleHome")
```

**PreSaleHome** (`branchs/patrimoniales/home/models.py`):
```python
class PreSaleHome(Address, BasePreSale):
    """Pre-venta/propuesta de seguro de hogar."""
    # Hereda Address: street, street_number, floor, apartment, id_localidad
    email = EmailField()
    phone_prefix = CharField(max_length=5)
    phone_number = CharField(max_length=15)
    validity_from = DateField()
    policy_notification_send = BooleanField(default=False)
```

### 12.2 Domain Layer Implementado

**entities.py** (`branchs/patrimoniales/home/domain/entities.py`):
```python
# TypedDicts para API payloads
class LocalityPayload(TypedDict):
    locality_code: int | None
    zip_code: str | None

class AdditionalPayload(TypedDict):
    code: str
    option: str

class QuotePayload(TypedDict):
    cover_codes: list[int]
    additionals: list[AdditionalPayload]
    locality: LocalityPayload
    organizer_code: int | None
    producer_code: int | None
    validity_from: str

# Value Objects
@dataclass(frozen=True)
class PropertyInfo:
    property_type: str | None = None
    amount: float | None = None
    zip_code: str | None = None

@dataclass(frozen=True)
class Locality:
    zip_code: int | None = None

# DTOs
@dataclass
class QuoteHomeDTO(RiskBase):
    property_info: PropertyInfo | None = None
    aditionals: list[dict[str, str]] | None = None
    cover_codes: list[int] | None = None
    locality: Locality | None = None

    def to_api_payload(self) -> dict:
        """Transforma DTO a formato API Sancor."""
        ...
```

**specifications.py** (`branchs/patrimoniales/home/domain/specifications.py`):
```python
class ZipCodeValidSpec(Specification[str]):
    """Valida codigo postal argentino (4 digitos)."""
    def is_satisfied_by(self, zip_code: str) -> bool:
        return zip_code.isdigit() and len(zip_code) == 4

    def get_error_message(self) -> str:
        return "El codigo postal debe tener 4 digitos"

class AmountPositiveSpec(Specification[float]):
    """Valida que el monto sea positivo."""
    def is_satisfied_by(self, amount: float) -> bool:
        return amount > 0

    def get_error_message(self) -> str:
        return "El monto asegurado debe ser mayor a 0"

class PropertyTypeValidSpec(Specification[str]):
    """Valida tipo de propiedad permitido."""
    VALID_TYPES = ["casa", "departamento", "ph", "country"]

    def is_satisfied_by(self, prop_type: str) -> bool:
        return prop_type.lower() in self.VALID_TYPES

    def get_error_message(self) -> str:
        return f"Tipo de propiedad debe ser: {', '.join(self.VALID_TYPES)}"
```

### 12.3 Services Implementados

**HomeCoverageProcessor** (`branchs/patrimoniales/home/services/coverage_processor.py`):
```python
class HomeCoverageProcessor(CoverageProcessorStrategy[QuotationHome, QuotationHomeCover]):
    def get_quotation_model(self) -> type[QuotationHome]:
        return QuotationHome

    def get_cover_model(self) -> type[QuotationHomeCover]:
        return QuotationHomeCover

    def extract_cover_data(self, coverage: dict) -> dict:
        return {
            "fee": coverage.get("fee"),
            "coverage_code": coverage.get("code"),
            "quotation_number": coverage.get("id_for_proposal", ""),
            "description": coverage.get("description", ""),
            "detail": coverage.get("detail", ""),
            "detail_type": coverage.get("detail_type", ""),
            "extra_data": coverage,  # Guardar respuesta completa
        }

    def post_process(self, quote_cover: QuotationHomeCover, coverage: dict) -> None:
        """Crea Features y CoverFeatures desde la respuesta API."""
        for item in coverage.get("coverages", []):
            feature, _ = Feature.objects.get_or_create(
                slug=slugify(item["description"]),
                product=quote_cover.cover.product,
                defaults={"name": item["description"]}
            )
            CoverFeature.objects.get_or_create(
                cover=quote_cover.cover,
                feature=feature,
            )

# Registro en el registry
CoverageProcessorRegistry.register("home", HomeCoverageProcessor)
```

**HomeQuoteDTOFactory** (`branchs/patrimoniales/home/services/dto_factory.py`):
```python
class HomeQuoteDTOFactory(QuoteDTOFactory[QuotationHome, QuoteHomeDTO]):
    def get_dto_class(self) -> type[QuoteHomeDTO]:
        return QuoteHomeDTO

    def build_product_specific_data(self) -> dict[str, Any]:
        return {
            "property_info": PropertyInfo(
                property_type=self.quotation.property.slug,
                amount=float(self.quotation.amount),
                zip_code=self.quotation.zip_code,
            ),
            "aditionals": get_default_additionals(),
            "cover_codes": get_default_cover_codes(),
            "locality": Locality(zip_code=int(self.quotation.zip_code)),
        }
```

**LocalityCache** (`branchs/patrimoniales/home/services/locality_cache.py`):
```python
CACHE_KEY = "home_locality_{company_slug}_{zip_code}"
CACHE_TTL = 86400  # 24 horas

@dataclass
class LocalityInfo:
    locality_code: int
    zip_code: int
    name: str = ""
    province: str = ""

class LocalityCache:
    def __init__(self, company_slug: str = "sancor"):
        self.company_slug = company_slug

    def get_locality(self, zip_code: str, force_refresh: bool = False) -> LocalityInfo | None:
        cache_key = CACHE_KEY.format(company_slug=self.company_slug, zip_code=zip_code)

        if not force_refresh:
            cached = cache.get(cache_key)
            if cached:
                return cached

        locality = self._fetch_from_api(zip_code)
        if locality:
            cache.set(cache_key, locality, CACHE_TTL)
        return locality

    def _fetch_from_api(self, zip_code: str) -> LocalityInfo | None:
        client = ApiClient(
            company=Company.objects.get(slug=self.company_slug),
            product=Product.objects.get(slug="home"),
        )
        is_ok, result = client.get(f"params/localities/zip-code/{zip_code}/")
        if is_ok and result:
            return LocalityInfo.from_api_response(result, zip_code)
        return None
```

### 12.4 Handler Sancor Home

**companies/sancor/handlers/home.py**:
```python
def cotizar(risk_dict: dict) -> dict[str, Any]:
    """
    Cotiza seguro de hogar con Sancor.

    Args:
        risk_dict: Diccionario con datos del riesgo (de QuoteHomeDTO._dict())

    Returns:
        dict con formato QuoteResponse.to_dict()
    """
    company_slug = "sancor"

    # 1. Validar entrada usando Specifications
    is_valid, error_msg = _validate_input(risk_dict)
    if not is_valid:
        return QuoteResponse.error(company=company_slug, msg=error_msg).to_dict()

    # 2. Obtener instancias
    company = Company.objects.get(slug=company_slug)
    quotation = QuotationHome.objects.get(pk=risk_dict["quote_pk"])

    # 3. Obtener locality del cache
    locality = get_locality_by_zip_code(quotation.zip_code, company_slug)
    if not locality:
        return QuoteResponse.error(
            company=company.name,
            msg="No se encontro la localidad para el codigo postal"
        ).to_dict()

    # 4. Construir payload
    payload = _build_quote_payload(risk_dict, locality)

    # 5. Llamar API
    client = ApiClient(company, Product.objects.get(slug="home"))
    is_ok, result = client.post("home/quote/", payload, quotation)

    if not is_ok:
        return QuoteResponse.error(company=company.name, msg=result or "Error en API").to_dict()

    # 6. Normalizar respuesta
    coverage_results = [
        CoverageResult.from_api_response(cover_data)
        for cover_data in result
    ]

    return QuoteResponse.success(company=company.name, results=coverage_results).to_dict()


def _validate_input(risk_dict: dict) -> tuple[bool, str]:
    """Valida entrada usando Specifications."""
    zip_code = risk_dict.get("property_info", {}).get("zip_code", "")

    zip_spec = ZipCodeValidSpec()
    if not zip_spec.is_satisfied_by(zip_code):
        return False, zip_spec.get_error_message()

    if not risk_dict.get("intermediaries", {}).get("producer_code"):
        return False, "Se requiere codigo de productor"

    return True, ""


def _build_quote_payload(risk_dict: dict, locality: LocalityInfo) -> QuotePayload:
    """Construye payload tipado para API."""
    return QuotePayload(
        cover_codes=risk_dict.get("cover_codes", []),
        additionals=risk_dict.get("aditionals", []),
        locality=LocalityPayload(
            locality_code=locality.locality_code,
            zip_code=str(locality.zip_code),
        ),
        organizer_code=risk_dict.get("intermediaries", {}).get("organizer_code"),
        producer_code=risk_dict.get("intermediaries", {}).get("producer_code"),
        validity_from=risk_dict.get("validity_from", ""),
    )
```

---

## 13. DTOs Base (core/domain/entities.py)

### 13.1 Jerarquia de DTOs

```
DataclassBase
    |-- _dict() -> dict                    # Serializa a diccionario
    |-- from_dict(data: dict) -> Self      # Reconstruye desde diccionario
    |
    └── RiskBase (extends DataclassBase)
            |-- quote_pk, quotation_pk, presale_pk: int | None
            |-- address: Address | None
            |-- intermediaries: Intermediaries | None
            |-- payment: Payment | None
            |-- person_holder: Person | None
            |-- validity_from, validity_to: str | None
            |-- product, cover_code, extra: str | dict | None
            |
            └── QuoteHomeDTO (extends RiskBase)
            └── QuoteVehicleDTO (extends RiskBase)
            └── QuoteARTDTO (extends RiskBase)
```

### 13.2 Value Objects Compartidos

```python
@dataclass
class Address:
    street: str | None = None
    street_number: str | None = None
    floor: str | None = None
    apartment: str | None = None
    zip_code: str | None = None
    province: str | None = None
    locality: str | None = None
    id_locality: int | None = None

@dataclass
class Person:
    document_type: str | None = None
    document_number: str | None = None
    first_name: str | None = None
    last_name: str | None = None
    email: str | None = None
    phone: str | None = None
    birthdate: str | None = None
    sex: str | None = None
    tax_type: str | None = None  # Monotributista, Resp. Inscripto, etc.

@dataclass
class Intermediaries:
    organizer_code: int | None = None
    producer_code: int | None = None
    commission_percentage: float | None = None

@dataclass
class Payment:
    method: str | None = None  # tarjeta, cbu, cupon
    card_number: str | None = None
    card_expiry: str | None = None
    bank_account: str | None = None
```

### 13.3 Serializacion Automatica

```python
# DataclassBase._dict() usa asdict() con manejo de nested dataclasses
def _dict(self) -> dict:
    return asdict(self)

# DataclassBase.from_dict() reconstruye nested dataclasses automaticamente
@classmethod
def from_dict(cls, data: dict) -> Self:
    # Detecta campos que son dataclasses y los reconstruye
    # Ej: {"address": {"street": "Av. Siempreviva"}} -> Address(street="Av. Siempreviva")
    ...
```

---

## 14. Sistema de Coberturas

### 14.1 Modelos del Sistema de Coberturas

```
covers/models.py:

Feature (caracteristica individual)
    |-- name, slug, default_description
    |-- product: ForeignKey(Product)
    |-- fee: DecimalField (opcional)
    |-- unique_together: (slug, product)
    |
    └── Ej: "Robo Total", "Incendio", "Responsabilidad Civil"

Cover (cobertura completa)
    |-- name, slug, description, code
    |-- company, product: ForeignKeys
    |-- static_fee, additional_code
    |-- require_inspection, active
    |-- unique_together: (company, code, product)
    |
    └── Ej: "Cobertura 70 - Terceros Completo", "Cobertura 71 - Todo Riesgo"

CoverFeature (relacion Cover <-> Feature)
    |-- cover: ForeignKey(Cover)
    |-- feature: ForeignKey(Feature)
    |-- private_description, private_quote
    |-- unique_together: (cover, feature)

CoverGroup (agrupacion para display)
    |-- name, slug, description
    |-- product: ForeignKey(Product)
    |-- active, order
    |-- unique_together: (slug, product)
    |
    └── Ej: "Coberturas Basicas", "Coberturas Adicionales"

CoverGroupCover (relacion CoverGroup <-> Cover)
    |-- cover_group: ForeignKey(CoverGroup)
    |-- cover: ForeignKey(Cover)
```

### 14.2 Flujo de Creacion de Coberturas

```
1. API responde con lista de coberturas
   [{"cover_code": 70, "fee": 1500, "coverages": [...]}]
         |
2. QuoteService.store_quote_result() procesa cada cobertura
         |
3. Cover.objects.get_or_create(company, code, product)
   - Si no existe, crea Cover con datos basicos
         |
4. CoverageProcessor.process() crea QuotationHomeCover
   - Extrae datos con extract_cover_data()
   - Crea relacion Quote <-> Cover
         |
5. CoverageProcessor.post_process() crea Features
   - Itera coverage["coverages"] (items individuales)
   - Feature.objects.get_or_create(slug, product)
   - CoverFeature.objects.get_or_create(cover, feature)
```

---

## 15. Dependency Injection Patterns

### 15.1 Constructor Injection en DTOFactory

```python
# En QuoteService.execute_company_tasks()
producer = ProducerCompanyService.get_company_producer(company.slug, user)
locality = ModelCompanyService.get_company_model_instance(company.slug, "locality", zip_code)

# Inyeccion via keyword-only arguments
risk_data = self.quote.build_data_class(
    company,
    producer=producer,   # Inyectado
    locality=locality,   # Inyectado
)

# HomeQuoteDTOFactory recibe dependencias explicitamente
class HomeQuoteDTOFactory(QuoteDTOFactory[QuotationHome, QuoteHomeDTO]):
    def __init__(self, quotation: QuotationHome, *, producer=None, locality=None):
        super().__init__(quotation, producer=producer, locality=locality)
```

### 15.2 Method Injection en CoverageProcessor

```python
# creation_user inyectado via keyword-only
processor.process(
    company_slug=company_slug,
    quote_id=quote_id,
    cover=cover,
    coverage=coverage_data,
    creation_user=user,  # Inyectado
)
```

### 15.3 Beneficios

- Dependencias explicitas en firmas de metodos
- Facil de mockear en tests
- Sin service locators ocultos
- Flujo de datos claro entre capas

---

## 16. Procesamiento Asincrono (Celery)

### 16.1 Arquitectura de Tareas

```
QuoteService.execute_company_tasks()
    |
    |-- cotizar.delay(company_slug, risk_dict=dto._dict())
    |       |
    |       v
    [Celery Worker]
        |-- execute_company_handler("sancor", risk_dict, "cotizar")
        |       |
        |       |-- import companies.sancor.handlers.home
        |       |-- cotizar(risk_dict) -> QuoteResponse
        |       v
        |-- QuoteService.store_quote_result()
                |
                |-- CoverageProcessorRegistry.get("home")
                |-- HomeCoverageProcessor.process()
                v
        Task STATUS = SUCCESS

Frontend Polling (cada 200ms):
    |
    v
GetTaskView -> build_task_response(task_id)
    |-- AsyncResult(task_id).status
    |       |
    |       |-- PENDING -> {"status": "PENDING"} -> retry
    |       |-- SUCCESS -> render HTML -> show prices
    |       |-- FAILURE -> {"status": "FAILURE"} -> show error
```

### 16.2 QuoteCompanyQuote (Tracking de Tareas)

```python
class QuoteCompanyQuote(BaseEntity):
    """Trackea tarea Celery por compania."""
    content_type = ForeignKey(ContentType)  # QuotationHome, QuotationVehicle, etc.
    object_id = PositiveIntegerField()
    quote = GenericForeignKey()

    company = ForeignKey(Company)
    task_id = CharField(max_length=100)  # Celery task ID

    class Meta:
        unique_together = ("content_type", "object_id", "company")
```

### 16.3 Configuracion por Producto

```python
# Productos con precio unico (home, ART)
# -> 1 tarea por compania

# Productos con precio variable por medio de pago (auto, moto)
# -> N tareas por compania (1 por metodo de pago)

def execute_company_tasks(self):
    if self.product.uses_payment_variation:
        # Auto: crear tarea por cada payment method
        for payment_method in ["tarjeta", "cbu", "cupon"]:
            risk_data = self.quote.build_data_class(...)
            risk_data.payment = Payment(method=payment_method)
            cotizar.delay(company_slug, risk_dict=risk_data._dict())
    else:
        # Home/ART: una sola tarea
        risk_data = self.quote.build_data_class(...)
        cotizar.delay(company_slug, risk_dict=risk_data._dict())
```

---

## 17. Decisiones Arquitectonicas

### 17.1 Generic Relations vs Herencia de Tablas

**Decision:** Usar `GenericForeignKey` en lugar de tablas separadas por producto.

**Razon:**
- Evita proliferacion de tablas (N productos x M modelos)
- Permite agregar productos sin migraciones en modelos base
- Mantiene type safety via `limit_choices_to`

**Implementacion:**
```python
class BaseQuotationCover(BaseEntity):
    content_type = ForeignKey(ContentType, limit_choices_to={
        "model__in": ["quotationart", "quotationvehicle", "quotationhome"]
    })
    object_id = PositiveIntegerField()
    quote = GenericForeignKey("content_type", "object_id")
```

### 17.2 Dataclasses vs Django Forms para DTOs

**Decision:** Usar `dataclasses` para DTOs de API, no Django Forms.

**Razon:**
- DTOs son para comunicacion entre capas y APIs externas
- No necesitan widgets ni rendering HTML
- Serializacion automatica con `asdict()`
- Type hints completos para validacion estatica

### 17.3 Dynamic Import para Handlers

**Decision:** Cargar handlers dinamicamente por `{company}/{product}`.

**Razon:**
- Agregar companias sin modificar codigo central
- Cada compania es un modulo independiente
- Facil de testear handlers en aislamiento

**Implementacion:**
```python
def execute_company_handler(company_slug: str, risk_dict: dict, action: str):
    product_slug = risk_dict.get("product", "home")
    module_name = f"companies.{company_slug}.handlers.{product_slug}"
    module = importlib.import_module(module_name)
    func = getattr(module, action)
    return func(risk_dict)
```

### 17.4 Cache con TTL para Datos de API

**Decision:** Cachear datos semi-estaticos de APIs externas (localidades, cover codes).

**Razon:**
- Reduce llamadas a APIs externas (rate limiting, latencia)
- Localidades argentinas cambian raramente
- TTL de 24 horas balancea frescura vs performance

---

## 18. Referencias

- **Plan DDD:** `.claude/plan.md`
- **Plan de Accion Home:** `ACTION_PLAN.md`
- **API Payload Home:** `SANCOR_API_PAYLOAD_APP_HOME.md`
- **API Response Home:** `SANCOR_API_RESPONSE_APP_HOME.md`
- **App Referencia Vehicles:** `branchs/patrimoniales/vehicles/`
- **App Referencia Home:** `branchs/patrimoniales/home/`
- **Cookie Cutter:** `cookie_cutter_insurance_base/`

---

**Ultima actualizacion:** 2025-12-19
**Version:** 3.0 (agregadas secciones 12-17 con implementaciones concretas)
