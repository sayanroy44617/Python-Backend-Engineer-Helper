# Application Architecture

## What

How to structure a FastAPI project as it grows beyond a single `main.py`:
layering (routes / services / repositories), package organization, and
where cross-cutting concerns (config, dependencies, exception handlers)
live.

## Why

FastAPI itself doesn't prescribe a project structure — that flexibility is
useful for small projects but becomes a liability at scale without a
deliberate convention. A clear layering keeps business logic testable
independent of HTTP, keeps route handlers thin, and makes it obvious where
new code belongs as the codebase grows.

## How

### A layered structure

```
app/
├── main.py                 # creates the FastAPI app, wires routers, lifespan
├── api/
│   ├── __init__.py
│   ├── dependencies.py     # shared Depends() functions
│   └── routes/
│       ├── users.py        # APIRouter for /users
│       └── orders.py       # APIRouter for /orders
├── services/
│   ├── user_service.py     # business logic, orchestrates repositories
│   └── order_service.py
├── repositories/
│   ├── user_repository.py  # DB access for User
│   └── order_repository.py
├── models/
│   ├── orm.py               # SQLAlchemy models
│   └── schemas.py           # Pydantic request/response models
├── core/
│   ├── config.py            # Settings (pydantic-settings)
│   └── exceptions.py        # domain exception hierarchy
└── db/
    └── session.py            # engine, session factory, get_db dependency
```

This mirrors the package structure principles from
[Modules and Packages](../python/04-modules-and-packages.md), applied to a
web application: each package has one responsibility, and dependencies flow
one direction (`api` → `services` → `repositories`), never the reverse.

### Layer responsibilities

| Layer | Responsibility | Should NOT contain |
|---|---|---|
| `api/routes` | HTTP concerns: request parsing, status codes, calling services | Business logic, direct DB queries |
| `services` | Business logic, orchestration, transactions | HTTP-specific code (no `Request`/`HTTPException`) |
| `repositories` | Data access (queries, ORM interaction) | Business rules |
| `models/schemas` | Pydantic request/response contracts | Database access logic |
| `models/orm` | SQLAlchemy models (persistence shape) | API-facing serialization concerns |
| `core` | Config, domain exceptions, cross-cutting setup | Route-specific logic |

### A thin route delegating to a service

```python
# api/routes/users.py
router = APIRouter(prefix="/users", tags=["users"])

@router.post("", response_model=UserOut)
def create_user(payload: UserCreate, service: UserService = Depends(get_user_service)):
    return service.create_user(payload)
```

```python
# services/user_service.py
class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    def create_user(self, payload: UserCreate) -> User:
        if self._repo.exists_by_email(payload.email):
            raise UserAlreadyExistsError(payload.email)
        return self._repo.create(payload)
```

The route only knows about HTTP concerns; `UserService` contains business
rules and knows nothing about `Request`/`HTTPException`, which keeps it
directly unit-testable and reusable outside the web layer (e.g. a CLI
script or background job calling the same service).

### Wiring it together with dependencies

```python
# api/dependencies.py
def get_user_repository(db: Session = Depends(get_db)) -> UserRepository:
    return UserRepository(db)

def get_user_service(repo: UserRepository = Depends(get_user_repository)) -> UserService:
    return UserService(repo)
```

This composes the dependency injection pattern from
[Dependency Injection](03-dependency-injection.md) into a full chain:
`get_db` → `get_user_repository` → `get_user_service` → route handler —
each layer only depends on the one directly below it.

### `main.py` as composition root

```python
# main.py
from fastapi import FastAPI
from app.api.routes import users, orders
from app.core.exceptions import register_exception_handlers
from app.db.session import lifespan

app = FastAPI(lifespan=lifespan)

app.include_router(users.router)
app.include_router(orders.router)
register_exception_handlers(app)
```

`main.py` stays small — it composes routers, exception handlers, and
lifespan setup, but doesn't contain business or route logic itself.

## When to use

- Introduce the service/repository split as soon as a route handler starts
  containing more than a few lines of logic beyond parsing input and
  calling something else.
- Keep Pydantic schemas (`models/schemas.py`) separate from ORM models
  (`models/orm.py`) from the start — retrofitting this separation later is
  more painful than starting with it.
