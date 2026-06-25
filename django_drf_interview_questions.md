# Django REST Framework (DRF) — 30 Interview Questions & Answers

A reference covering core concepts, serializers, views, authentication, performance, and best practices.

---

## Core Concepts

### 1. What is Django REST Framework and why use it over plain Django?
DRF is a toolkit built on top of Django for building Web APIs. It adds serialization (model ↔ JSON), browsable API, authentication/permission classes, viewsets, routers, throttling, and content negotiation. Plain Django is geared toward server-rendered HTML; DRF removes the boilerplate of parsing requests, validating data, and rendering responses for API clients.

### 2. What is serialization and deserialization?
**Serialization** converts complex types (querysets, model instances) into native Python datatypes that can be rendered into JSON/XML. **Deserialization** does the reverse: parses incoming data, validates it, and converts it back into complex types (e.g., model instances). The `Serializer.data` property gives serialized output; `is_valid()` + `validated_data` handle deserialization.

### 3. Difference between `Serializer` and `ModelSerializer`?
`Serializer` requires you to declare every field and write `create()`/`update()` manually. `ModelSerializer` auto-generates fields from the model, provides default `create()`/`update()` implementations, and auto-includes validators (e.g., unique constraints). Use `ModelSerializer` for standard CRUD on models; use plain `Serializer` for non-model data or custom shapes.

### 4. What is the `Meta` class in a serializer?
It configures a `ModelSerializer`: `model` (the bound model), `fields` (list or `'__all__'`), `exclude`, `read_only_fields`, and `extra_kwargs` (per-field options like `write_only`, `required`). Prefer explicit `fields` over `'__all__'` to avoid accidentally exposing data.

### 5. Explain `read_only`, `write_only`, and `required` field options.
- `read_only=True`: included in output, ignored on input (e.g., `id`, timestamps).
- `write_only=True`: accepted on input, never serialized to output (e.g., `password`).
- `required=False`: field may be omitted in input (used for PATCH or optional fields).

---

## Views & Routing

### 6. Difference between `APIView`, generic views, and `ViewSet`?
- `APIView`: lowest level, you define `get()`, `post()`, etc. Full control.
- **Generic views** (`ListAPIView`, `RetrieveUpdateDestroyAPIView`): mix in common behavior via `queryset` + `serializer_class`.
- `ViewSet`/`ModelViewSet`: groups related actions (`list`, `retrieve`, `create`, `update`, `destroy`) into one class, wired by routers.

### 7. What is a `ModelViewSet`?
A viewset that provides full CRUD by combining all mixins (`Create`, `Retrieve`, `Update`, `Destroy`, `List`) with `GenericViewSet`. You just set `queryset`, `serializer_class`, and optionally `permission_classes`. The router maps HTTP methods to actions automatically.

### 8. What are routers and what do they do?
Routers (`DefaultRouter`, `SimpleRouter`) auto-generate URL patterns for viewsets. `router.register(r'users', UserViewSet)` creates list/detail routes and maps verbs to actions. `DefaultRouter` also adds an API root view and format suffixes. They reduce manual `urlpatterns` boilerplate.

### 9. How do you add a custom action to a ViewSet?
Use the `@action` decorator:
```python
from rest_framework.decorators import action

class UserViewSet(ModelViewSet):
    @action(detail=True, methods=['post'])
    def set_password(self, request, pk=None):
        ...
```
`detail=True` creates `/users/{pk}/set_password/`; `detail=False` creates `/users/set_password/`.

### 10. What is the difference between `get_queryset()` and `queryset`?
`queryset` is a static attribute evaluated once at class load. `get_queryset()` is called per-request, letting you filter dynamically based on `self.request.user`, query params, etc. Use `get_queryset()` whenever the result depends on the request.

---

## Authentication & Permissions

### 11. What authentication classes does DRF provide?
`SessionAuthentication` (uses Django sessions/cookies), `BasicAuthentication` (HTTP Basic, dev only), `TokenAuthentication` (DB-backed tokens), and `RemoteUserAuthentication`. JWT is provided via third-party packages like `djangorestframework-simplejwt`.

### 12. Difference between authentication and permissions?
**Authentication** identifies *who* the requester is (sets `request.user` / `request.auth`). **Permissions** decide *whether* that identified user may perform the action. Authentication runs first; permissions run after, on every request.

### 13. How does Token Authentication work?
The client obtains a token (via an endpoint), then sends `Authorization: Token <key>` on each request. DRF looks up the token, attaches the associated user to `request.user`. Tokens don't expire by default with the built-in `TokenAuthentication` — use JWT/SimpleJWT for expiry and refresh.

### 14. What is JWT and how is it used in DRF?
JWT (JSON Web Token) is a stateless, signed token containing claims. With `simplejwt`, the client gets an `access` + `refresh` token pair, sends `Authorization: Bearer <access>`. The server verifies the signature without a DB lookup. Access tokens are short-lived; refresh tokens get new access tokens.

### 15. How do you write a custom permission class?
```python
from rest_framework.permissions import BasePermission

class IsOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user
```
`has_permission` runs for the view; `has_object_permission` runs per-object (e.g., in `RetrieveUpdateDestroy`). Return a boolean.

---

## Validation

### 16. What are the levels of validation in DRF?
- **Field-level**: `validate_<fieldname>(self, value)`.
- **Object-level**: `validate(self, data)` for cross-field checks.
- **Validators**: reusable functions/classes attached to fields (e.g., `UniqueValidator`, `UniqueTogetherValidator`).
Validation runs when `serializer.is_valid()` is called.

