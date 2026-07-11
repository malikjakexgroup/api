# Architecture

Full architecture of the **Book Platform** — a three-repository book discovery app:
a **React** frontend, a **WordPress** backend that *is* the API (the middle man), and
**Google Books** as the data source. Every diagram renders natively on GitHub (Mermaid).

**Repos:** `frontend` (React) · `backend` (WordPress + API) · `api` (this — spec, Swagger, docs)

---

## 1. The middle man (gateway)

The frontend never talks to Google. **The backend sits in the middle** — it decides,
stores, hides the API key, and serves clean data.

```mermaid
flowchart LR
    browser["⚛️ frontend<br/>React"]
    api["🎯 backend — the middle man<br/>WordPress (the API)"]
    db[("🗄️ Database<br/>saved books")]
    google["🌐 Google Books<br/>external source"]

    browser -->|"asks"| api
    api -->|"answers"| browser
    api -->|"read / write"| db
    api -->|"only for NEW books"| google

    classDef mid fill:#e5efe9,stroke:#2f6b57,stroke-width:2px,color:#1c3f34;
    classDef ext fill:#f0e7d6,stroke:#9c6f2f,color:#4a3412;
    class api mid;
    class google ext;
```

---

## 2. Component / container view

```mermaid
flowchart TB
    subgraph client["Client (browser)"]
        web["⚛️ frontend — React SPA<br/>pages · hooks · api.js"]
    end

    subgraph wp["🔌 backend — WordPress plugin (the API)"]
        direction TB
        rest["REST controller<br/>rest.php"]
        auth["Auth<br/>auth.php · tokens"]
        gb["Google client<br/>google-books.php"]
        store["Storage<br/>store.php · book CPT"]
        cache["Cache<br/>cache.php · transients"]
    end

    admin["🛠️ wp-admin"]
    db[("🗄️ MySQL / SQLite")]
    google["🌐 Google Books API"]

    web -->|"HTTPS · JSON<br/>/wp-json/books/v1/*"| rest
    rest --> auth
    rest --> cache
    rest --> store
    rest --> gb
    gb -->|"new queries only"| google
    store --> db
    auth --> db
    admin --> db

    classDef ext fill:#f0e7d6,stroke:#9c6f2f,color:#4a3412;
    class google ext;
```

---

## 3. Three-repo map

```mermaid
flowchart TB
    frontend["⚛️ frontend<br/>React app"]
    backend["🔌 backend<br/>WordPress + API"]
    apidocs["📖 api<br/>OpenAPI · Swagger · docs"]

    frontend -->|"REST API (runtime)"| backend
    apidocs -. documents .-> backend
    apidocs -. documents .-> frontend
```

Each box is an **independent Git repository** — built, tested, and deployed on its own.

---

## 4. Flow — Search

```mermaid
sequenceDiagram
    actor U as User
    participant W as frontend (React)
    participant B as backend (API)
    participant D as Database
    participant G as Google Books

    U->>W: search "atomic habits"
    W->>B: GET /books/v1/search?q=
    B->>D: bb_search_stored(q)
    alt Already saved
        D-->>B: matching books
        B-->>W: 200 — saved books (0 Google calls)
    else New query
        B->>G: fetch from Google
        alt Google OK
            G-->>B: results
            B->>D: bb_upsert_book() — store permanently
            B-->>W: 200 — books
        else Google down / rate-limited
            B-->>W: 503 — never fabricated data
        end
    end
    W-->>U: render book grid
```

---

## 5. Flow — Book detail

```mermaid
sequenceDiagram
    actor U as User
    participant W as frontend
    participant B as backend
    participant D as Database
    participant G as Google Books

    U->>W: open a book
    W->>B: GET /books/v1/book/{id}
    B->>D: bb_find_book_post(id)
    alt Saved
        D-->>B: book
        B-->>W: 200 — saved (permanent)
    else Not saved yet
        B->>G: bb_google_get(id)
        alt Found
            G-->>B: book
            B->>D: bb_upsert_book()
            B-->>W: 200 — book
        else Missing / unavailable
            B-->>W: 404 / 503
        end
    end
    W-->>U: render detail page
```

---

## 6. Flow — Authentication

```mermaid
sequenceDiagram
    actor U as User
    participant W as frontend
    participant B as backend
    participant D as WordPress Users

    U->>W: sign up (name, email, password)
    W->>B: POST /register
    B->>D: wp_create_user()
    B->>B: issue token → store SHA-256 hash
    B-->>W: { token, user }
    W->>W: save token in localStorage

    Note over W,B: every later request sends<br/>Authorization: Bearer token

    W->>B: GET /me (Bearer token)
    B->>D: resolve token → user
    B-->>W: current user
```

---

## 7. Backend internal pipeline (function-level)

```mermaid
flowchart TD
    A["GET /books/v1/search?q="] --> B["bb_rest_search()"]
    B --> C["bb_search_stored(q)"]
    C --> D{"matches in<br/>database?"}
    D -->|Yes| E["bb_book_to_array()<br/>for each"] --> R["200 · JSON"]
    D -->|No| F["bb_google_search(q)"]
    F --> G["bb_google_url() + wp_remote_get()"]
    G --> H{"Google<br/>responded?"}
    H -->|Yes| I["bb_normalize() each"] --> J["bb_upsert_book()<br/>store permanently"] --> R
    H -->|No| K["503 · unavailable"]

    classDef ok fill:#e5efe9,stroke:#2f6b57,color:#1c3f34;
    classDef bad fill:#f6dcdc,stroke:#c0392b,color:#5a1f1a;
    class E,I,J,R ok;
    class K bad;
```

---

## 8. Resilience

```mermaid
flowchart TD
    A["Search q"] --> B{"Saved in<br/>database?"}
    B -->|Yes| C["Return saved books<br/>permanent · 0 API calls"]
    B -->|No| D{"Google<br/>available?"}
    D -->|Yes| E["Fetch + store<br/>permanently"] --> F["Return books"]
    D -->|No| G["503<br/>never fabricated data"]

    classDef ok fill:#bbf7d0,stroke:#16a34a,color:#08331c;
    classDef bad fill:#fecaca,stroke:#dc2626,color:#5a1a1a;
    class C,F ok;
    class G bad;
```

| Scenario | Result |
|----------|--------|
| Repeat search | Served from DB — **0 Google calls** |
| Google down, book previously saved | Still served |
| Google down, brand-new query | `503` — no fake data |
| API key removed | Everything already saved keeps working |

---

## 9. Data model

```mermaid
erDiagram
    BOOK {
        string google_id PK
        string title
        json   authors
        string publisher
        text   description
        string thumbnail
        int    pages
        json   categories
        float  rating
    }
    USER {
        int    id PK
        string name
        string email
        string bb_token_hash
    }
    FAVORITE {
        string google_id FK
    }
    USER ||--o{ FAVORITE : "saves (client-side)"
    FAVORITE }o--|| BOOK : references
```

---

## 10. Deployment

```mermaid
flowchart LR
    user((User))
    cdn["nginx / CDN<br/>frontend static build"]
    wp["backend — WordPress"]
    db[("MySQL")]
    g["Google Books API"]

    user --> cdn
    cdn -->|"/wp-json/*"| wp
    wp --> db
    wp -->|"new queries only"| g

    classDef ext fill:#f0e7d6,stroke:#9c6f2f,color:#4a3412;
    class g ext;
```

See [How It Works](how-it-works.md) for a function-by-function breakdown, and the
[Swagger API Reference](swagger.md) for the live endpoint docs.
