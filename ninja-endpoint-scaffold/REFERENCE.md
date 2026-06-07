# Reference: Django Ninja API Endpoint Scaffold

> Back to [SKILL.md](SKILL.md)

---

## 1. Complete Correct Example

This example demonstrates all patterns from `openspec/changes/archive/2026-01-07-improve-llm-coding-rules/`.

### 1.1 API Layer (`sheetScript/api/item_api.py`)

```python
import logging
from ninja import Router, Schema
from sheetScript.api import core
from sheetScript.api.bearer import AuthBearer
from sheetScript.api.core import NinjaHttpRequest, Reply
from sheetScript.models import ActionName
from sheetScript.services.item_service import ItemService, ItemCreateParams

logger = logging.getLogger(__name__)
router = Router(auth=AuthBearer())


class ItemOut(Schema):
    """Output schema for Item."""
    id: int
    name: str
    category: str | None = None
    created_at: str

    model_config = {"from_attributes": True}


class ItemList(Schema):
    items: list[ItemOut]


class CreateIn(Schema):
    name: str
    category: str | None = None


class IdOut(Schema):
    id: int


@router.get("/", response={200: Reply[ItemList]})
def list_items(request: NinjaHttpRequest):
    """List all items."""
    service = ItemService(user=request.auth)
    items = service.get_items()
    return Reply(data=ItemList(items=[
        ItemOut.from_orm(item) for item in items
    ]))


@router.get("/{item_id}", response={200: Reply[ItemOut], 404: Error})
def get_item(request: NinjaHttpRequest, item_id: int):
    """Get item by ID."""
    service = ItemService(user=request.auth)
    item = service.get_item(item_id)
    return Reply(data=ItemOut.from_orm(item))


@core.validate_permission(ActionName.CREATE_ITEMS)
@router.post("/", response={201: Reply[IdOut]})
def create_item(request: NinjaHttpRequest, data: CreateIn):
    """Create a new item."""
    service = ItemService(user=request.auth)
    create_data = ItemCreateParams(**data.model_dump())
    item_id = service.create_item(create_data)
    return 201, Reply(data=IdOut(id=item_id), info="Item created successfully")
```

### 1.2 Service Layer (`sheetScript/services/item_service.py`)

```python
import logging
from django.db import transaction
from sheetScript.models import Item, NocAcc
from sheetScript.exceptions import ItemNotFoundError, ItemValidationError

logger = logging.getLogger(__name__)


class ItemCreateParams:
    def __init__(self, name: str, category: str | None = None):
        self.name = name
        self.category = category


class ItemService:
    def __init__(self, user: NocAcc | None = None):
        self.user = user

    def get_items(self) -> list[Item]:
        try:
            return list(Item.objects.all().order_by("-created_at"))
        except Exception as e:
            logger.error(f"Failed to get items: {e}", exc_info=True)
            raise

    def get_item(self, item_id: int) -> Item:
        try:
            return Item.objects.get(id=item_id)
        except Item.DoesNotExist as e:
            raise ItemNotFoundError(f"Item {item_id} not found") from e
        except Exception as e:
            logger.error(f"Failed to get item {item_id}: {e}", exc_info=True)
            raise

    @transaction.atomic
    def create_item(self, data: ItemCreateParams) -> int:
        if not data.name or len(data.name.strip()) == 0:
            raise ItemValidationError("Item name cannot be empty")

        try:
            item = Item.objects.create(
                name=data.name.strip(),
                category=data.category.strip() if data.category else None,
                created_by=self.user,
            )
            logger.info(f"Created item {item.id}: {item.name}")
            return item.id
        except Exception as e:
            logger.error(f"Failed to create item: {e}", exc_info=True)
            raise
```

### 1.3 Exception Registration (`SheetManage/exceptions.py`)

```python
import logging
from django.http import HttpRequest
from ninja import NinjaAPI
from sheetScript.services.item_service import ItemNotFoundError, ItemValidationError

logger = logging.getLogger(__name__)


def register_handlers(api: NinjaAPI) -> None:
    @api.exception_handler(ItemNotFoundError)
    def handle_item_not_found(request: HttpRequest, exc: ItemNotFoundError):
        logger.warning(f"Item not found: {exc}")
        return api.create_response(
            request,
            {"detail": f"Item not found: {exc}"},
            status=404,
        )

    @api.exception_handler(ItemValidationError)
    def handle_item_validation_error(request: HttpRequest, exc: ItemValidationError):
        logger.warning(f"Item validation error: {exc}")
        return api.create_response(
            request,
            {"detail": f"Validation error: {exc}"},
            status=400,
        )
```

---

## 2. Anti-Patterns to Avoid

### 2.1 Business Logic in API Layer

