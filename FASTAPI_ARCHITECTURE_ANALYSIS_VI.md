# Phân Tích Kiến Trúc FastAPI - Hướng Dẫn Chi Tiết

**Mục đích**: Tài liệu này phân tích sâu về kiến trúc và cách xây dựng FastAPI để giúp bạn hiểu rõ Python nâng cao và có thể tự xây dựng framework của riêng mình.

---

## 📚 Mục Lục

1. [Tổng Quan Kiến Trúc](#1-tổng-quan-kiến-trúc)
2. [Core Components - Thành Phần Cốt Lõi](#2-core-components---thành-phần-cốt-lõi)
3. [Dependency Injection System](#3-dependency-injection-system)
4. [Request Processing Pipeline](#4-request-processing-pipeline)
5. [Type System & Validation](#5-type-system--validation)
6. [Async/Await Implementation](#6-asyncawait-implementation)
7. [OpenAPI Auto-Documentation](#7-openapi-auto-documentation)
8. [Design Patterns Được Sử Dụng](#8-design-patterns-được-sử-dụng)
9. [Bài Học Quan Trọng](#9-bài-học-quan-trọng)
10. [Hướng Dẫn Xây Dựng Framework Riêng](#10-hướng-dẫn-xây-dựng-framework-riêng)

---

## 1. Tổng Quan Kiến Trúc

### 1.1. Kiến Trúc Phân Tầng (Layered Architecture)

```
┌─────────────────────────────────────────────┐
│   Application Layer (FastAPI class)        │  ← API cho người dùng
├─────────────────────────────────────────────┤
│   Routing Layer (APIRouter, APIRoute)      │  ← Quản lý routes
├─────────────────────────────────────────────┤
│   Dependency Layer (Dependant, solve_deps) │  ← Dependency Injection
├─────────────────────────────────────────────┤
│   Request Handler Layer                     │  ← Xử lý request/response
├─────────────────────────────────────────────┤
│   Validation Layer (Pydantic)              │  ← Type validation
├─────────────────────────────────────────────┤
│   ASGI Layer (Starlette)                   │  ← HTTP protocol
└─────────────────────────────────────────────┘
```

### 1.2. Các Thành Phần Chính

FastAPI được xây dựng trên 3 foundation libraries:

1. **Starlette**: ASGI framework, xử lý HTTP/WebSocket
2. **Pydantic**: Data validation và serialization
3. **Python Type Hints**: Type system và metadata

---

## 2. Core Components - Thành Phần Cốt Lõi

### 2.1. Class FastAPI (applications.py)

**File**: `fastapi/applications.py` (4,588 lines)

```python
class FastAPI(Starlette):
    """Main application class"""

    def __init__(
        self,
        title: str = "FastAPI",
        version: str = "0.1.0",
        openapi_url: Optional[str] = "/openapi.json",
        docs_url: Optional[str] = "/docs",
        ...
    ):
        # 1. Khởi tạo Starlette parent class
        super().__init__(...)

        # 2. Tạo internal router
        self.router = APIRouter(...)

        # 3. Setup OpenAPI schema
        self.openapi_version = "3.1.0"
        self.openapi_schema = None  # Lazy loading

        # 4. Register exception handlers
        self.add_exception_handler(HTTPException, http_exception_handler)
        self.add_exception_handler(RequestValidationError, ...)

        # 5. Setup docs endpoints
        if docs_url:
            self.add_route(docs_url, self.swagger_ui_html, ...)
```

**Điểm quan trọng**:
- Kế thừa từ `Starlette` để có ASGI support
- Sử dụng **composition pattern**: chứa `APIRouter` thay vì kế thừa
- **Lazy loading** cho OpenAPI schema (chỉ generate khi cần)
- **Decorator methods** delegate tới internal router

### 2.2. Class APIRouter (routing.py)

**File**: `fastapi/routing.py` (4,440 lines)

```python
class APIRouter(routing.Router):
    """Router quản lý các path operations"""

    def __init__(
        self,
        prefix: str = "",
        tags: Optional[List[str]] = None,
        dependencies: Optional[Sequence[Depends]] = None,
        ...
    ):
        self.prefix = prefix
        self.tags = tags or []
        self.dependencies = dependencies or []
        self.routes: List[BaseRoute] = []

    def add_api_route(
        self,
        path: str,
        endpoint: Callable,
        *,
        response_model: Any = None,
        status_code: Optional[int] = None,
        tags: Optional[List[str]] = None,
        dependencies: Optional[Sequence[Depends]] = None,
        ...
    ) -> None:
        # 1. Merge router-level và route-level dependencies
        route_dependencies = (self.dependencies or []) + (dependencies or [])

        # 2. Tạo APIRoute object
        route = APIRoute(
            path,
            endpoint=endpoint,
            response_model=response_model,
            status_code=status_code,
            tags=(self.tags or []) + (tags or []),
            dependencies=route_dependencies,
            ...
        )

        # 3. Add vào routes list
        self.routes.append(route)

    # Decorator factory pattern
    def get(self, path: str, **kwargs):
        def decorator(func):
            self.add_api_route(path, func, methods=["GET"], **kwargs)
            return func
        return decorator

    # Tương tự cho post, put, delete, patch...
```

**Kỹ thuật quan trọng**:

1. **Decorator Factory Pattern**:
```python
@app.get("/items/")  # app.get() returns decorator
def read_items():    # decorator wraps this function
    pass

# Internally:
def get(path):
    def decorator(func):
        add_api_route(path, func, methods=["GET"])
        return func
    return decorator
```

2. **Router Composition**:
```python
# Sub-router
items_router = APIRouter(prefix="/items", tags=["items"])

@items_router.get("/")
def get_items(): pass

# Include vào main app
app.include_router(items_router)
# Result: /items/ endpoint với tag "items"
```

### 2.3. Class APIRoute (routing.py)

```python
class APIRoute(routing.Route):
    """Đại diện cho 1 endpoint"""

    def __init__(
        self,
        path: str,
        endpoint: Callable,
        *,
        response_model: Any = None,
        status_code: Optional[int] = None,
        dependencies: Optional[Sequence[Depends]] = None,
        ...
    ):
        # 1. Phân tích endpoint function signature
        self.dependant = get_dependant(path=path, call=endpoint)

        # 2. Merge dependencies
        for depends in dependencies or []:
            self.dependant.dependencies.append(
                get_parameterless_sub_dependant(depends=depends, path=path)
            )

        # 3. Xử lý response model
        if response_model:
            self.response_field = create_model_field(
                name="Response",
                type_=response_model,
            )

        # 4. Tạo actual ASGI handler
        self.app = get_request_handler(
            dependant=self.dependant,
            body_field=self.body_field,
            status_code=status_code,
            response_field=self.response_field,
            ...
        )
```

---

## 3. Dependency Injection System

Đây là **trái tim của FastAPI** - một hệ thống DI phức tạp và elegant.

### 3.1. Dataclass Dependant

**File**: `fastapi/dependencies/models.py`

```python
from dataclasses import dataclass, field
from typing import List, Optional, Callable

@dataclass
class Dependant:
    """Đại diện cho một callable và dependency graph của nó"""

    # Parameters được phân loại theo location
    path_params: List[ModelField] = field(default_factory=list)    # từ URL path
    query_params: List[ModelField] = field(default_factory=list)   # từ query string
    header_params: List[ModelField] = field(default_factory=list)  # từ headers
    cookie_params: List[ModelField] = field(default_factory=list)  # từ cookies
    body_params: List[ModelField] = field(default_factory=list)    # từ request body

    # Nested dependencies (RECURSIVE!)
    dependencies: List["Dependant"] = field(default_factory=list)

    # Security requirements
    security_requirements: List[SecurityRequirement] = field(default_factory=list)

    # Special parameters
    request_param_name: Optional[str] = None      # tên parameter cho Request object
    response_param_name: Optional[str] = None     # tên parameter cho Response object
    background_tasks_param_name: Optional[str] = None

    # The actual callable
    call: Optional[Callable] = None

    # Caching
    use_cache: bool = True
    cache_key: Tuple = field(init=False)
```

**Tại sao dùng dataclass?**
- Immutable (sau khi tạo)
- Auto-generated `__init__`, `__repr__`, `__eq__`
- Type hints được enforced
- Clean và readable

### 3.2. Hàm get_dependant() - Phân Tích Function Signature

**File**: `fastapi/dependencies/utils.py`

```python
def get_dependant(
    *,
    path: str,
    call: Callable[..., Any],
    name: Optional[str] = None,
    security_scopes: Optional[List[str]] = None,
    use_cache: bool = True,
) -> Dependant:
    """
    Phân tích function signature và tạo Dependant object
    """
    # 1. Tạo empty Dependant
    dependant = Dependant(call=call, name=name, path=path, use_cache=use_cache)

    # 2. Lấy function signature
    signature = inspect.signature(call)

    # 3. Duyệt qua từng parameter
    for param_name, param in signature.parameters.items():
        # 4. Phân tích parameter annotation
        type_annotation = param.annotation

        # 5. Check nếu là special type (Request, Response, WebSocket, etc.)
        if type_annotation is Request:
            dependant.request_param_name = param_name
            continue
        elif type_annotation is Response:
            dependant.response_param_name = param_name
            continue
        elif type_annotation is BackgroundTasks:
            dependant.background_tasks_param_name = param_name
            continue

        # 6. Phân tích Annotated types
        param_details = analyze_param(
            param_name=param_name,
            annotation=type_annotation,
            value=param.default,
        )

        # 7. Nếu là dependency (Depends)
        if param_details.depends is not None:
            sub_dependant = get_param_sub_dependant(
                param_name=param_name,
                depends=param_details.depends,
                path=path,
            )
            dependant.dependencies.append(sub_dependant)

        # 8. Nếu là parameter thông thường
        elif param_details.field is not None:
            add_param_to_fields(field=param_details.field, dependant=dependant)

    return dependant
```

**Kỹ thuật quan trọng**:

#### Signature Introspection với `inspect` module:

```python
import inspect

def my_endpoint(
    item_id: int,
    skip: int = 0,
    limit: int = Query(10, gt=0),
):
    pass

# Phân tích:
sig = inspect.signature(my_endpoint)

for name, param in sig.parameters.items():
    print(f"Name: {name}")
    print(f"Annotation: {param.annotation}")
    print(f"Default: {param.default}")
    print(f"Kind: {param.kind}")
    print("---")

# Output:
# Name: item_id
# Annotation: <class 'int'>
# Default: <class 'inspect._empty'>
# Kind: POSITIONAL_OR_KEYWORD
# ---
# Name: skip
# Annotation: <class 'int'>
# Default: 0
# ...
```

#### Xử lý `Annotated` type:

```python
from typing import Annotated, get_origin, get_args

# Ví dụ annotation
annotation = Annotated[int, Query(gt=0, le=100)]

# Extract type và metadata
origin = get_origin(annotation)  # typing.Annotated
args = get_args(annotation)       # (int, Query(...))

if origin is Annotated:
    actual_type = args[0]         # int
    metadata = args[1:]           # (Query(...),)

    for item in metadata:
        if isinstance(item, Query):
            # Đây là Query parameter với validation
            print(f"Query param: {item.gt=}, {item.le=}")
```

### 3.3. Hàm solve_dependencies() - Recursive Resolution

**File**: `fastapi/dependencies/utils.py`

```python
async def solve_dependencies(
    *,
    request: Union[Request, WebSocket],
    dependant: Dependant,
    body: Optional[Union[Dict, FormData]] = None,
    dependency_cache: Optional[Dict] = None,
    async_exit_stack: AsyncExitStack,
    ...
) -> SolvedDependency:
    """
    Recursively resolve tất cả dependencies
    """
    values: Dict[str, Any] = {}
    errors: List[Any] = []
    dependency_cache = dependency_cache or {}

    # 1. RECURSIVE: Resolve sub-dependencies trước
    for sub_dependant in dependant.dependencies:
        call = sub_dependant.call

        # Check cache
        if sub_dependant.use_cache and sub_dependant.cache_key in dependency_cache:
            solved = dependency_cache[sub_dependant.cache_key]
        else:
            # RECURSIVE CALL!
            solved_result = await solve_dependencies(
                request=request,
                dependant=sub_dependant,  # ← Recursion here!
                body=body,
                dependency_cache=dependency_cache,
                async_exit_stack=async_exit_stack,
                ...
            )

            # Check errors
            if solved_result.errors:
                errors.extend(solved_result.errors)
                continue

            # Execute the dependency
            if is_gen_callable(call):
                # Generator - supports cleanup
                solved = await solve_generator(
                    call=call,
                    stack=async_exit_stack,
                    sub_values=solved_result.values
                )
            elif is_coroutine_callable(call):
                # Async function
                solved = await call(**solved_result.values)
            else:
                # Sync function - run in thread pool
                solved = await run_in_threadpool(call, **solved_result.values)

            # Cache result
            if sub_dependant.cache_key not in dependency_cache:
                dependency_cache[sub_dependant.cache_key] = solved

        # Store in values dict
        if sub_dependant.name is not None:
            values[sub_dependant.name] = solved

    # 2. Parse path parameters
    path_values, path_errors = request_params_to_args(
        dependant.path_params,
        request.path_params
    )
    values.update(path_values)
    errors.extend(path_errors)

    # 3. Parse query parameters
    query_values, query_errors = request_params_to_args(
        dependant.query_params,
        request.query_params
    )
    values.update(query_values)
    errors.extend(query_errors)

    # 4. Parse header parameters
    header_values, header_errors = request_params_to_args(
        dependant.header_params,
        request.headers
    )
    values.update(header_values)
    errors.extend(header_errors)

    # 5. Parse cookie parameters
    cookie_values, cookie_errors = request_params_to_args(
        dependant.cookie_params,
        request.cookies
    )
    values.update(cookie_values)
    errors.extend(cookie_errors)

    # 6. Parse request body
    if dependant.body_params:
        body_values, body_errors = await request_body_to_args(
            required_params=dependant.body_params,
            received_body=body,
        )
        values.update(body_values)
        errors.extend(body_errors)

    # 7. Add special parameters (Request, Response, etc.)
    if dependant.request_param_name:
        values[dependant.request_param_name] = request
    if dependant.response_param_name:
        values[dependant.response_param_name] = response

    return SolvedDependency(
        values=values,
        errors=errors,
        dependency_cache=dependency_cache,
        ...
    )
```

**Điểm mạnh của thiết kế này**:

1. **Recursive Graph Resolution**: Tự động resolve dependencies của dependencies
2. **Caching**: Tránh execute cùng dependency nhiều lần
3. **Error Accumulation**: Collect tất cả validation errors cùng lúc
4. **Generator Support**: Cho phép cleanup code với `yield`
5. **Async/Sync Support**: Tự động detect và xử lý cả sync và async functions

### 3.4. Ví Dụ Dependency Injection Thực Tế

```python
from fastapi import Depends, FastAPI

app = FastAPI()

# Level 1: Database connection
async def get_db():
    db = Database()
    try:
        yield db
    finally:
        await db.close()

# Level 2: Repository (depends on DB)
async def get_user_repo(db = Depends(get_db)):
    return UserRepository(db)

# Level 3: Service (depends on Repository)
async def get_user_service(repo = Depends(get_user_repo)):
    return UserService(repo)

# Level 4: Current user (depends on Service)
async def get_current_user(
    token: str = Header(...),
    service = Depends(get_user_service)
):
    return await service.get_user_by_token(token)

# Endpoint sử dụng tất cả
@app.get("/profile")
async def get_profile(user = Depends(get_current_user)):
    return user

# Dependency graph:
# get_profile
#   ↓
# get_current_user
#   ↓
# get_user_service
#   ↓
# get_user_repo
#   ↓
# get_db
```

**FastAPI tự động**:
1. Phân tích dependency graph
2. Resolve theo đúng thứ tự (bottom-up)
3. Cache các dependencies đã execute
4. Cleanup generators khi request kết thúc

---

## 4. Request Processing Pipeline

### 4.1. Luồng Xử Lý Request

**File**: `fastapi/routing.py` - function `get_request_handler()`

```python
def get_request_handler(
    dependant: Dependant,
    body_field: Optional[ModelField] = None,
    status_code: Optional[int] = None,
    response_class: Type[Response] = JSONResponse,
    response_field: Optional[ModelField] = None,
    ...
) -> Callable[[Request], Coroutine[Any, Any, Response]]:
    """
    Tạo ASGI-compatible request handler
    Returns một async function nhận Request và return Response
    """

    # Check if endpoint is async or sync
    is_coroutine = asyncio.iscoroutinefunction(dependant.call)
    is_body_form = body_field and isinstance(body_field.field_info, params.Form)

    # Inner async function - the actual handler
    async def app(request: Request) -> Response:
        """
        Đây là function thực sự xử lý request
        """
        response: Union[Response, None] = None

        # Context manager để cleanup files
        async with AsyncExitStack() as file_stack:
            # STEP 1: Parse request body
            body: Any = None
            if body_field:
                if is_body_form:
                    # Form data (multipart/form-data hoặc application/x-www-form-urlencoded)
                    body = await request.form()
                    file_stack.push_async_callback(body.close)
                else:
                    # JSON body
                    body_bytes = await request.body()
                    if body_bytes:
                        # Check content-type
                        content_type = request.headers.get("content-type")
                        if content_type and "json" in content_type:
                            body = await request.json()
                        else:
                            body = body_bytes

            errors: List[Any] = []

            # Context manager để cleanup dependencies
            async with AsyncExitStack() as async_exit_stack:
                # STEP 2: Solve all dependencies
                solved_result = await solve_dependencies(
                    request=request,
                    dependant=dependant,
                    body=body,
                    async_exit_stack=async_exit_stack,
                    ...
                )

                errors = solved_result.errors

                if not errors:
                    # STEP 3: Call endpoint function
                    raw_response = await run_endpoint_function(
                        dependant=dependant,
                        values=solved_result.values,
                        is_coroutine=is_coroutine,
                    )

                    # STEP 4: Handle response
                    if isinstance(raw_response, Response):
                        # Endpoint returned Response directly
                        response = raw_response
                    else:
                        # STEP 5: Serialize response với response_model
                        content = await serialize_response(
                            field=response_field,
                            response_content=raw_response,
                            exclude_unset=response_model_exclude_unset,
                            ...
                        )

                        # STEP 6: Create Response object
                        response = response_class(
                            content,
                            status_code=status_code,
                            background=solved_result.background_tasks,
                        )

            # STEP 7: Raise validation errors if any
            if errors:
                raise RequestValidationError(errors, body=body)

        # STEP 8: Return response
        return response

    return app
```

### 4.2. Diagram Luồng Xử Lý

```
HTTP Request
    ↓
[Starlette Router] - Match route
    ↓
[APIRoute.app] - ASGI handler
    ↓
┌────────────────────────────────────┐
│  get_request_handler() creates:    │
│                                    │
│  async def app(request):           │
│    1. Parse body (JSON/Form)       │
│    2. solve_dependencies()         │
│    3. Call endpoint function       │
│    4. Validate response            │
│    5. Serialize response           │
│    6. Return Response              │
└────────────────────────────────────┘
    ↓
[Starlette] - Send response
    ↓
HTTP Response
```

### 4.3. Function run_endpoint_function()

```python
async def run_endpoint_function(
    *,
    dependant: Dependant,
    values: Dict[str, Any],
    is_coroutine: bool
) -> Any:
    """
    Execute endpoint function với resolved dependency values
    """
    assert dependant.call is not None

    if is_coroutine:
        # Async function - await directly
        return await dependant.call(**values)
    else:
        # Sync function - run in thread pool để không block event loop
        return await run_in_threadpool(dependant.call, **values)
```

**Tại sao run sync functions trong thread pool?**

Async event loop là single-threaded. Nếu một sync function chạy lâu (e.g., CPU-intensive task, blocking I/O), nó sẽ block toàn bộ event loop, khiến server không thể xử lý requests khác.

Solution: FastAPI tự động detect sync functions và run chúng trong thread pool với `run_in_threadpool()` (từ Starlette).

```python
# Sync endpoint - FastAPI runs this in thread pool
@app.get("/sync")
def sync_endpoint():
    time.sleep(1)  # Blocking call - không block event loop
    return {"message": "ok"}

# Async endpoint - runs directly on event loop
@app.get("/async")
async def async_endpoint():
    await asyncio.sleep(1)  # Non-blocking
    return {"message": "ok"}
```

---

## 5. Type System & Validation

### 5.1. Pydantic Integration

FastAPI sử dụng Pydantic để:
1. Parse và validate request parameters
2. Serialize response data
3. Generate JSON schemas cho OpenAPI

#### Request Body Validation:

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    price: float = Field(..., gt=0)
    tax: Optional[float] = None
    tags: List[str] = []

@app.post("/items/")
async def create_item(item: Item):
    return item

# Request:
# POST /items/
# {"name": "Laptop", "price": 999.99, "tags": ["electronics"]}

# FastAPI internally:
# 1. Nhận JSON body
# 2. Parse thành dict
# 3. Gọi Item(**dict) - Pydantic validates
# 4. Nếu valid: pass Item instance vào function
# 5. Nếu invalid: return 422 với validation errors
```

### 5.2. ModelField - Wrapper cho Pydantic Fields

FastAPI sử dụng Pydantic's `ModelField` để represent parameters:

```python
from fastapi.utils import create_model_field

# Tạo ModelField cho một parameter
field = create_model_field(
    name="item_id",
    type_=int,
    default=Path(..., gt=0),  # Path parameter với validation
    field_info=Path(..., gt=0),
)

# ModelField chứa:
# - name: "item_id"
# - type_: int
# - required: True
# - field_info: Path object with validators
```

### 5.3. Parameter Parsing với request_params_to_args()

```python
def request_params_to_args(
    required_params: Sequence[ModelField],
    received_params: Union[Mapping[str, Any], QueryParams, Headers],
) -> Tuple[Dict[str, Any], List[Any]]:
    """
    Parse và validate parameters
    """
    values = {}
    errors = []

    for field in required_params:
        # Get value từ request
        if field.name in received_params:
            value = received_params[field.name]
        elif field.required:
            # Missing required parameter
            errors.append(get_missing_field_error(field))
            continue
        else:
            # Use default value
            value = field.default

        # Validate với Pydantic
        v_, errors_ = field.validate(value, {}, loc=(field.name,))

        if errors_:
            errors.extend(errors_)
        else:
            values[field.name] = v_

    return values, errors
```

### 5.4. Response Serialization

```python
async def serialize_response(
    *,
    field: Optional[ModelField] = None,
    response_content: Any,
    exclude_unset: bool = False,
    exclude_defaults: bool = False,
    exclude_none: bool = False,
    ...
) -> Any:
    """
    Validate và serialize response
    """
    if field:
        # STEP 1: Validate response content
        value, errors = field.validate(response_content, {}, loc=("response",))

        if errors:
            raise ResponseValidationError(errors, body=response_content)

        # STEP 2: Serialize to JSON-compatible format
        if hasattr(field, "serialize"):  # Pydantic v2
            return field.serialize(
                value,
                exclude_unset=exclude_unset,
                exclude_defaults=exclude_defaults,
                exclude_none=exclude_none,
            )
        else:  # Pydantic v1
            return jsonable_encoder(
                value,
                exclude_unset=exclude_unset,
                ...
            )
    else:
        # No response model - just encode
        return jsonable_encoder(response_content)
```

---

## 6. Async/Await Implementation

### 6.1. Async Context Management

FastAPI sử dụng `AsyncExitStack` để quản lý cleanup:

```python
from contextlib import AsyncExitStack

async def handle_request(request):
    async with AsyncExitStack() as stack:
        # Register cleanup callbacks

        # Example 1: Generator dependency
        async def get_db():
            db = Database()
            try:
                yield db
            finally:
                await db.close()

        db = await stack.enter_async_context(get_db())

        # Use db...

        # When exiting the context, all cleanup code runs automatically
```

### 6.2. Generator Dependencies

FastAPI supports generator dependencies với cleanup:

```python
async def get_db():
    """Generator dependency với cleanup"""
    db = Database()
    try:
        yield db  # Provide dependency
    finally:
        await db.close()  # Cleanup after request

# Internal implementation:
async def solve_generator(
    *,
    call: Callable,
    stack: AsyncExitStack,
    sub_values: Dict[str, Any]
) -> Any:
    if is_gen_callable(call):
        # Sync generator
        cm = contextmanager_in_threadpool(contextmanager(call)(**sub_values))
    elif is_async_gen_callable(call):
        # Async generator
        cm = asynccontextmanager(call)(**sub_values)

    # Enter context và register cleanup
    return await stack.enter_async_context(cm)
```

### 6.3. Detection: Async vs Sync

```python
import inspect

def is_coroutine_callable(call: Callable) -> bool:
    """Check if callable is async"""
    if inspect.isroutine(call):
        return inspect.iscoroutinefunction(call)
    if inspect.isclass(call):
        return False
    # Check __call__ method for callable objects
    dunder_call = getattr(call, "__call__", None)
    return inspect.iscoroutinefunction(dunder_call)

def is_async_gen_callable(call: Callable) -> bool:
    """Check if callable is async generator"""
    if inspect.isasyncgenfunction(call):
        return True
    dunder_call = getattr(call, "__call__", None)
    return inspect.isasyncgenfunction(dunder_call)

def is_gen_callable(call: Callable) -> bool:
    """Check if callable is sync generator"""
    if inspect.isgeneratorfunction(call):
        return True
    dunder_call = getattr(call, "__call__", None)
    return inspect.isgeneratorfunction(dunder_call)
```

### 6.4. Thread Pool cho Sync Functions

```python
from starlette.concurrency import run_in_threadpool

# FastAPI's strategy:
async def execute_dependency(call, values):
    if is_coroutine_callable(call):
        # Async - run directly
        return await call(**values)
    else:
        # Sync - run in thread pool
        return await run_in_threadpool(call, **values)
```

**Best Practice**: Prefer async functions khi có thể, nhưng FastAPI vẫn support sync functions một cách transparent.

---

## 7. OpenAPI Auto-Documentation

### 7.1. Schema Generation Pipeline

**File**: `fastapi/openapi/utils.py`

```python
def get_openapi(
    *,
    title: str,
    version: str,
    openapi_version: str = "3.1.0",
    description: Optional[str] = None,
    routes: Sequence[BaseRoute],
    tags: Optional[List[Dict[str, Any]]] = None,
    servers: Optional[List[Dict[str, Union[str, Any]]]] = None,
    ...
) -> Dict[str, Any]:
    """
    Generate complete OpenAPI schema
    """
    info: Dict[str, Any] = {"title": title, "version": version}
    if description:
        info["description"] = description

    output: Dict[str, Any] = {"openapi": openapi_version, "info": info}

    if servers:
        output["servers"] = servers

    # Main part: Extract paths from routes
    paths: Dict[str, Dict[str, Any]] = {}

    for route in routes:
        if isinstance(route, APIRoute):
            # Get flat dependency list
            flat_dependant = get_flat_dependant(route.dependant, skip_repeats=True)

            # Generate operation schema
            operation = {
                "summary": route.summary or route.name,
                "operationId": route.operation_id or route.unique_id,
                "responses": get_openapi_operation_responses(...),
            }

            # Add parameters (path, query, header, cookie)
            if parameters := get_openapi_operation_parameters(flat_dependant):
                operation["parameters"] = parameters

            # Add request body
            if request_body := get_openapi_operation_request_body(
                body_field=route.body_field,
                ...
            ):
                operation["requestBody"] = request_body

            # Add tags
            if route.tags:
                operation["tags"] = route.tags

            # Add security
            if security_requirements := get_openapi_security_requirements(
                flat_dependant.security_requirements
            ):
                operation["security"] = security_requirements

            # Add to paths
            path = route.path_format
            if path not in paths:
                paths[path] = {}

            for method in route.methods:
                paths[path][method.lower()] = operation

    output["paths"] = paths

    # Add components (schemas, security schemes)
    if components := get_openapi_components(...):
        output["components"] = components

    return output
```

### 7.2. Lazy Loading Pattern

```python
class FastAPI(Starlette):
    def __init__(self, ...):
        self.openapi_schema: Optional[Dict[str, Any]] = None

    def openapi(self) -> Dict[str, Any]:
        """
        Generate OpenAPI schema (cached)
        """
        if self.openapi_schema:
            # Already generated - return cached
            return self.openapi_schema

        # Generate for the first time
        self.openapi_schema = get_openapi(
            title=self.title,
            version=self.version,
            routes=self.routes,
            ...
        )

        return self.openapi_schema
```

**Tại sao lazy loading?**
- OpenAPI schema chỉ cần generate 1 lần
- Không cần generate nếu không ai access `/docs` hoặc `/openapi.json`
- Tiết kiệm startup time

### 7.3. Automatic Docs Endpoints

```python
def setup(self) -> None:
    """Setup docs endpoints"""
    if self.openapi_url:
        # OpenAPI JSON endpoint
        async def openapi(req: Request) -> JSONResponse:
            return JSONResponse(self.openapi())

        self.add_route(self.openapi_url, openapi, include_in_schema=False)

    if self.docs_url:
        # Swagger UI
        async def swagger_ui_html(req: Request) -> HTMLResponse:
            return get_swagger_ui_html(
                openapi_url=self.openapi_url,
                title=self.title + " - Swagger UI",
            )

        self.add_route(self.docs_url, swagger_ui_html, include_in_schema=False)

    if self.redoc_url:
        # ReDoc
        async def redoc_html(req: Request) -> HTMLResponse:
            return get_redoc_html(
                openapi_url=self.openapi_url,
                title=self.title + " - ReDoc",
            )

        self.add_route(self.redoc_url, redoc_html, include_in_schema=False)
```

---

## 8. Design Patterns Được Sử Dụng

### 8.1. Decorator Factory Pattern

```python
# Pattern:
def decorator_factory(**kwargs):
    def decorator(func):
        # Do something with func and kwargs
        return func
    return decorator

# Usage:
@decorator_factory(param="value")
def my_function():
    pass

# FastAPI implementation:
class APIRouter:
    def get(self, path: str, **kwargs):
        def decorator(func: Callable) -> Callable:
            self.add_api_route(path, func, methods=["GET"], **kwargs)
            return func
        return decorator
```

### 8.2. Dependency Injection Pattern

```python
# Classic DI pattern
class Service:
    def __init__(self, repo: Repository):
        self.repo = repo

# FastAPI's DI pattern
def get_service(repo: Repository = Depends(get_repo)):
    return Service(repo)

# DI Container equivalent:
container = {
    Repository: lambda: get_repo(),
    Service: lambda: get_service(container[Repository]()),
}
```

### 8.3. Lazy Initialization Pattern

```python
class FastAPI:
    def __init__(self):
        self._openapi_schema = None

    @property
    def openapi_schema(self):
        if self._openapi_schema is None:
            self._openapi_schema = self._generate_openapi()
        return self._openapi_schema
```

### 8.4. Composition over Inheritance

```python
# FastAPI uses composition
class FastAPI(Starlette):
    def __init__(self):
        super().__init__()
        self.router = APIRouter()  # ← Composition

    # Delegate methods
    def get(self, path: str, **kwargs):
        return self.router.get(path, **kwargs)
```

### 8.5. Strategy Pattern (Response Classes)

```python
# Different response strategies
class JSONResponse(Response):
    def render(self, content: Any) -> bytes:
        return json.dumps(content).encode("utf-8")

class ORJSONResponse(Response):
    def render(self, content: Any) -> bytes:
        return orjson.dumps(content)

# Usage:
@app.get("/", response_class=ORJSONResponse)
def endpoint():
    return {"message": "hello"}
```

### 8.6. Builder Pattern (Parameter Functions)

```python
# Building parameter with validation
def Query(
    default: Any = ...,
    *,
    alias: Optional[str] = None,
    gt: Optional[float] = None,
    ge: Optional[float] = None,
    lt: Optional[float] = None,
    le: Optional[float] = None,
    min_length: Optional[int] = None,
    max_length: Optional[int] = None,
    regex: Optional[str] = None,
    ...
) -> Any:
    return params.Query(
        default=default,
        alias=alias,
        gt=gt,
        ge=ge,
        ...
    )

# Usage - building a complex parameter:
limit: int = Query(10, ge=1, le=100, description="Items per page")
```

---

## 9. Bài Học Quan Trọng

### 9.1. Python Advanced Techniques

#### 1. **Type Introspection với `inspect` module**

```python
import inspect
from typing import get_type_hints

def analyze_function(func):
    # Get signature
    sig = inspect.signature(func)

    # Get type hints (resolves forward references)
    hints = get_type_hints(func)

    # Get parameters
    for name, param in sig.parameters.items():
        print(f"{name}: {hints.get(name, 'Any')} = {param.default}")
```

#### 2. **Dataclasses cho Data Structures**

```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class Dependant:
    path_params: List[str] = field(default_factory=list)
    query_params: List[str] = field(default_factory=list)

    # Post-init processing
    def __post_init__(self):
        self.all_params = self.path_params + self.query_params
```

#### 3. **AsyncExitStack cho Resource Management**

```python
from contextlib import AsyncExitStack

async def handle_with_cleanup():
    async with AsyncExitStack() as stack:
        # Multiple resources
        db = await stack.enter_async_context(get_db())
        cache = await stack.enter_async_context(get_cache())

        # All cleanup happens automatically
```

#### 4. **Generic Type Handling**

```python
from typing import get_origin, get_args, Union

def analyze_type(tp):
    origin = get_origin(tp)  # List, Dict, Union, etc.
    args = get_args(tp)      # Type arguments

    if origin is Union:
        # Optional[X] is Union[X, None]
        types = args
    elif origin is list:
        item_type = args[0] if args else Any
```

#### 5. **Function Wrapping với functools**

```python
from functools import wraps

def decorator(func):
    @wraps(func)  # Preserves metadata
    async def wrapper(*args, **kwargs):
        # Pre-processing
        result = await func(*args, **kwargs)
        # Post-processing
        return result
    return wrapper
```

### 9.2. Architecture Lessons

#### 1. **Separation of Concerns**

FastAPI tách biệt rõ ràng:
- Routing logic (`routing.py`)
- Dependency resolution (`dependencies/utils.py`)
- Validation (`_compat.py`, Pydantic)
- OpenAPI generation (`openapi/utils.py`)
- Security (`security/`)

#### 2. **Single Responsibility Principle**

Mỗi class/function có 1 nhiệm vụ rõ ràng:
- `Dependant`: Represent dependency graph
- `get_dependant()`: Analyze function signature
- `solve_dependencies()`: Resolve dependencies
- `get_request_handler()`: Create request handler

#### 3. **Open/Closed Principle**

FastAPI extensible without modification:
- Custom response classes
- Custom exception handlers
- Custom dependencies
- Middleware

#### 4. **Dependency Inversion**

High-level modules không phụ thuộc vào low-level:
- FastAPI depends on abstract `Starlette`
- Uses Pydantic interface, not implementation details
- Dependency injection inverts control flow

### 9.3. Performance Optimizations

#### 1. **Lazy Loading**

```python
# Don't generate OpenAPI until needed
if not self.openapi_schema:
    self.openapi_schema = generate_openapi(...)
```

#### 2. **Dependency Caching**

```python
# Cache expensive dependencies
if cache_key in dependency_cache:
    return dependency_cache[cache_key]

result = await compute_dependency()
dependency_cache[cache_key] = result
```

#### 3. **Thread Pool cho Sync Functions**

```python
# Avoid blocking event loop
if is_sync_function(call):
    result = await run_in_threadpool(call, **kwargs)
```

#### 4. **Async Context Managers**

```python
# Efficient resource management
async with AsyncExitStack() as stack:
    # Automatic cleanup
```

---

## 10. Hướng Dẫn Xây Dựng Framework Riêng

Dựa trên phân tích FastAPI, đây là roadmap để xây dựng framework của bạn:

### Phase 1: Foundation (Week 1-2)

#### Step 1: Choose Base (ASGI Framework)

```python
# Option 1: Build on Starlette
from starlette.applications import Starlette
from starlette.routing import Route

class MyFramework(Starlette):
    pass

# Option 2: Build on raw ASGI
async def application(scope, receive, send):
    """Raw ASGI app"""
    pass
```

**Recommendation**: Start với Starlette để focus vào features, không phải low-level ASGI.

#### Step 2: Implement Basic Routing

```python
class MyFramework:
    def __init__(self):
        self.routes = []

    def route(self, path: str, methods: List[str] = ["GET"]):
        def decorator(func):
            self.routes.append({
                "path": path,
                "methods": methods,
                "handler": func,
            })
            return func
        return decorator

    def get(self, path: str):
        return self.route(path, methods=["GET"])

    def post(self, path: str):
        return self.route(path, methods=["POST"])
```

### Phase 2: Dependency Injection (Week 3-4)

#### Step 1: Signature Introspection

```python
import inspect
from typing import get_type_hints

def analyze_handler(handler):
    sig = inspect.signature(handler)
    hints = get_type_hints(handler)

    dependencies = {}
    for param_name, param in sig.parameters.items():
        param_type = hints.get(param_name)
        dependencies[param_name] = {
            "type": param_type,
            "default": param.default,
        }

    return dependencies
```

#### Step 2: Implement Dependency Resolution

```python
class DependencyManager:
    def __init__(self):
        self.providers = {}

    def register(self, type_: Type, provider: Callable):
        """Register dependency provider"""
        self.providers[type_] = provider

    async def resolve(self, type_: Type):
        """Resolve dependency"""
        if type_ in self.providers:
            provider = self.providers[type_]
            return await provider()

        # Auto-instantiate if possible
        return type_()
```

### Phase 3: Validation (Week 5-6)

#### Option 1: Use Pydantic

```python
from pydantic import BaseModel

class MyFramework:
    async def handle_request(self, request, handler):
        # Analyze handler signature
        sig = inspect.signature(handler)
        hints = get_type_hints(handler)

        kwargs = {}
        for param_name, param in sig.parameters.items():
            param_type = hints.get(param_name)

            if issubclass(param_type, BaseModel):
                # Parse request body as Pydantic model
                body = await request.json()
                kwargs[param_name] = param_type(**body)

        return await handler(**kwargs)
```

#### Option 2: Build Your Own Validator

```python
class Field:
    def __init__(self, type_: Type, required: bool = True, validators: List = None):
        self.type_ = type_
        self.required = required
        self.validators = validators or []

    def validate(self, value):
        # Type validation
        if not isinstance(value, self.type_):
            try:
                value = self.type_(value)
            except:
                raise ValidationError(f"Expected {self.type_}")

        # Custom validators
        for validator in self.validators:
            validator(value)

        return value
```

### Phase 4: Documentation (Week 7-8)

#### Generate OpenAPI Schema

```python
def generate_openapi_schema(routes):
    schema = {
        "openapi": "3.1.0",
        "info": {"title": "My API", "version": "1.0.0"},
        "paths": {},
    }

    for route in routes:
        path = route["path"]
        method = route["methods"][0].lower()
        handler = route["handler"]

        # Analyze handler
        sig = inspect.signature(handler)
        hints = get_type_hints(handler)

        operation = {
            "summary": handler.__name__,
            "parameters": [],
            "responses": {"200": {"description": "Success"}},
        }

        # Add to schema
        if path not in schema["paths"]:
            schema["paths"][path] = {}
        schema["paths"][path][method] = operation

    return schema
```

### Phase 5: Advanced Features (Week 9-12)

1. **Middleware Support**
2. **WebSocket Support**
3. **Background Tasks**
4. **Testing Client**
5. **CLI Tools**

### Example: Minimal Framework

```python
# myframework.py
import inspect
import json
from typing import Any, Callable, Dict, List, get_type_hints
from starlette.applications import Starlette
from starlette.requests import Request
from starlette.responses import JSONResponse
from starlette.routing import Route
from pydantic import BaseModel

class MyFramework:
    def __init__(self, title: str = "My API"):
        self.title = title
        self.routes: List[Dict] = []

    def get(self, path: str):
        """Decorator for GET endpoints"""
        def decorator(func: Callable):
            self.routes.append({
                "path": path,
                "methods": ["GET"],
                "handler": func,
            })
            return func
        return decorator

    def post(self, path: str):
        """Decorator for POST endpoints"""
        def decorator(func: Callable):
            self.routes.append({
                "path": path,
                "methods": ["POST"],
                "handler": func,
            })
            return func
        return decorator

    async def _handle_request(self, handler: Callable, request: Request) -> JSONResponse:
        """Handle request with automatic dependency injection"""
        sig = inspect.signature(handler)
        hints = get_type_hints(handler)

        kwargs = {}

        # Inject dependencies
        for param_name, param in sig.parameters.items():
            param_type = hints.get(param_name)

            if param_type is Request:
                # Inject Request object
                kwargs[param_name] = request
            elif param_type and issubclass(param_type, BaseModel):
                # Parse body as Pydantic model
                body = await request.json()
                kwargs[param_name] = param_type(**body)
            elif param_name in request.path_params:
                # Path parameter
                kwargs[param_name] = request.path_params[param_name]
            elif param_name in request.query_params:
                # Query parameter
                value = request.query_params[param_name]
                # Type coercion
                if param_type:
                    value = param_type(value)
                kwargs[param_name] = value

        # Call handler
        result = await handler(**kwargs) if inspect.iscoroutinefunction(handler) else handler(**kwargs)

        # Return JSON response
        return JSONResponse(result)

    def build_asgi_app(self):
        """Build ASGI application"""
        routes = []

        for route_info in self.routes:
            path = route_info["path"]
            methods = route_info["methods"]
            handler = route_info["handler"]

            async def endpoint(request: Request, handler=handler):
                return await self._handle_request(handler, request)

            routes.append(Route(path, endpoint, methods=methods))

        return Starlette(routes=routes)

# Usage example:
app = MyFramework(title="My API")

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    return {"item_id": item_id}

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/")
async def create_item(item: Item):
    return {"name": item.name, "price": item.price}

# Run:
# uvicorn myframework:app.build_asgi_app()
```

### Key Takeaways cho Framework Development

1. **Start Simple**: Implement core routing trước
2. **Iterate**: Add features từng bước một
3. **Use Existing Libraries**: Đừng reinvent the wheel (Starlette, Pydantic)
4. **Type Hints**: Foundation cho modern Python frameworks
5. **Documentation**: Generate từ code, đừng viết manual
6. **Testing**: Test framework code kỹ lưỡng
7. **Performance**: Profile và optimize critical paths
8. **Community**: Release early, get feedback

---

## Kết Luận

FastAPI là một kiệt tác về:

1. **Modern Python**: Type hints, async/await, dataclasses
2. **Clean Architecture**: Separation of concerns, SOLID principles
3. **Developer Experience**: Auto-documentation, validation, error messages
4. **Performance**: Async by default, minimal overhead

Để xây dựng framework riêng, học từ FastAPI:

- Sử dụng type introspection để build magic
- Dependency injection cho modularity
- Validation với Pydantic
- OpenAPI cho documentation
- Async/await cho performance

**Next Steps**:
1. Đọc source code các modules cụ thể
2. Implement một mini framework
3. Study Starlette và Pydantic chi tiết
4. Thử nghiệm với các design patterns

Good luck với việc xây dựng framework của bạn! 🚀
