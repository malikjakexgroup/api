# API

The WordPress backend exposes the public API under `/wp-json/books/v1/` (default host
`http://localhost:8080`). WordPress also serves a route index at `/wp-json/`.

> 👉 Prefer an interactive view? See the **[Swagger API Reference](swagger.md)** —
> expand each endpoint, see schemas, and try live requests. Spec: [`openapi.yaml`](openapi.yaml).

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/wp-json/books/v1/search?q=` | Search books. |
| GET | `/wp-json/books/v1/book/{google_id}` | Book by Google volume id. |
| GET | `/wp-json/books/v1/categories` | Static category list. |
| POST | `/wp-json/books/v1/register` | Create account — `{name, email, password}` → `{token, user}`. |
| POST | `/wp-json/books/v1/login` | Log in — `{email, password}` → `{token, user}`. |
| GET | `/wp-json/books/v1/me` | Current user. Send `Authorization: Bearer <token>`. |

## Auth

Token-based. Register or login returns a `token`; send it as `Authorization: Bearer
<token>` on later requests. The token's hash is stored in WordPress user meta.

## Errors

- `400` — missing `q`.
- `404` — book not found (and not previously stored).
- `503` — Google Books unavailable and nothing cached. **Never fabricated data.**

## Admin

There is no separate admin app. Browse and edit stored books at **wp-admin → Books**.