### 17. How does `is_valid()` work and what is `raise_exception`?
`is_valid()` runs all validation and populates `validated_data` (on success) or `errors` (on failure), returning a boolean. `is_valid(raise_exception=True)` raises `ValidationError`, returning a `400` response automatically — common in views to avoid manual error handling.

### 18. What is the difference between `validated_data` and `initial_data`?
`initial_data` is the raw input passed to the serializer before validation. `validated_data` is the cleaned, type-coerced data available only *after* `is_valid()` succeeds. Always use `validated_data` in `create()`/`update()`.

---

## Relationships & Nested Data

### 19. How do you represent relationships in serializers?
Options include `PrimaryKeyRelatedField` (IDs), `StringRelatedField` (`__str__`), `SlugRelatedField` (a unique field), `HyperlinkedRelatedField` (URLs), and **nested serializers** (full object embedding). Choice depends on payload size and client needs.

### 20. How do you handle writable nested serializers?
By default nested serializers are read-only. To write, override `create()`/`update()` to pop nested data and create related objects manually:
```python
def create(self, validated_data):
    items = validated_data.pop('items')
    order = Order.objects.create(**validated_data)
    for item in items:
        OrderItem.objects.create(order=order, **item)
    return order
```

### 21. What is `source` in a serializer field?
`source` maps a serializer field to a differently named model attribute or method. E.g., `full_name = serializers.CharField(source='get_full_name')` or `source='profile.bio'` for nested attributes. `source='*'` passes the whole object.

### 22. What is a `SerializerMethodField`?
A read-only field whose value comes from a method `get_<fieldname>(self, obj)` on the serializer. Useful for computed/derived values:
```python
is_new = serializers.SerializerMethodField()
def get_is_new(self, obj):
    return obj.created > some_date
```

---

## Performance

### 23. How do you avoid the N+1 query problem in DRF?
Use `select_related` (forward FK/one-to-one, SQL JOIN) and `prefetch_related` (reverse FK/many-to-many, separate query) in `get_queryset()`. For nested serializers especially, prefetching related objects prevents one extra query per row.

### 24. How does pagination work in DRF?
Set `DEFAULT_PAGINATION_CLASS` and `PAGE_SIZE` globally, or `pagination_class` per view. Styles: `PageNumberPagination` (`?page=2`), `LimitOffsetPagination` (`?limit=10&offset=20`), and `CursorPagination` (opaque cursor, best for large/real-time datasets and consistent ordering).

### 25. How do you implement filtering, searching, and ordering?
Use `django-filter` with `DjangoFilterBackend` (`filterset_fields`), `SearchFilter` (`search_fields`, `?search=`), and `OrderingFilter` (`ordering_fields`, `?ordering=-created`). Set them in `filter_backends` on the view.

### 26. How does throttling work?
Throttling limits request rates. Configure `DEFAULT_THROTTLE_CLASSES` and `DEFAULT_THROTTLE_RATES` (e.g., `'anon': '100/day'`). `AnonRateThrottle` limits unauthenticated users by IP; `UserRateThrottle` limits authenticated users by ID; `ScopedRateThrottle` applies per-view scopes.

---

## Advanced & Practical

### 27. What is the request/response lifecycle in DRF?
A request hits the view → DRF wraps it in `Request` → runs **authentication** → **permission** checks → **throttling** → parses body (parsers) → dispatches to the handler (`get`/`post`/action) → serializer validates/serializes → returns a `Response` → content negotiation picks a renderer → final HTTP response. Exceptions are handled by the exception handler.

### 28. How do you customize error handling globally?
Override the exception handler:
```python
# settings.py
REST_FRAMEWORK = {'EXCEPTION_HANDLER': 'myapp.utils.custom_handler'}
```
The handler receives `(exc, context)`, can call DRF's default handler, then reshape the response (e.g., consistent `{"error": ...}` envelope) or handle non-DRF exceptions.

### 29. What are parsers and renderers?
**Parsers** deserialize the incoming request body based on `Content-Type` (`JSONParser`, `FormParser`, `MultiPartParser` for file uploads). **Renderers** serialize the outgoing response based on `Accept` header (`JSONRenderer`, `BrowsableAPIRenderer`). Content negotiation selects the appropriate one.

### 30. How do you version a DRF API?
Set `DEFAULT_VERSIONING_CLASS`. Strategies: `URLPathVersioning` (`/v1/users/`), `NamespaceVersioning`, `QueryParameterVersioning` (`?version=1`), `AcceptHeaderVersioning` (`Accept: application/json; version=1.0`), and `HostNameVersioning`. `request.version` is then available in views/serializers to alter behavior.

---

## Quick Reference Cheat Sheet

| Topic | Key Class/Setting |
|---|---|
| Full CRUD | `ModelViewSet` |
| Auto URLs | `DefaultRouter` |
| Model → JSON | `ModelSerializer` |
| Per-request queryset | `get_queryset()` |
| Stateless auth | `simplejwt` (`Bearer`) |
| Object-level access | `has_object_permission` |
| Cross-field validation | `validate(self, data)` |
| Avoid N+1 | `select_related` / `prefetch_related` |
| Rate limiting | `DEFAULT_THROTTLE_RATES` |
| Large datasets | `CursorPagination` |
