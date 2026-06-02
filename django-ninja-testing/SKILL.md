---
name: django-ninja-testing
description: Write and troubleshoot Django Ninja API endpoint tests. Use when the user asks to write tests for Django Ninja APIs, encounters TestClient ConfigError, needs to test authenticated endpoints, or wants test patterns for ninja routers/operations.
---

# Django Ninja API Testing

## Quick start

```python
# tests/sheetScript/test_my_api.py
import os, sys
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "SheetManage.settings")
sys.path.insert(0, ".")
import django; django.setup()

from ninja.testing import TestClient
from SheetManage.api import api as main_api

# CRITICAL: TestClient is a process-wide singleton.
# Creating it more than once raises
#   ninja.errors.ConfigError: Looks like you created multiple NinjaAPIs or TestClients

_client: TestClient | None = None

def _get_client() -> TestClient:
    global _client
    if _client is None:
        _client = TestClient(main_api)
    return _client


def test_my_endpoint_success():
    client = _get_client()
    resp = client.get("/my-app/my-endpoint", headers=_auth_headers(user))
    assert resp.status_code == 200
    assert resp.json()["info"] == "Success"

def test_my_endpoint_no_auth_401():
    client = _get_client()
    resp = client.get("/my-app/my-endpoint")
    assert resp.status_code == 401
```

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
client.get("/audit/comparison/task")       # NOT "/task"
client.post("/audit/tasks/{uuid}/apply")    # NOT "/apply"

# Avoid: sub_router with short paths — may cause routing/auth mismatches
client = TestClient(comparison_subrouter)
client.get("/task")  # fragile
```

## Testing authenticated endpoints

```python
import jwt, time
from django.conf import settings

def _make_jwt(user: NocAcc) -> str:
    payload = {
        "acc": user.sps_acc,
        "area": user.area,
        "username": user.username,
        "iat": int(time.time()),
        "exp": int(time.time()) + 3600 * 24,
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")

def _auth_headers(user: NocAcc) -> dict[str, str]:
    return {"Authorization": f"Bearer {_make_jwt(user)}"}

def test_auth_endpoint():
    user = _ensure_test_user()
    client = _get_client()
    resp = client.get("/protected/endpoint", headers=_auth_headers(user))
    assert resp.status_code == 200
```

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

## Common pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Multiple TestClient instances | `ConfigError: Looks like you created multiple NinjaAPIs or TestClients` | Use singleton/fixture |
| Sub-router paths without prefix | 404 for paths that work in browser | Use `TestClient(main_api)` with full path |
| Missing Content-Type for uploads | 415 Unsupported Media Type | Use `format="multipart"` as kwarg, not header |
| Auth headers not passed | 401 on every request | Verify `Authorization: Bearer <token>` header set |
| Test DB not isolated | Test data leaks between runs | Wrap in `transaction.atomic()` or clean up in `finally` |
| Pytest fixture scope="function" | ConfigError because fixture re-created per test | Use `scope="session"` or `scope="module"` |
