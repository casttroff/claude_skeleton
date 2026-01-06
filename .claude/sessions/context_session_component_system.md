# Context Session: Component System

## Feature Overview

Implement the core component system for the page builder, allowing users to create reusable, validated components (navbar, hero, card, etc.) with Pydantic schemas and DRF serializers.

This is the foundation of the entire page builder - all other features depend on this system working correctly.

**Reference**: PROJECT_SPECS.md - ETAPA 1

## Status and Timeline

- Start Date: TBD
- Target: Day 7 of 30
- Status: Not Started
- Current Day: Day 0

## Plan

### Phase 1: Core Architecture (Days 1-3)
**Goal**: Establish component validation and registration system

- [ ] Create `ComponentBase` Pydantic schema with type field
- [ ] Create `COMPONENT_REGISTRY` dict mapping types to schemas
- [ ] Create `ComponentSerializerBase` DRF serializer
- [ ] Create `validate_component()` utility function
- [ ] Write comprehensive tests for validation system

**Tests**:
- Valid component passes validation
- Invalid component raises ValidationError
- Unknown component type rejected
- Registry properly maps types to schemas

### Phase 2: Initial Components (Days 4-5)
**Goal**: Implement 3 foundational components

- [ ] Navbar component (NavbarConfig + template)
- [ ] Hero component (HeroConfig + template)
- [ ] Card component (CardConfig + template)

**Tests for each**:
- Schema validation (required fields, optional fields)
- Serializer validation
- Template rendering
- Edge cases (missing fields, invalid values)

### Phase 3: Integration (Days 6-7)
**Goal**: Connect components to Page model

- [ ] Page model with JSONField for components list
- [ ] Custom validation in Page.clean()
- [ ] API endpoint for component preview
- [ ] Tests for multi-component pages

**Tests**:
- Page with single component
- Page with multiple components
- Page with invalid component rejected
- Component order preserved

## User Decisions

### Decision 1: Validation Strategy
**Question**: Should we validate components on save (Pydantic) or on API request (DRF serializer)?
**Answer**: BOTH - Pydantic in model clean(), DRF in API serializer
**Rationale**: Defense in depth - catch errors in multiple layers

### Decision 2: Component Registry
**Question**: Should component types be hardcoded or dynamic?
**Answer**: Hardcoded in COMPONENT_REGISTRY for initial version
**Rationale**: Simpler, more secure, easier to test. Can make dynamic later if needed.

### Decision 3: Template Location
**Question**: Should component templates be in apps/pages/templates/ or root templates/?
**Answer**: Root templates/components/ for reusability
**Rationale**: Components may be used across multiple apps in future

## Technical Challenges

### Challenge 1: Pydantic + DRF Integration
**Problem**: Pydantic and DRF have different validation patterns
**Solution**: Create `ComponentSerializerBase` that calls Pydantic schema in validate() method
**Status**: Resolved

**Implementation**:
```python
class ComponentSerializerBase(serializers.Serializer):
    def validate(self, data):
        # Get schema from registry
        component_type = data.get('type')
        schema = COMPONENT_REGISTRY[component_type]

        # Validate with Pydantic
        try:
            schema(**data)
        except ValidationError as e:
            raise serializers.ValidationError(str(e))

        return data
```

### Challenge 2: JSONField Validation
**Problem**: Django JSONField doesn't validate structure automatically
**Solution**: Override Page.clean() to validate each component
**Status**: Resolved

**Implementation**:
```python
class Page(models.Model):
    components = models.JSONField(default=list)

    def clean(self):
        for component_data in self.components:
            validate_component(component_data)
```

## Key Learnings

- Pydantic validation is extremely fast (thousands per second)
- DRF serializers provide better error messages for API
- Using both provides defense in depth
- JSONField is powerful but needs custom validation

## Architecture Decisions

### Decision: Component Storage Format
**Options Considered**:
1. **JSONField with list of dicts** (chosen)
   - Simple, flexible
   - Easy to query with JSON operators
   - No extra tables

2. **Separate Component model with ForeignKey**
   - More normalized
   - More complex queries
   - Harder to reorder

3. **PostgreSQL JSONB with custom operators**
   - Maximum performance
   - More complex
   - Harder to migrate

**Chosen**: Option 1 (JSONField with list)
**Reason**: Simplest implementation, good enough performance for 10k components/page, easy to understand and test

### Decision: Component Type System
**Options Considered**:
1. **String literal types** ('navbar', 'hero', 'card')
2. **Enum with component types**
3. **Integer foreign key to ComponentType table**

**Chosen**: Option 1 (String literals)
**Reason**: Matches JSON naturally, easy to read, Pydantic Literal type provides validation

## Code Patterns Used

### Pattern 1: Registry Pattern
```python
COMPONENT_REGISTRY: Dict[str, Type[ComponentBase]] = {
    'navbar': NavbarConfig,
    'hero': HeroConfig,
    'card': CardConfig,
}

def validate_component(data: Dict[str, Any]) -> ComponentBase:
    component_type = data.get('type')
    if component_type not in COMPONENT_REGISTRY:
        raise ValueError(f"Invalid component type: {component_type}")
    schema = COMPONENT_REGISTRY[component_type]
    return schema(**data)
```

**Used in**: `src/core/schemas.py`, `apps/pages/serializers.py`
**Reason**: Easy to extend with new components, type-safe, testable

### Pattern 2: Base Schema Inheritance
```python
class ComponentBase(BaseModel):
    type: str
    # Common fields for all components

class NavbarConfig(ComponentBase):
    type: Literal['navbar'] = 'navbar'
    # Navbar-specific fields
```

**Used in**: All component schemas
**Reason**: DRY principle, ensures all components have required base fields

## Research Documentation

To be created by agents:
- `.claude/doc/component_system/frontend.md` (DaisyUI component patterns)
- `.claude/doc/component_system/security.md` (XSS prevention in templates)

## Dependencies

- **Depends on**: None (this is foundational)
- **Blocks**:
  - Page Builder Canvas (needs components to display)
  - Component Library (needs component system working)
  - Page Preview (needs components to render)

## Next Steps

**Immediate (Next Session)**:
1. Create `src/core/schemas.py` with ComponentBase
2. Create COMPONENT_REGISTRY
3. Write tests for validate_component()
4. Run tests (should FAIL - RED)

**Short-term (This Week)**:
- Implement ComponentBase and registry
- Implement 3 initial components (navbar, hero, card)
- Integrate with Page model
- All tests passing (GREEN)

**Long-term (This ETAPA)**:
- 10+ components implemented
- API endpoints for component CRUD
- Component preview working
- Tests at 70% coverage

## Progress Metrics

- Tests Written: 0
- Tests Passing: 0
- Coverage: 0%
- Security Audit: Pending
- Accessibility Audit: N/A (no UI yet)

## Notes

**CRITICAL**: This component system is the foundation of the entire project. Take time to get it right. All future features depend on this working correctly.

**Security Considerations**:
- All component rendering must use Django template escaping
- No use of `|safe` unless content sanitized with bleach
- Validate all component data with Pydantic before saving

**Performance Considerations**:
- Pydantic validation is fast enough for real-time use
- May need caching for pages with 100+ components
- Monitor query count when loading pages with many components

---

**Last Updated**: Not started yet
**Updated By**: System generated template
