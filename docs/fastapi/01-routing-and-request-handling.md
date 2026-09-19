# Routing and Request Handling

## What

FastAPI routes map HTTP methods and paths to handler functions. Request
data arrives as **path parameters**, **query parameters**, **headers**, and
**request bodies** — FastAPI extracts and validates each based on the
function signature's type hints (see
[Type Hints](../python/05-type-hints.md)).

## Why

Correctly declaring where data comes from (path vs query vs body) and how
it's validated is the foundation of every endpoint. Getting this wrong
produces confusing 422 errors for API clients or, worse, silently accepts
malformed input that later breaks business logic.

## How

### Basic routing

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/{user_id}")
def get_user(user_id: int):
    return {"id": user_id}

@app.post("/users")
def create_user(name: str):
    return {"name": name}
```

Path parameters (`{user_id}`) are matched by name against function
arguments. FastAPI infers `user_id` is a path parameter because it appears
in the path template; type hints (`int`) drive automatic parsing and
validation — a non-numeric `user_id` returns a `422` before your function
even runs.

### Query parameters

```python
@app.get("/items")
def list_items(skip: int = 0, limit: int = 20, q: str | None = None):
    return {"skip": skip, "limit": limit, "q": q}
```

Any function parameter that isn't part of the path and isn't a Pydantic
model is treated as a query parameter. Default values make them optional;
`str | None = None` makes `q` an optional query parameter.

### Explicit parameter declarations

```python
from fastapi import Query, Path

@app.get("/items")
def list_items(
    limit: int = Query(default=20, ge=1, le=100, description="Max items to return"),
):
    ...

@app.get("/users/{user_id}")
def get_user(user_id: int = Path(gt=0)):
    ...
```

`Query`/`Path` let you attach validation constraints (`ge`, `le`, `gt`,
regex patterns) and OpenAPI metadata (`description`, `title`) beyond what a
bare type hint conveys.

### Request bodies

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str

@app.post("/users")
def create_user(payload: UserCreate):
    return {"name": payload.name, "email": payload.email}
```

A Pydantic model as a parameter tells FastAPI to parse and validate the
request body as JSON matching that schema — see
[Pydantic and Validation](02-pydantic-and-validation.md) for model design
in depth.

### Combining path, query, and body

```python
@app.put("/users/{user_id}")
def update_user(user_id: int, payload: UserCreate, notify: bool = False):
    ...
```

FastAPI determines each parameter's source by its type: path parameters
from the path template, Pydantic models from the body, everything else
from the query string (unless explicitly marked otherwise).

### Path operation configuration

```python
@app.get(
    "/users/{user_id}",
    response_model=dict,
    status_code=200,
    tags=["users"],
    summary="Get a user by ID",
)
def get_user(user_id: int):
    ...
```

`tags`, `summary`, and similar metadata feed directly into the generated
OpenAPI docs (see
[OpenAPI and Testing](07-openapi-and-testing.md)).

### Routers for larger applications

```python
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/{user_id}")
def get_user(user_id: int):
    ...

# main.py
app.include_router(router)
```

`APIRouter` groups related routes into a separate module, mounted onto the
main `app` — the standard way to avoid one giant file as an application
grows (see
[Application Architecture](08-application-architecture.md)).

## When to use

- Path parameters for identifying a specific resource (`/users/{id}`).
- Query parameters for filtering, pagination, and optional modifiers.
- Request bodies (Pydantic models) for structured data on `POST`/`PUT`/
  `PATCH`.
- `APIRouter` as soon as a project has more than a handful of related
  endpoints — don't wait until the single-file `main.py` becomes
  unmanageable.

## When NOT to use

- Don't put large or structured data in query parameters — use a request
  body instead; query strings have practical length/encoding limits and
  poor ergonomics for nested data.
- Don't accept a raw `dict`/untyped body when a Pydantic model would give
  you validation and documentation for free — see
  [Pydantic and Validation](02-pydantic-and-validation.md).
- Don't put everything in one flat `main.py` with dozens of routes — split
  into routers per resource/domain area.

