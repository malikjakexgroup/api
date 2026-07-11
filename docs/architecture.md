# Architecture

```
        book-web (React)
              │  fetch /wp-json/books/v1/*
              ▼
   ┌──────────────────────────┐
   │  WordPress + book-backend │
   │                          │
   │  REST route  ─▶ Transient cache (24h)   ◀─ Redis if object cache installed
   │       │  miss                            │
   │       ▼                                  │
   │  book CPT (MySQL)  ◀── upsert ──┐        │
   └───────────────────────────────┼─────────┘
              │ miss / stale        │
              ▼                     │
        Google Books API ──────────┘
```

WordPress collapses what used to be separate services: the REST API, cache, storage,
and admin all live in one plugin + the WordPress core.

| Concern | Provided by |
|---------|-------------|
| Public API | `register_rest_route` → `/wp-json/books/v1/*` |
| Cache | Transients API (24h TTL) |
| Storage | `book` Custom Post Type in MySQL |
| Admin | wp-admin → Books |

## Request flow — `/wp-json/books/v1/search?q=`

1. Check transient `bb_search_{md5(q)}`. **Hit → return.**
2. Miss → `wp_remote_get` to Google Books.
3. Normalize each item and upsert it into the `book` CPT (found by `google_id` meta).
4. Cache the result list in a transient (24h) and return it.

`/book/{id}` is the same, except the CPT is served directly when the stored row is
fresh (within `BB_CACHE_TTL`) before falling back to Google.

## Resilience — when Google is down

- Repeat query still in a transient → served from cache.
- Book already stored as a CPT → served (stale-but-present).
- Never stored anywhere → **HTTP 503**. Never fabricated data.

## Deferred

Search-log table, auth / API keys / rate limiting. wp-admin already covers browsing
stored books.