```python
# WRONG: Direct database access and business logic in API
@router.post("/items")
def create_item(request, data: CreateIn):
    if Item.objects.filter(name=data.name).exists():
        return 400, {"detail": "Item already exists"}
    if not is_valid_category(data.category):
        return 400, {"detail": "Invalid category"}
    item = Item.objects.create(name=data.name, category=data.category)
    return 201, {"id": item.id}
```

### 2.2 Manual Error Response Returns

```python
# WRONG: Returning error responses directly from API
@router.get("/items/{item_id}")
def get_item(request, item_id: int):
    try:
        item = Item.objects.get(id=item_id)
        return Reply(data=ItemOut.from_orm(item))
    except Item.DoesNotExist:
        return 404, {"detail": "Item not found"}  # WRONG
    except Exception as e:
        logger.error(f"Error: {e}")
        return 500, {"detail": "Internal server error"}  # WRONG
```

### 2.3 Incorrect Reply Usage with FileResponse

```python
# WRONG: Wrapping FileResponse in Reply
@router.get("/download/{file_id}", response={200: Reply})  # WRONG
def download_file(request, file_id: int):
    path = service.get_file_path(file_id)
    return Reply(data=FileResponse(open(path, "rb")))  # WRONG

# CORRECT
@router.get("/download/{file_id}")
def download_file(request, file_id: int):
    path = service.get_file_path(file_id)
    return FileResponse(open(path, "rb"), filename="document.pdf")
```

### 2.4 pydantic V1 Config Style

```python
# WRONG: class Config (pydantic V1)
class ItemOut(Schema):
    class Config:
        from_attributes = True

# CORRECT: model_config (pydantic V2)
class ItemOut(Schema):
    model_config = {"from_attributes": True}
```

### 2.5 Incorrect Type Hints

```python
# WRONG: Deprecated typing constructs
from typing import List, Dict, Optional

def process(items: List[Item]) -> Dict[str, int]:
    counts: Dict[str, int] = {}
    for item in items:
        category: Optional[str] = item.category

# CORRECT: Modern type hints
def process(items: list[Item]) -> dict[str, int]:
    counts: dict[str, int] = {}
    for item in items:
        category: str | None = item.category
```

---

## 3. Common Scenarios and Solutions

### Scenario 1: New API endpoint with database operations

1. Create service method for database operations
2. API endpoint calls service method
3. Service throws exceptions for errors
4. API returns `Reply` for success

### Scenario 2: File upload and processing

1. Service handles file I/O and processing
2. API validates upload and calls service
3. Service throws `FileProcessingError` for I/O failures
4. API returns appropriate response based on service result

### Scenario 3: Complex business workflow

1. Business layer implements specialized modules
2. Service layer orchestrates business modules
3. API layer handles request/response only
4. All errors thrown as exceptions

### Scenario 4: Multi-status-code batch operations

For batch operations that may partially succeed (e.g., batch import), use HTTP 207:

```python
# Service returns structured result
def batch_import(...) -> BatchImportSuccessResponse | BatchImportPartialResponse:
    ...

# API builds response based on result type
def _build_batch_import_reply(result):
    if result["failed"] == 0:
        return Reply(data=BatchImportSuccessOut(...))
    return 207, Reply(data=BatchImportPartialOut(...))
```

### Scenario 5: Pagination with filtering

```python
class ItemFilter(FilterSchema):
    name: str | None = Field(None, description="Filter by name")
    status: str | None = Field(None, description="Filter by status")

@wrap_reply()
@paginate(ReplyPagination)
def list_items(request, filters: ItemFilter = Query(...)):
    service = ItemService()
    qs = service.get_items()
    return filters.filter(qs)
```

---

## 4. Implementation Checklist

### Before Coding
- [ ] Query Context7 for Django 4.2 and Django Ninja 1.3 documentation
- [ ] Review existing patterns in similar modules
- [ ] Load `django-ninja-testing` skill for test patterns

### During Implementation
- [ ] Identify service orchestration logic and move to service layer
- [ ] Use `Reply` pattern for success responses (except FileResponse/HttpResponse)
- [ ] Throw exceptions for errors, don't return error responses
- [ ] Use modern type hints: `list[T]`, `dict[K, V]`, `T | None`
- [ ] Use pydantic V2 `model_config` instead of `class Config`
- [ ] Follow architecture layers: API → Service → Business

### After Implementation
- [ ] Register custom exceptions in `SheetManage/exceptions.py`
- [ ] Write tests for service methods and API endpoints
- [ ] Verify exception propagation through all layers
- [ ] Check response formats match project standards

### Code Review Checklist
- [ ] No business logic in API layer
- [ ] No manual error responses in API functions
- [ ] No deprecated type hints (List, Dict, Optional)
- [ ] No pydantic V1 `class Config`
- [ ] All exceptions registered in root API
- [ ] FileResponse/HttpResponse not wrapped in Reply
- [ ] Non-200 success codes manually specified (201, 207, etc.)
- [ ] Tests cover both success and error cases
