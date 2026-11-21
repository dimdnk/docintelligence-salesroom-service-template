## 📜 API Documentation

Base path: `/api/v1/pitch`

All endpoints require a valid RS256 JWT issued by the IAM service unless marked as public.
The JWT must be passed as a `Bearer` token in the `Authorization` header.

---

### Ping (Reachability Probes)

| Method | Path           | Auth                 | Description                                                           |
| ------ | -------------- | -------------------- | --------------------------------------------------------------------- |
| `GET`  | `/ping`        | public               | Unauthenticated reachability probe — no JWT or tenant header required |
| `GET`  | `/tenant/ping` | JWT + `X-Tenant-ID`  | Tenant-scoped probe — echoes resolved tenant key                      |
| `GET`  | `/admin/ping`  | JWT `PLATFORM_ADMIN` | Admin probe — echoes caller's `user_id` claim                         |

`GET /ping` is suitable for infrastructure health checks and smoke tests that run before auth is available.

`GET /tenant/ping` verifies that tenant resolution (from `X-Tenant-ID` header or JWT `tenant_id` claim) is working correctly.

`GET /admin/ping` confirms the caller's JWT is decoded correctly and carries the `PLATFORM_ADMIN` authority.

---

### Tenant & Authority Context

Two conventions apply to every non-public path:

**Tenant context.** The tenant key is resolved before the request is handled, from the `X-Tenant-ID` header (priority 1) or the JWT `tenant_id` claim (priority 2). When neither yields a value, the request is rejected with `400` and an RFC 7807 body — it never reaches a handler. Paths under `/admin/` are exempt: platform administrators operate across tenants, so each admin handler scopes its own request.

**Authorities.** The JWT `authorities` claim drives access control. Tenant callers carry `MEMBER`, `TENANT_OWNER` or `ADMIN`; platform administrators carry `PLATFORM_ADMIN`, which every `/admin/**` path requires. A caller without the required authority receives `403`.

---

### Domain Surfaces

No domain endpoints are implemented yet — `ping` is the only shipped surface. New surfaces arrive as feature slices, each mirroring the public / tenant / admin tiers above. The slices the Digital Sales Room brief calls for:

- **Venue selection** — search the venue library in the shared catalog database and add venues to a pitch
- **Pitch generation** — one click renders the selected venues as a branded, interactive client page
- **Share links** — client-scoped links that open on any device with no account
- **Client engagement** — photo and floor-plan browsing, plus per-venue comments and Q&A answered in context
- **Approval** — client approval locks every field, file, and decision into an immutable, timestamped record

---

### Error Responses

All errors follow RFC 7807 `ProblemDetail` format:

```json
{
    "type": "about:blank",
    "title": "Validation Failed",
    "status": 400,
    "detail": "Request validation failed",
    "instance": "/api/v1/pitch/ping",
    "correlationId": "abc12345",
    "requestId": "req-a1b2c3d4"
}
```

| Status | When                                                       |
| ------ | ---------------------------------------------------------- |
| 400    | Validation failure, or a tenant ID that cannot be resolved |
| 401    | Missing or invalid JWT                                     |
| 403    | Insufficient authority (e.g. accessing admin endpoints)    |
| 404    | Referenced resource not found                              |
| 500    | Unexpected internal error                                  |

---

### Interactive Documentation

Swagger UI is available at `http://localhost:8080/swagger-ui.html` when running locally.
The Gateway aggregates the Pitch Service spec at `/api/v1/pitch/api-docs`.

> Auth legend: `public` = no token required; `JWT` = valid Bearer token; `JWT + X-Tenant-ID` = valid Bearer token plus a resolvable tenant; `JWT PLATFORM_ADMIN` = JWT with platform administrator authority.
