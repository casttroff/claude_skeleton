# New Component Command

Scaffold a new DaisyUI component for the page builder.

---

## Workflow

You will create a complete component following TDD principles.

### Step 1: Component Information
Ask the user for:
1. **Component name** (e.g., "hero", "cta", "testimonial")
2. **Component purpose** (what it displays)
3. **Required fields** (e.g., title, subtitle, button_text)
4. **Optional fields** (e.g., background_color, alignment)

### Step 2: Research Phase (Use frontend-integrator)
```bash
> Use frontend-integrator to research DaisyUI patterns for [component_name]
```

Agent should:
- Research DaisyUI components that match requirements
- Document recommended structure
- Create `.claude/doc/component_{name}/frontend.md`

### Step 3: Create Pydantic Schema
**File**: `src/core/schemas.py`

```python
class {ComponentName}Config(ComponentBase):
    """
    ABOUTME: Pydantic schema for {component_name} component with validation
    ABOUTME: rules for required fields and optional customization
    """
    type: Literal['{component_name}'] = '{component_name}'

    # Required fields
    title: str = Field(..., min_length=1, max_length=200)
    subtitle: Optional[str] = Field(None, max_length=500)

    # Optional fields
    button_text: Optional[str] = Field(None, max_length=50)
    button_url: Optional[str] = Field(None, max_length=500)
    background_color: Optional[str] = Field('base-100', pattern=r'^[a-z0-9-]+$')

    @model_validator(mode='after')
    def validate_button(self):
        if self.button_text and not self.button_url:
            raise ValueError("button_url required when button_text provided")
        return self
```

### Step 4: Create Serializer
**File**: `apps/pages/serializers.py`

```python
class {ComponentName}Serializer(ComponentSerializerBase):
    """
    ABOUTME: DRF serializer for {component_name} component with field validation
    ABOUTME: and Pydantic schema integration for API endpoints
    """
    type = serializers.CharField(default='{component_name}')
    title = serializers.CharField(max_length=200)
    subtitle = serializers.CharField(max_length=500, required=False, allow_blank=True)
    # ... other fields

    def validate(self, data):
        try:
            {ComponentName}Config(**data)
        except ValidationError as e:
            raise serializers.ValidationError(str(e))
        return data
```

### Step 5: Create Template
**File**: `templates/components/{component_name}.html`

```html
<!-- ABOUTME: DaisyUI {component_name} component template with responsive design -->
<!-- ABOUTME: and customizable background, text colors, and button styles -->

<div class="hero min-h-screen bg-{{ component.background_color|default:'base-100' }}">
  <div class="hero-content text-center">
    <div class="max-w-md">
      <h1 class="text-5xl font-bold">{{ component.title }}</h1>
      {% if component.subtitle %}
        <p class="py-6">{{ component.subtitle }}</p>
      {% endif %}
      {% if component.button_text %}
        <a href="{{ component.button_url }}" class="btn btn-primary">
          {{ component.button_text }}
        </a>
      {% endif %}
    </div>
  </div>
</div>
```

### Step 6: Register in Component Registry
**File**: `src/core/schemas.py` (COMPONENT_REGISTRY)

```python
COMPONENT_REGISTRY: Dict[str, Type[ComponentBase]] = {
    'navbar': NavbarConfig,
    'hero': HeroConfig,
    '{component_name}': {ComponentName}Config,  # ← Add this line
    # ... other components
}
```

### Step 7: Create Tests (RED Phase)
**File**: `tests/test_component_{component_name}.py`

