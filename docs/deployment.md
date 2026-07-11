# Deployment

## Local (WordPress + MySQL + React)

From the repo root:

```bash
docker compose up -d
```

Starts MySQL and WordPress with the `book-backend` plugin mounted at
`wp-content/plugins/book-backend`. Then:

1. Open `http://localhost:8080` and run the one-time WordPress installer.
2. **wp-admin → Plugins → activate "Book Backend"**.
3. Run the frontend:

```bash
cd book-web && npm install && cp .env.example .env && npm run dev   # :5173
```

## Production shape (future)

```
book-web ──▶ CDN ──▶ WordPress (book-backend plugin) ──▶ Google Books
                          │
                        MySQL  +  object cache (Redis) for transients
```

- Put a real object cache (e.g. Redis via a drop-in) behind the Transients API for
  shared, persistent caching.
- **Add before going public:** authentication / API keys / rate limiting on the REST
  routes, and lock down CORS (the dev build allows all origins).
