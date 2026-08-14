---
name: django-ninja-testing
description: Write and troubleshoot Django Ninja API endpoint tests. Use when the user asks to write tests for Django Ninja APIs, encounters TestClient ConfigError, needs to test authenticated endpoints, wants test patterns for ninja routers/operations, needs to diagnose a Django/Ninja runtime error, or wants to audit test coverage.
---

# Django Ninja API Testing & Diagnosis

> For architecture-layer rules (API/Service/Business separation, Reply/Response patterns, exception handling), see `AGENTS.md` §1–3.

---

## Quick start

```python
# tests/sheetScript/test_my_api.py
import pytest

pytestmark = pytest.mark.django_db

from ninja.testing import TestClient
from sheetScript.api.bearer import get_jwt_token
from sheetScript.models import NocAcc


def _get_client() -> TestClient:
    from .test_client import get_test_client
    return get_test_client()


def test_my_endpoint_success(test_user, auth_headers):
    client = _get_client()
    resp = client.get("/my-app/my-endpoint", headers=auth_headers)
    assert resp.status_code == 200
    assert resp.json()["info"] == "Success"


def test_my_endpoint_no_auth_401():
    client = _get_client()
    resp = client.get("/my-app/my-endpoint")
    assert resp.status_code == 401
```

Note:
- `pytestmark = pytest.mark.django_db` is required for any test that touches the database
- `test_user` and `auth_headers` are session-scoped fixtures from `conftest.py`
- NEVER call `django.setup()` or set `DJANGO_SETTINGS_MODULE` manually — pytest-django handles this
- NEVER create `TestClient(router)` directly; always use `get_test_client()` from `test_client.py`
- DB cleanup is handled automatically by pytest-django transaction rollback — do NOT add manual `.delete()` calls in `finally:` blocks

## TestClient Singleton Rule

Django Ninja's `TestClient` internally registers URL routes on construction. A second `TestClient(...)` call triggers:

```
ninja.errors.ConfigError: Looks like you created multiple NinjaAPIs or TestClients
```

