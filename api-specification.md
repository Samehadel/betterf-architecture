# API Conventions

> API response format and authentication are defined in [Backend Architecture](./backend-architecture.md). This document captures the shared HTTP conventions used across endpoints. It is a reference document, not a generated-contract source of truth.

---

## Base URL & Content Type

- **Base URL**: `/api/v1`
- **Content-Type**: `application/json`

---

## Endpoint Conventions

All endpoints follow standard REST conventions:

| Method | Pattern | Returns | Description |
|---|---|---|---|
| `POST` | `/api/v1/{resource}` | `201 Created` | Create a new resource |
| `GET` | `/api/v1/{resource}` | `200 OK` | List with pagination |
| `GET` | `/api/v1/{resource}/{id}` | `200 OK` | Fetch a single resource |
| `PUT` | `/api/v1/{resource}/{id}` | `200 OK` | Replace a resource |
| `PATCH` | `/api/v1/{resource}/{id}` | `200 OK` | Partial update |
| `DELETE` | `/api/v1/{resource}/{id}` | `204 No Content` | Delete a resource |

---

## Pagination

List endpoints accept standard pagination query parameters:

```
GET /api/v1/{resource}?page=0&size=20&sort=created_at,desc
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | integer | `0` | Page number (0-indexed) |
| `size` | integer | `20` | Items per page (max 100) |
| `sort` | string | — | Field and direction, e.g. `created_at,desc` |

Paginated responses follow this shape inside `ApiResponse.data`:

```json
{
  "content": [...],
  "page_number": 0,
  "page_size": 20,
  "total_elements": 150,
  "total_pages": 8,
  "is_last": false
}
```

---

## Filtering

Endpoints that support filtering accept them as query parameters:

```
GET /api/v1/{resource}?status=ACTIVE&created_after=2026-01-01
```

Filter parameters are endpoint-specific and always documented in the corresponding YAML file.

---

## Application Error Codes

The `code` field in error responses is a machine-readable string used for client-side handling. Every domain module defines its own codes. Common system-level categories include:

| Error Code | HTTP Status | Description |
|---|---|---|
| `VAL_*` | 400 | Request body or parameter failed validation |
| `AUTH_*` | 401 / 403 | Missing token, invalid token, or insufficient permission |
| `RES_*` | 404 / 409 | Missing resource or conflicting resource state |
| `SYS_*` | 500 | Unexpected system or infrastructure failure |

The authoritative error-code catalog lives in [backend-guides/exception-handling/exception-handling.md](backend/exception-handling/exception-handling.md). Keep this file aligned with those conventions, not with invented field names.

---

## Live Documentation

| Interface | URL |
|---|---|
| Swagger UI | `http://localhost:8080/swagger-ui/` |
| OpenAPI JSON | `http://localhost:8080/v3/api-docs` |
| Redoc | `http://localhost:8080/redoc.html` |
