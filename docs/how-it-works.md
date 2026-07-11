# How It Works — Function by Function

This page explains the platform end to end: every component, every important
function, and the exact path a request takes.

---

## 1. The big picture

```
┌──────────────┐   HTTPS    ┌────────────────────────────┐   HTTPS   ┌───────────────┐
│  book-web    │ ─────────▶ │  WordPress + book-backend  │ ────────▶ │ Google Books  │
│  (React SPA) │            │  (REST API + storage)      │           │     API       │
│              │ ◀───────── │                            │ ◀──────── │               │
└──────────────┘   JSON     └────────────┬───────────────┘   JSON    └───────────────┘
                                          │
                                   ┌──────┴───────┐
                                   │   Database   │  (books stored permanently)
                                   │ MySQL/SQLite │
                                   └──────────────┘
```

- **book-web** never talks to Google directly — only to the backend.
- **book-backend** is the single source of truth: it fetches, normalizes, stores,
  and serves. Google is contacted **only for brand-new searches**.
- **Storage is permanent**: once a book is saved, it is served from the database
  forever (store-first).

---

## 2. Backend — `book-backend` (WordPress plugin)

The plugin is plain, prefixed PHP (`bb_`) split by responsibility.

### 2.1 `book-backend.php` — bootstrap
| Function / hook | What it does |
|-----------------|--------------|
| plugin header | Registers the plugin with WordPress; defines `BB_CACHE_TTL`. |
| `require_once …` | Loads the four include files. |
| `add_action('init', 'bb_register_book_cpt')` | Registers the `book` post type on every load. |
| `add_action('rest_api_init', 'bb_register_routes')` | Registers the public book endpoints. |
| `add_action('rest_api_init', 'bb_register_auth_routes')` | Registers register/login/me. |
| CORS filter | Sends `Access-Control-Allow-*` headers so the React app (different origin) can call the API. |

### 2.2 `google-books.php` — the external data source
| Function | Input → Output | Role |
|----------|----------------|------|
| `bb_google_url($path, $args)` | path + params → URL | Builds the Google Books URL, appends `BB_GOOGLE_API_KEY` if defined. |
| `bb_normalize($item)` | Google volume → our fields | Maps Google's nested `volumeInfo` into our flat book shape (title, authors, thumbnail, pages…). Safe on missing fields. |
| `bb_google_search($q)` | query → array \| null | Calls Google; returns normalized books, or `null` if Google is unavailable/rate-limited. |
| `bb_google_get($id)` | volume id → book \| 'not_found' \| null | Fetches one volume; distinguishes "missing" (404) from "unavailable". |

### 2.3 `store.php` — permanent storage (Custom Post Type)
| Function | Role |
|----------|------|
| `bb_register_book_cpt()` | Declares the `book` post type so wp-admin lists/edits books for free. |
| `bb_find_book_post($google_id)` | Looks up an existing book by its `google_id` meta. Returns the post id or 0. |
| `bb_upsert_book($data)` | Insert-or-update: creates/updates the `book` post and its meta (authors, thumbnail, rating…). The **single writer** of book data. |
| `bb_search_stored($q)` | Full-text search across stored books — the permanent, no-API search path. |
| `bb_book_to_array($post_id)` | Builds the public JSON shape from a stored post. |
| `bb_is_fresh($post_id)` | Freshness helper (kept for reference; store-first no longer expires). |

### 2.4 `cache.php` — fast repeat responses
| Function | Role |
|----------|------|
| `bb_cache_key($kind, $val)` | Deterministic cache key (`bb_search_{hash}`). |
| `bb_cache_get($key)` / `bb_cache_set($key, $v)` | Read/write WordPress transients (Redis-backed if an object cache exists). |

### 2.5 `auth.php` — token authentication
| Function | Role |
|----------|------|
| `bb_issue_token($user_id)` | Generates a random token, stores its **SHA-256 hash** in user meta, returns the raw token. |
| `bb_user_by_token($token)` | Resolves a bearer token back to a `WP_User`. |
| `bb_bearer_token($req)` | Extracts the token from the `Authorization: Bearer …` header. |
| `bb_user_public($user)` | Shapes a safe user object (id, name, email — never the password). |
| `bb_rest_register($req)` | Validates input, creates a real WordPress user, issues a token. |
| `bb_rest_login($req)` | Verifies credentials via `wp_authenticate`, issues a token. |
| `bb_rest_me($req)` | Returns the current user for a valid token, else `401`. |