This is a known framework limitation ([GitHub Issue #229](https://github.com/vitalik/django-ninja/issues/229)). The route registry uses global mutable state that can't be reset.

**Solutions** (pick one):

### A. Module-level lazy singleton (standalone scripts)

```python
_client: TestClient | None = None

def _get_client() -> TestClient:
    global _client
    if _client is None:
        _client = TestClient(main_api)
    return _client
```

### B. pytest session-scoped fixture (pytest projects)

```python
import pytest
from ninja.testing import TestClient
from SheetManage.api import api as main_api

@pytest.fixture(scope="session")
def api_client():
    return TestClient(main_api)

def test_list(api_client):
    resp = api_client.get("/audit/tasks/")
    assert resp.status_code == 200
```

### C. Django TestCase setUpClass (unittest-style)

```python
from django.test import TestCase
from ninja.testing import TestClient
from SheetManage.api import api as main_api

class AuditAPITests(TestCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.client = TestClient(main_api)

    def test_list(self):
        resp = self.client.get("/audit/tasks/")
        assert resp.status_code == 200
```

**Avoid**: Never create `TestClient(router)` in individual test functions or `setUp()`. Never mix creating `TestClient(main_api)` and `TestClient(sub_router)` in the same process.

## Route path conventions

Use `TestClient(main_api)` with **full paths**, not `TestClient(sub_router)` with short paths:

```python
# Prefer: full path on main_api
client = TestClient(main_api)
client.get("/audit/comparison/tasks")       # NOT "/tasks"
client.post("/audit/tasks/{uuid}/apply")    # NOT "/apply"

# Avoid: sub_router with short paths — may cause routing/auth mismatches
client = TestClient(comparison_subrouter)
client.get("/tasks")  # fragile
```

> **Always verify route paths match the API definition.** Mismatches (e.g. `/task` vs `/tasks`) produce `Exception: Cannot resolve "..."` from Ninja's TestClient, which looks like a framework error but is usually a typo in the test.

## Testing authenticated endpoints

```python
def test_auth_endpoint(auth_headers):
    client = _get_client()
    resp = client.get("/protected/endpoint", headers=auth_headers)
    assert resp.status_code == 200
```

The `auth_headers` fixture (from `conftest.py`) provides a valid JWT for the admin test user. If you need a regular user, use `NocAcc.objects.get(sps_acc="test_regular_user")` (future).

## Testing non-200 status codes

```python
def test_concurrent_409():
    client = _get_client()
    resp = client.post("/audit/comparison/task", ...)
    assert resp.status_code == 409
    assert resp.json().get("error") == "DUPLICATE_TASK"

def test_invalid_token_401():
    client = _get_client()
    resp = client.get("/protected/", headers={"Authorization": "Bearer bad.token"})
    assert resp.status_code == 401
```

## Testing file uploads

```python
from django.core.files.uploadedfile import SimpleUploadedFile

def test_file_upload():
    file = SimpleUploadedFile("test.xlsx", b"...", content_type="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")
    client = _get_client()
    resp = client.post("/upload", FILES={"files": file}, headers=_auth_headers(user))
    assert resp.status_code == 201
```

## Seeding legacy dirty rows: raw SQL insert fixture

Model-level validation (Manager/QuerySet/save overrides, custom field `get_prep_value`) **rejects or silently normalizes** rows that still exist in production from before validation was added. Such legacy dirty rows cannot be created through the ORM:

- 「占用无 an」rows — strict-mode model validation rejects them, `objects.create()` raises
- Host-bits-nonzero subnets (`10.0.0.5/24`) — `CidrField.get_prep_value` normalizes to `10.0.0.0/24`, so the dirty value never reaches the DB via the ORM

**Use the shared conftest fixture `insert_raw_lans_ip`** (`tests/sheetScript/conftest.py`) instead of copying raw SQL into each test file. It inserts via `connection.cursor()` (bypassing model validation) and accepts keyword params including `state`, `gateway`, and `an`:

```python
@pytest.mark.integration
def test_contains_finds_dirty_row(self, insert_raw_lans_ip):
    insert_raw_lans_ip(
        ip_id=990002006,
        bas_ip="10.99.60.9",
        ip_subnet="10.99.85.5/24",  # host bits nonzero — stays dirty
        ip="10.99.85.10",
        # state="空闲" (default), gateway="10.88.0.1" (default), an=None (default)
    )
    assert LansIp.objects.filter(ip_subnet__contains="10.99.85.5/24").exists()
```

Rules:
- Raw inserts still run inside the pytest-django per-test transaction and roll back — no manual cleanup
- Mark these tests `@pytest.mark.integration` (real DB row, per AGENTS.md §6.1 white-list)
- Do NOT hand-roll per-file raw SQL — request the shared fixture so bypass points stay in one place

## Compile-time SQL parameter assertions (no DB)

When the question is "does field X / lookup Y normalize its RHS before hitting the DB", assert on **compiled SQL params** rather than on query results. `qs.query.sql_with_params()` returns `(sql_string, params_tuple)` at compile time without touching the database — no `django_db` mark, no seeded rows:

```python
def test_pattern_lookup_params_stay_raw():
    # Pattern lookups (contains/icontains/startswith/endswith) are
    # PatternLookup(prepare_rhs=False): the RHS passes through RAW
    cases = {
        "contains": ("10.0.0.5/24", "%10.0.0.5/24%"),
        "startswith": ("10.0.0.5/2", "10.0.0.5/2%"),
    }
    for lookup, (value, expected) in cases.items():
        _, params = LansIp.objects.filter(
            **{f"ip_subnet__{lookup}": value}
        ).query.sql_with_params()
        assert params[0] == expected, lookup

    # Control group: exact / in DO go through get_prep_value normalization
    _, params = LansIp.objects.filter(ip_subnet="10.0.0.5/24").query.sql_with_params()
    assert params[0] == "10.0.0.0/24"
```

Why not the alternatives:
- `str(qs.query)` is unreliable — params may be inlined or quoted differently and are not stable for assertions (this approach failed in practice before switching to `sql_with_params()`)
- DB-level assertions (seed a row, filter, check hit) require real data and conflate "param value" with "matching semantics"

Use this to lock ORM semantics (field prep, lookup classes) into regression tests, and to **verify ORM claims by compiling SQL before accepting review findings** about normalization behavior.

## Running tests: MUST use pytest

**所有测试必须通过 pytest 运行，禁止直接 `python test_xxx.py`。**

```bash
# 正确 — 事务隔离，自动回滚，完整报告
python -m pytest tests/sheetScript/test_my_api.py

# 错误 — 无事务保护，数据泄露，错误信息被吞
python tests/sheetScript/test_my_api.py
```

### 为什么禁止直接运行

| 问题 | 后果 |
|------|------|
| 没有事务回滚 | 测试数据泄漏到数据库，影响后续运行 |
| 没有测试隔离 | 一个测试的状态污染下一个测试 |
| 手动 `_assert()` 吞错误 | 打印 "FAIL" 但不传播，退出码始终为 0 |
| 手动 `django.setup()` | 与 pytest-django 的自动初始化冲突 |
| 异常导致后续测试跳过 | 第一个断言失败后所有剩余测试不执行 |

使用 `python -m pytest`：

- **事务回滚**：每个测试在独立事务中运行，结束后自动 ROLLBACK
- **测试隔离**：一个测试的失败不影响其他测试
- **完整报告**：`X passed, Y failed` + 完整的 traceback
- **自动初始化**：pytest-django 自动设置 `DJANGO_SETTINGS_MODULE` 和 Django 环境
- **CI 兼容**：失败时退出码非 0

> **所有测试文件必须是 pytest 风格**：使用纯函数、`pytest` fixture、`assert` 语句。禁止使用 `if __name__ == "__main__"` 作为测试入口。禁止在测试文件中手动调用 `django.setup()` 或设置 `DJANGO_SETTINGS_MODULE`。
>
> 测试数据的清理由 pytest-django 的事务回滚自动处理，不要手动添加 `.delete()` 清理代码。

## Common pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Running test file directly (`python test.py`) | Looks like tests pass but exceptions are hidden; later tests silently skipped | Always use `python -m pytest` |
| Multiple TestClient instances | `ConfigError: Looks like you created multiple NinjaAPIs or TestClients` | Use singleton/fixture |
| Sub-router paths without prefix | 404 for paths that work in browser | Use `TestClient(main_api)` with full path |
| Missing Content-Type for uploads | 415 Unsupported Media Type | Use `format="multipart"` as kwarg, not header |
| Auth headers not passed | 401 on every request | Verify `Authorization: Bearer <token>` header set |
| Test DB not isolated | Test data leaks between runs | Use `pytest.mark.django_db` — pytest-django automaically rolls back each test |
| Pytest fixture scope="function" | ConfigError because fixture re-created per test | Use `scope="session"` or `scope="module"` |
| Seeding legacy dirty rows via `objects.create()` | "occupied without an" raises validation; dirty subnet silently normalized | Use shared conftest fixture `insert_raw_lans_ip` (raw SQL) + `@pytest.mark.integration` |
| Asserting query params via `str(qs.query)` | Params inlined/unstable, assertions mislead | Use `qs.query.sql_with_params()` and assert on the params tuple |

---

## Diagnose → Fix → Verify Loop

When the user pastes an error (curl request + stack trace, or a runtime exception), follow this disciplined loop:

```
Reproduce → Minimise → Hypothesise → Instrument → Fix → Regression-test
```

### 1. Reproduce

- Read the exact error trace, note the **file, line, exception type**.
- If the user provides a curl request, attempt to reproduce with the same payload.
- Query Context7 for Django 4.2 and Django Ninja 1.3 documentation before proposing a fix.

### 2. Minimise

- Strip the repro to the smallest failing unit (one endpoint, one query, one schema).
- Check if the error is in **test code** vs **production code** vs **framework**.

### 3. Hypothesise

Match the error against the **Known Error Pattern Database** below. If it matches, explain the root cause and apply the known fix pattern.

### 4. Instrument

- Add targeted logging or `print()` in the failing code path.
- Verify assumptions about data types, query results, and state.

### 5. Fix

- Make the **minimal** change that resolves the error.
- Follow architecture layers (fix in the correct layer: API/Service/Business).
- If the fix changes exception handling, verify the exception handler is registered in `SheetManage/api.py`.

### 6. Regression-test

- Write or update the test that would have caught this bug.
- Run `python -m pytest` on the affected test file.
- Confirm both **success** and **error** paths pass.

---

## Known Error Pattern Database

These are recurring failure modes observed in the `data-support-platform` codebase.

### Pattern 1: `Cannot resolve keyword 'task_uuid'` (Django ORM FK field name mismatch)

**Symptom:**
```
FieldError: Cannot resolve keyword 'task_uuid' into field.
```

**Root cause:** The model defines a `ForeignKey` named `task`, but code tries to filter by `task_uuid`. Django ORM uses the **field name** (`task`), not the database column name (`task_uuid`).

**Fix:**
```python
# WRONG
ComparisonInconsistentRows.objects.filter(task_uuid=uuid)

# CORRECT
ComparisonInconsistentRows.objects.filter(task__uuid=uuid)
```

**Regression test:**
```python
def test_query_by_task_uuid():
    client = _get_client()
    resp = client.get(f"/audit/comparison/tasks/{task.uuid}/inconsistencies")
    assert resp.status_code == 200
```

### Pattern 2: `select_for_update` outside transaction

**Symptom:**
```
TransactionManagementError: select_for_update cannot be used outside of a transaction.
```

**Root cause:** `QuerySet.select_for_update()` requires an active `transaction.atomic()` block.

**Fix:**
```python
from django.db import transaction

def acquire_task(task_id: int):
    with transaction.atomic():
        return AuditTask.objects.select_for_update().get(id=task_id)
```

**Regression test:**
```python
def test_concurrent_task_lock():
    client = _get_client()
    # Verify 409 or proper serialization on concurrent access
```

### Pattern 3: MySQL JSON NaN serialization

**Symptom:**
```
JSON format invalid; NaN is not allowed in JSON
```

**Root cause:** `pandas` DataFrames may contain `NaN`/`Inf`, which MySQL's JSON column rejects.

**Fix:**
```python
import math

def _sanitize_json_value(value):
    if isinstance(value, float) and (math.isnan(value) or math.isinf(value)):
        return None
    return value

def df_to_json_safe(df: pd.DataFrame) -> dict:
    return df.applymap(_sanitize_json_value).to_dict(orient="records")
```

**Regression test:**
```python
def test_df_with_nan_serialization():
    df = pd.DataFrame({"a": [1.0, float("nan"), 3.0]})
    result = df_to_json_safe(df)
    assert result[1]["a"] is None
```

### Pattern 4: pydantic V2 `class-based config` deprecation

**Symptom:**
```
PydanticDeprecatedSince20: Support for class-based `config` is deprecated...
```

**Root cause:** pydantic V2 no longer supports `class Config` inside Schema classes.

**Fix:**
```python
# WRONG (pydantic V1 style)
class MyOut(Schema):
    class Config:
        from_attributes = True

# CORRECT (pydantic V2 style)
class MyOut(Schema):
    model_config = {"from_attributes": True}
```

**Regression test:**
```python
def test_schema_no_deprecation():
    with warnings.catch_warnings(record=True) as w:
        warnings.simplefilter("always")
        MyOut.model_validate({"id": 1})
        assert not any("class-based config" in str(x.message) for x in w)
```

### Pattern 5: `prefetch_related` on non-existent relation

**Symptom:**
```
AttributeError: 'ComparisonInconsistentRows' object has no attribute 'task'
```

**Root cause:** `prefetch_related("task")` was used but the model uses `select_related` (FK) or the related name is different.

**Fix:**
- Use `select_related("task")` for ForeignKey, `prefetch_related("items")` for reverse Many-to-Many/One-to-Many.
- Verify the related name in the model definition.

**Regression test:**
```python
def test_endpoint_with_prefetch():
    client = _get_client()
    resp = client.get("/audit/comparison/tasks/", headers=_auth_headers(user))
    assert resp.status_code == 200
```

---

## Exception Handler Chain Validation

When an error occurs in production but returns **500** instead of the expected status code (e.g., 404 or 422), follow this checklist:

1. **Service layer throws the correct exception?**
   ```python
   # Verify the exception is raised, not swallowed
   raise ItemNotFoundError(f"Item {item_id} not found") from e
   ```

2. **Exception registered in `SheetManage/api.py`?**
   ```python
   @api.exception_handler(ItemNotFoundError)
   def handle_item_not_found(request, exc):
       return api.create_response(request, {"detail": str(exc)}, status=404)
   ```

3. **Test verifies the correct status code?**
   ```python
   def test_not_found_returns_404():
       client = _get_client()
       resp = client.get("/items/99999")
       assert resp.status_code == 404  # NOT 500
   ```

4. **No generic `except Exception` swallowing the custom exception?**
   ```python
   # WRONG — swallows custom exceptions, causing 500
   try:
       return service.get_item(item_id)
   except Exception:
       logger.error(...)
       raise  # re-raise is OK, but bare `except Exception` is risky
   ```

---

## Brownfield Test Coverage Audit

When the user says "检查...的测试覆盖率" or "添加...接口的API层集成测试", follow this workflow:

### Step 1: Scan

Read the target module(s) and identify:
- All public API endpoints (router operations)
- All public service methods
- All exception paths (custom exceptions)

### Step 2: Matrix

Produce a coverage matrix:

| Endpoint/Method | Has Test? | Test Type | Gaps |
|-----------------|-----------|-----------|------|
| `GET /tasks/` | ❌ | — | success + auth + pagination |
| `POST /tasks/` | ✅ | API integration | missing 409 duplicate |
| `AuditService.import_excel()` | ❌ | — | success + NaN data + bad file |

### Step 3: Prioritize

Label each gap:
- **P0** (must have): Happy path + critical error paths (auth, 404, 409)
- **P1** (should have): Edge cases (empty input, pagination limits)
- **P2** (nice to have): Rare error conditions, concurrent access

### Step 4: Implement

Generate tests following the patterns below. Reference `tests/sheetScript/test_lans_api.py` or `tests/sheetScript/test_audit_api.py` for style.

### Step 5: Verify

Run `python -m pytest` and confirm all new tests pass.

---

## Test Structure & Requirements

### Structure (Arrange-Act-Assert)

```python
import pytest

pytestmark = pytest.mark.django_db

from sheetScript.services.item_service import ItemService, ItemNotFoundError


class TestItemService:
    """Service layer tests using pytest style."""

    def test_get_item_success(self, test_user):
        """Test successful item retrieval."""
        # Arrange
        service = ItemService(user=test_user)
        item = Item.objects.create(name="Test Item")

        # Act
        result = service.get_item(item.id)

        # Assert
        assert result.id == item.id
        assert result.name == "Test Item"

    def test_get_item_not_found(self, test_user):
        """Test item not found raises exception."""
        service = ItemService(user=test_user)
        with pytest.raises(ItemNotFoundError):
            service.get_item(999)  # Non-existent ID

    def test_create_item_with_valid_data(self, test_user):
        """Test item creation with valid data."""
        service = ItemService(user=test_user)
        item_id = service.create_item(name="New Item", category="Test")

        assert isinstance(item_id, int)
        assert Item.objects.filter(id=item_id).exists()
```

Note:
- Use `pytest.raises(...)` instead of `self.assertRaises(...)`
- Use `assert` instead of `self.assertEqual(...)`
- Use `test_user` fixture for authenticated service instances
- Service tests that mock the DB layer can omit `pytestmark`

### Requirements

1. **Service layer**: Write unit tests for all service methods
2. **API endpoints**: Write integration tests for critical endpoints
3. **Error cases**: Test both success and error scenarios
4. **Authentication**: Test permission and authentication when applicable
5. **Data validation**: Test input validation and error handling

### Scenario: Testing a new service method

1. Reference `test_lans_api.py` or `test_audit_service.py` patterns
2. Test success cases with valid data
3. Test error cases with invalid data
4. Test exception propagation
5. Mock external dependencies when needed