- Use `core/config.py` (Pydantic Settings) as the single source of
  environment-driven configuration, injected where needed rather than read
  ad hoc via `os.environ` throughout the codebase.

## When NOT to use

- Don't introduce every layer (service, repository, use-case, DTO, etc.)
  for a genuinely small project — a flatter structure (routes calling a
  thin data-access layer directly) is fine until complexity justifies more
  layers.
- Don't let services depend on FastAPI-specific types (`Request`,
  `HTTPException`) — that couples business logic to the web framework and
  makes it harder to reuse or test in isolation.
- Don't organize purely by technical layer with no domain grouping in a
  large app (e.g. hundreds of files under one flat `services/`) — group by
  domain/bounded context (`services/orders/`, `services/users/`) once the
  project grows large enough that a flat layer becomes unwieldy.

## Common mistakes

- Business logic embedded directly in route handlers, making it
  untestable without spinning up the whole HTTP stack.
- Raising `HTTPException` from deep inside a service/repository layer,
  coupling business logic to HTTP status codes (use domain exceptions +
  handlers instead — see
  [Middleware and Exception Handling](04-middleware-and-exception-handling.md)).
- Circular dependencies between layers (e.g. a repository importing from a
  service) — dependencies should flow one direction only.
- Mixing ORM models and Pydantic schemas into a single class to "save
  boilerplate," coupling the API contract to the database schema so
  tightly that changing one always risks breaking the other.

## Interview questions

1. Why keep API routes, services, and repositories as separate layers
   instead of one flat file?

   **Answer:** Because they change for different reasons: HTTP shape, business rules, and persistence are separate concerns. That separation keeps route handlers thin and lets you change storage or transport without rewriting everything.
2. Why shouldn't a service layer raise `HTTPException` directly?

   **Answer:** `HTTPException` is a web concern, and putting it in services hard-couples business logic to FastAPI. A service should raise a domain error like `UserAlreadyExistsError`, and the API layer should translate that into the right status code.

   ```python
   class UserAlreadyExistsError(Exception):
       pass

   def create_user(email_exists: bool) -> None:
       if email_exists:
           raise UserAlreadyExistsError()
   ```
3. How does this layering make unit testing business logic easier?

   **Answer:** You can test the service with a fake repository directly, without HTTP requests, dependency injection, or a running app. That makes tests faster and much more targeted.

   ```python
   class FakeUserRepository:
       def exists_by_email(self, email: str) -> bool:
           return False
   ```
4. Why separate Pydantic schemas from ORM models instead of using one
   class for both?

   **Answer:** API contracts and database shape drift for different reasons, so forcing them into one class creates unnecessary coupling. Separate models let you change a column, hide an internal field, or version the API without dragging the persistence layer along.
5. How would you compose a chain of dependencies (`get_db` →
   `get_user_repository` → `get_user_service`) using FastAPI's DI system?

   **Answer:** Build one dependency per layer and let each one depend on the next lower layer. That keeps construction centralized and makes the route depend only on the service it actually needs.

   ```python
   from fastapi import Depends

   def get_user_service(repo: UserRepository = Depends(get_user_repository)) -> UserService:
       return UserService(repo)
   ```

## Senior-level considerations

- Layering is a trade-off, not a universal rule — over-layering a small
  service adds indirection without benefit; the right amount of structure
  scales with team size and codebase complexity, and should be revisited
  as the project grows rather than decided once and never questioned; for
  example, a two-endpoint internal tool may not need repositories yet, while a
  larger product API probably will.
- Keeping business logic framework-agnostic (no FastAPI imports inside
  `services/`) is what allows reusing the same logic from a CLI, a
  background worker, or a different web framework entirely if needed — a
  common real-world scenario when a synchronous batch job needs to reuse
  the same business rules as the API; for example, both a `/users` route and a
  nightly CSV import job can call the same `UserService.create_user()`.
- Domain-oriented package boundaries (grouping by bounded context rather
  than by technical layer alone) tend to age better in large codebases —
  this connects directly to the System Design section's discussion of
  service boundaries, even within a single deployable application; for
  example, `services/billing/` and `services/catalog/` usually scale better
  than hundreds of unrelated files in one flat `services/` directory.