## Common mistakes

- Expecting a value to be a path parameter when it doesn't appear in the
  path template — it silently becomes a query parameter instead.
- Forgetting default values on optional query parameters, making them
  required and breaking existing clients.
- Mixing up when to use `Query()`/`Path()` (for validation/metadata) versus
  a plain default value (for simple optionality) — both work, but explicit
  declarations are needed for constraints beyond a default.
- Returning a raw ORM object directly instead of a `response_model` (or
  Pydantic model) — mixing internal representations with the public API
  contract.

## Interview questions

- How does FastAPI decide whether a parameter is a path parameter, query
    parameter, or body?

    **Answer:** If the name appears in the path template, it's a path
    parameter. If the value is a Pydantic model, FastAPI reads it from the
    body; otherwise it treats it as a query parameter unless you mark it
    explicitly.

    ```python
    from fastapi import FastAPI
    from pydantic import BaseModel

    app = FastAPI()
    class UserCreate(BaseModel): name: str

    @app.put("/users/{user_id}")
    def update_user(user_id: int, payload: UserCreate, notify: bool = False) -> dict[str, object]:
       return {"user_id": user_id, "notify": notify, "name": payload.name}
    ```

- What's the difference between using a bare type hint with a default
    value versus `Query(...)` for a query parameter?

    **Answer:** A plain default like `limit: int = 20` is enough for basic
    optionality. `Query(...)` is what you use when you also want validation
    rules or OpenAPI metadata such as bounds or descriptions.

    ```python
    from fastapi import FastAPI, Query

    app = FastAPI()

    @app.get("/items")
    def list_items(limit: int = Query(default=20, ge=1, le=100)) -> dict[str, int]:
       return {"limit": limit}
    ```

- Why would you use an `APIRouter` instead of registering all routes
    directly on `app`?

    **Answer:** `APIRouter` keeps related endpoints together so the app stays
    modular as it grows. It also makes shared prefixes, tags, and dependency
    wiring much easier to manage.

    ```python
    from fastapi import APIRouter
    
    router = APIRouter(prefix="/users", tags=["users"])
    ```

- What happens if a client sends a non-numeric value for a path parameter
    typed as `int`?

    **Answer:** FastAPI rejects it during request validation and returns a
    `422` response before your handler runs. Your business logic never sees
    the bad value.

    ```python
    from fastapi import FastAPI

    app = FastAPI()

    @app.get("/users/{user_id}")
    def get_user(user_id: int) -> dict[str, int]:
       return {"id": user_id}
    ```

- How would you add pagination (`skip`/`limit`) to a list endpoint?

    **Answer:** Put `skip` and `limit` on the route as query parameters with
    sensible defaults and bounds. Keep the names consistent across endpoints
    so clients don't have to relearn pagination every time.

    ```python
    from fastapi import FastAPI, Query

    app = FastAPI()

    @app.get("/items")
    def list_items(skip: int = 0, limit: int = Query(default=20, le=100)) -> dict[str, int]:
       return {"skip": skip, "limit": limit}
    ```

## Senior-level considerations

- Consistent parameter conventions (e.g. always `skip`/`limit` for
  pagination, always `sort`/`filter` naming) across a large API surface
  reduce cognitive load for API consumers — worth codifying as a team
  convention, not deciding ad hoc per endpoint. For example, don't use
  `page_size` on one route and `limit` on another for the same API.
- Router organization (by resource, by bounded context/domain) is an
  architectural decision that scales or doesn't as the number of endpoints
  grows into the hundreds — plan router/module boundaries early (see
  [Application Architecture](08-application-architecture.md)). For
  example, keep `/users/*` handlers in one router and billing routes in a
  separate billing module instead of mixing them together.
- Avoid leaking internal/ORM types through route signatures — decoupling
  request/response schemas from database models keeps the API contract
  stable even as internal models evolve. For example, return a
  `UserResponse` schema instead of exposing an ORM model that also contains
  internal flags or database-only fields.
