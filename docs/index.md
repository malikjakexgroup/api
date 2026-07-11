# Book Platform

A Google Books proxy built with a **WordPress** backend and a **React** frontend,
demonstrating caching, durable storage, and a clean API boundary.

## Repositories

| Repo | Role |
|------|------|
| `book-backend` | WordPress plugin. REST API (`/wp-json/books/v1/*`), Google Books fetch, Transient cache, `book` Custom Post Type storage. wp-admin is the admin panel. |
| `book-web` | React frontend. Calls only the WordPress REST API. |
| `book-docs` | This site. |

See [Architecture](architecture.md) for the request flow and the failure policy.