### 2.6 `rest.php` — the API surface
| Route | Handler | Behavior |
|-------|---------|----------|
| `GET /wp-json/books/v1/search?q=` | `bb_rest_search` | **Store-first**: return saved matches; else fetch Google once, store, return; `503` if nothing found and Google is down. |
| `GET /wp-json/books/v1/book/{id}` | `bb_rest_get_book` | Saved book if present (permanent); else fetch once, store, return; `404`/`503` otherwise. |
| `GET /wp-json/books/v1/categories` | `bb_rest_categories` | Static category list. |

---

## 3. Frontend — `book-web` (React)

### 3.1 `services/api.js` — the only network layer
| Function | Calls |
|----------|-------|
| `searchBooks(q)` | `GET /search?q=` |
| `getBook(id)` | `GET /book/{id}` |
| `registerUser` / `loginUser` / `fetchMe` | auth endpoints |
| `authHeaders()` | Attaches `Authorization: Bearer <token>` from `localStorage`. |

### 3.2 Hooks — data + state
| Hook | Role |
|------|------|
| `useSearch(q)` | TanStack Query wrapper around `searchBooks` — caching, loading, error states. |
| `useBook(id)` | Same for a single book. |
| `useAuth()` | Auth context: `user`, `login`, `register`, `logout`; restores session from the saved token on load. |
| `useFavorites()` | Favorites in `localStorage`, synced across tabs. |

### 3.3 Pages & components
| Piece | Role |
|-------|------|
| `Search` | Hero, search bar, suggestion chips, result grid, skeletons/empty states. |
| `BookDetails` | Cover, metadata, category tags, description, favorite button. |
| `Favorites` | Grid of saved books. |
| `Login` / `Signup` | Auth forms → `useAuth`. |
| `BookCard` / `BookCover` / `Stars` / `SearchBar` | Reusable UI. |
| `App` | Layout, header/nav, routes, logged-in state. |

---

## 4. Request flows

### 4.1 Search
```
User types "atomic habits"
  → useSearch → searchBooks("atomic habits")
  → GET /wp-json/books/v1/search?q=atomic+habits
     → bb_rest_search
        → bb_search_stored()  ── found? ──▶ return saved books        (permanent, no Google)
                              └─ empty? ──▶ bb_google_search()
                                              ├─ ok  → bb_upsert_book() each → return
                                              └─ null → 503 (nothing to serve)
  → React renders BookCards
```

### 4.2 Open a book
```
Click card → getBook(id) → GET /book/{id}
  → bb_rest_get_book
     → bb_find_book_post()  ── exists? ──▶ return saved (permanent)
                            └─ new?    ──▶ bb_google_get() → bb_upsert_book() → return
```

### 4.3 Sign up / log in
```
Signup form → registerUser({name,email,password})
  → POST /register → bb_rest_register
     → wp_create_user() → bb_issue_token() → { token, user }
  → useAuth saves token in localStorage → header shows "Hi, <name>"

Later requests → authHeaders() → Authorization: Bearer <token>
  → bb_rest_me() resolves token → current user
```

---

## 5. Data model

**`book` (Custom Post Type)** — `post_title` = title, `post_content` = description, plus meta:
`google_id` (unique), `authors`, `publisher`, `thumbnail`, `language`, `pages`, `categories`, `rating`, `cached_at`.

**Users** — standard WordPress users; auth token hash stored as `bb_token` user meta.

---

## 6. Why it is resilient

| Scenario | Result |
|----------|--------|
| Repeat search | Served from the database — **0 Google calls** |
| Google rate-limited / down, book previously saved | Still served (permanent store) |
| Google down, brand-new query never saved | `503` — never fabricated data |
| API key removed | Everything already saved keeps working |

This design keeps the app fast, cheap on API quota, and available even when the
external provider is not.

---

## 7. Deployment topology

```
        Browser
           │
   ┌───────┴────────┐
   │  book-web      │  static build served by nginx (or `npm run dev`)
   └───────┬────────┘
           │ /wp-json/books/v1/*
   ┌───────┴────────┐
   │  WordPress     │  Apache/PHP + book-backend plugin
   │  + MySQL       │  (or portable PHP + SQLite for low-resource dev)
   └───────┬────────┘
           │ only on new queries
     Google Books API
```

Local dev runs via Docker (`make up`) or portable PHP + SQLite — see
[Deployment](deployment.md).
