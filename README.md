# URL Shortener

A small URL shortener built with Flask, PostgreSQL and Redis, run with Docker Compose.

- **PostgreSQL** stores every URL permanently. Each URL gets an auto-incrementing ID.
- **Base62 encoding** turns that ID into a short code using `0-9`, `a-z` and `A-Z` (for example, ID `125` becomes `21`).
- **Redis** caches short code → URL lookups so most redirects skip the database. It also holds the rate-limit counters.

## Project structure

```
.
├── docker-compose.yml   # web, db (Postgres 16) and redis (Redis 7) services
└── app/
    ├── app.py           # Flask application
    ├── init.sql         # Creates the urls table on first database start
    ├── requirements.txt
    └── Dockerfile
```

## Running it

You need Docker with Docker Compose.

```bash
docker compose up --build -d
```

The API is then at `http://localhost:5000`. The web service waits for Postgres and Redis to pass their health checks before it starts.

Check that it's up:

```bash
curl http://localhost:5000/health
```

Follow the logs, which show cache hits and misses:

```bash
docker compose logs -f web
```

Stop everything:

```bash
docker compose down        # keeps the database volume
docker compose down -v     # also deletes all stored URLs
```

> `init.sql` only runs when the `postgres_data` volume is first created. If you change the schema, run `docker compose down -v` to recreate it.

## API

### `POST /shorten`

Creates a short URL. Limited to **10 requests per 60 seconds per client IP**.

Request body:

```json
{ "url": "https://www.google.com" }
```

Response `201 Created`:

```json
{ "short_url": "http://localhost:5000/1", "short_code": "1" }
```

Errors:

| Status | Cause |
| ------ | ----- |
| `400` | The body is missing or has no `url` field |
| `429` | Rate limit exceeded |

### `GET /<short_code>`

Redirects (`302`) to the original URL. It checks Redis first, then falls back to Postgres and caches the result. Returns `404` if the code doesn't exist.

### `GET /health`

Returns `{"status": "healthy"}`.

## Example requests

macOS / Linux / Git Bash:

```bash
curl -X POST http://localhost:5000/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.google.com"}'

curl -i http://localhost:5000/1
```

Windows `cmd` doesn't treat single quotes as quotes, so wrap the body in double quotes and escape the inner ones:

```cmd
curl -X POST http://localhost:5000/shorten -H "Content-Type: application/json" -d "{\"url\": \"https://www.google.com\"}"
```

PowerShell:

```powershell
Invoke-RestMethod -Method Post -Uri http://localhost:5000/shorten -ContentType "application/json" -Body '{"url": "https://www.google.com"}'
```

## Configuration

The web service reads these environment variables, which are set in `docker-compose.yml`:

| Variable | Default |
| -------- | ------- |
| `REDIS_HOST` | `redis` |
| `REDIS_PORT` | `6379` |
| `POSTGRES_HOST` | `db` |
| `POSTGRES_DB` | `urlshortener` |
| `POSTGRES_USER` | `postgres` |
| `POSTGRES_PASSWORD` | `postgres` |

The default credentials are for local development only.

## Known limitations

- `/shorten` doesn't check that the URL is valid.
- A short code containing a character outside `0-9a-zA-Z` (for example, `/favicon.ico`) causes a `500` error instead of a `404`.
- The app runs on Flask's development server, so it isn't suitable for production as-is.
- Short URLs always use `http://localhost:5000`.