```python
# ABOUTME: Tests for {component_name} component validation, serialization
# ABOUTME: and rendering with various field combinations and edge cases

import pytest
from pydantic import ValidationError
from src.core.schemas import {ComponentName}Config
from apps.pages.serializers import {ComponentName}Serializer

class Test{ComponentName}Schema:
    """Test Pydantic schema validation"""

    def test_valid_component(self):
        data = {
            'type': '{component_name}',
            'title': 'Test Title',
            'subtitle': 'Test Subtitle',
        }
        component = {ComponentName}Config(**data)
        assert component.title == 'Test Title'

    def test_missing_title_raises_error(self):
        data = {'type': '{component_name}'}
        with pytest.raises(ValidationError):
            {ComponentName}Config(**data)

    def test_button_without_url_raises_error(self):
        data = {
            'type': '{component_name}',
            'title': 'Test',
            'button_text': 'Click me',
            # Missing button_url
        }
        with pytest.raises(ValidationError):
            {ComponentName}Config(**data)

class Test{ComponentName}Serializer:
    """Test DRF serializer"""

    def test_valid_serialization(self):
        data = {
            'type': '{component_name}',
            'title': 'Test Title',
        }
        serializer = {ComponentName}Serializer(data=data)
        assert serializer.is_valid()

    def test_invalid_background_color(self):
        data = {
            'type': '{component_name}',
            'title': 'Test',
            'background_color': 'invalid color!',
        }
        serializer = {ComponentName}Serializer(data=data)
        assert not serializer.is_valid()

class Test{ComponentName}Rendering:
    """Test template rendering"""

    def test_renders_title(self, db, client):
        # Create page with component
        page = Page.objects.create(
            site_id=1,
            path='/',
            components=[{
                'type': '{component_name}',
                'title': 'Test Title',
            }]
        )

        response = client.get('/')
        assert 'Test Title' in response.content.decode()

    def test_renders_with_button(self, db, client):
        page = Page.objects.create(
            site_id=1,
            path='/',
            components=[{
                'type': '{component_name}',
                'title': 'Test',
                'button_text': 'Click',
                'button_url': '/action',
            }]
        )

        response = client.get('/')
        assert 'Click' in response.content.decode()
        assert '/action' in response.content.decode()
```

### Step 8: Run Tests (Should Fail - RED)
```bash
python manage.py test tests.test_component_{component_name} -v 2
```

Expected output: **FAILED** (because implementation doesn't exist yet)

### Step 9: Verify Files Created
```bash
ls -la src/core/schemas.py
ls -la apps/pages/serializers.py
ls -la templates/components/{component_name}.html
ls -la tests/test_component_{component_name}.py
```

### Step 10: Summary
Print summary:
```markdown
## Component Scaffold Complete

**Component**: {component_name}

**Files Created**:
- ✅ Schema: `src/core/schemas.py` (updated)
- ✅ Serializer: `apps/pages/serializers.py` (updated)
- ✅ Template: `templates/components/{component_name}.html`
- ✅ Tests: `tests/test_component_{component_name}.py`

**Tests Status**: ❌ FAILING (RED phase)

**Next Steps**:
1. Review component structure
2. Run: `python manage.py test tests.test_component_{component_name}`
3. Use django-implementer to implement missing logic
4. Tests should pass (GREEN phase)

**Research Documentation**:
- See `.claude/doc/component_{component_name}/frontend.md`
```

---

## Usage Examples

```bash
# User invokes command
> /component-new

# Claude asks questions
> What component would you like to create? (e.g., hero, cta, testimonial)

# User responds
> hero

# Claude asks for details
> What fields should the hero component have?

# User provides fields
> title (required), subtitle (optional), button_text (optional), button_url (optional)

# Claude executes workflow
# ... creates all files ...

# Output
Component scaffold complete. Tests are FAILING (RED).
Ready for implementation with django-implementer.
```

---

## Verification Checklist

After running this command, verify:
- [ ] Schema exists in `src/core/schemas.py`
- [ ] Schema has type hints and Field validators
- [ ] Serializer exists in `apps/pages/serializers.py`
- [ ] Template exists in `templates/components/{name}.html`
- [ ] Template uses DaisyUI classes
- [ ] Tests exist in `tests/test_component_{name}.py`
- [ ] Tests cover: valid case, missing required field, invalid field
- [ ] Tests are FAILING (RED phase)
- [ ] Component registered in COMPONENT_REGISTRY

---

## Benefits

- **Standardized structure**: All components follow same pattern
- **TDD enforcement**: Tests created before implementation
- **Complete scaffold**: Nothing missed
- **Fast iteration**: Create new components in minutes
- **Documentation**: Research phase creates reusable docs
