# Go Proxy Backend

Generic proxy server with JWT authentication and row-level authorization for NocoDB.

## Features

- **JWT Authentication** - Validates tokens and extracts user claims
- **Row-Level Authorization** - Automatically filters data based on `created_by` field
- **Generic Middleware** - Domain-agnostic, works with any NocoDB table
- **Secure Proxying** - Hides NocoDB xc-token from frontend
- **MetaCache** - Automatically resolves friendly table names to NocoDB IDs
- **Environment Loading** - Automatically loads `.env` file with logging
- **Comprehensive Logging** - Detailed logs at every step for debugging

## Setup

1. Copy environment variables:
```bash
cp .env.example .env
```

2. Edit `.env` with your configuration:
```env
PORT=8080
NOCODB_URL=http://localhost:8090/api/v3/data/pbf7tt48gxdl50h/
NOCODB_BASE_ID=pbf7tt48gxdl50h
NOCODB_TOKEN=your_nocodb_token_here
JWT_SECRET=your_jwt_secret_here
```

**Important:** `NOCODB_BASE_ID` is required for MetaCache to work. Get it from your NocoDB URL.

3. Install dependencies:
```bash
go mod download
```

4. Run the server:
```bash
go run main.go
```

The server will start on `http://localhost:8080`

**Startup Logs:**
```
[STARTUP] Initializing Generic Proxy Server...
[STARTUP] .env file loaded successfully
[STARTUP] Configuration loaded:
  - Port: 8080
  - NocoDB URL: http://localhost:8090/api/v3/data/pbf7tt48gxdl50h/
  - NocoDB Base ID: pbf7tt48gxdl50h
  - JWT Secret: 4c6c****28af
[STARTUP] Meta Base URL: http://localhost:8090/api/v2/
[META] Starting auto-refresh goroutine (interval: 10m0s)
[META] Fetching table metadata from NocoDB...
[META] Metadata URL: http://localhost:8090/api/v2/meta/bases/pbf7tt48gxdl50h/tables
[META] Mapped 'Products' -> 'm7rl42lk4m0nq27'
[META] Mapped 'Quotes' -> 'mqsc4pb7g3vj2ex'
[META] Mapped 'Contacts' -> 'mpfy0dt80i6vba7'
[META] Successfully loaded 6 table mappings from NocoDB
```

If `.env` file is not found, you'll see:
```
[STARTUP WARN] .env file not found or could not be loaded - using defaults
```

## API Endpoints

### POST /login
Authenticate and receive JWT token.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "user123"
}
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user_id": "user-001",
  "role": "user"
}
```

### GET /proxy/*
Generic proxy endpoint for NocoDB requests. Requires JWT authentication.

**With MetaCache (Friendly Names):**
```bash
# Use friendly table names - automatically resolved to IDs
curl http://localhost:8080/proxy/products/records \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"

curl http://localhost:8080/proxy/quotes/records \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

The proxy automatically resolves:
- `products` → `m7rl42lk4m0nq27` (actual NocoDB table ID)
- `quotes` → `mqsc4pb7g3vj2ex`
- etc.

## MetaCache System

The MetaCache automatically loads table metadata from NocoDB and maintains a mapping of friendly names to table IDs.

### How It Works

1. **On Startup**: Fetches table metadata from `/api/v2/meta/bases/{baseId}/tables`
2. **Builds Mapping**: Creates lowercase name → ID mappings
3. **Auto-Refresh**: Refreshes cache every 10 minutes
4. **Thread-Safe**: Uses read-write locks for concurrent access

### Request Flow with MetaCache

```
Frontend: GET /proxy/products/records
    ↓
[AUTH] Validates JWT
    ↓
[AUTHORIZE] Applies row-level filtering (if needed)
    ↓
[META] Resolves 'products' → 'm7rl42lk4m0nq27'
    ↓
[PROXY] Forwards to: http://nocodb/api/v3/data/pbf7tt48gxdl50h/m7rl42lk4m0nq27/records
    ↓
NocoDB returns data
```

### Logs Example

```
[META] Resolved table 'products' -> 'm7rl42lk4m0nq27'
[PROXY] Target URL: http://100.103.198.65:8090/api/v3/data/pbf7tt48gxdl50h/m7rl42lk4m0nq27/records
[PROXY] NocoDB responded with status: 200 OK
```

### Benefits

- ✅ **No Hardcoded IDs** - Everything driven by NocoDB metadata
- ✅ **User-Friendly** - Frontend uses readable names like "products", "quotes"
- ✅ **Automatic Updates** - Cache refreshes periodically
- ✅ **Fallback** - If no mapping found, uses raw name (backward compatible)

## Architecture

### Middleware Chain

```
Request → CORS → Auth Middleware → Authorize Middleware → Proxy Handler → NocoDB
```

1. **CORS Middleware** - Handles cross-origin requests
2. **Auth Middleware** - Validates JWT and extracts claims
3. **Authorize Middleware** - Applies row-level filtering for non-admin users
4. **Proxy Handler** - Forwards request to NocoDB with xc-token

### Row-Level Filtering

For non-admin users, the authorization middleware automatically injects a `where` clause:

```
?where=(created_by,eq,<user_id>)
```

This ensures users only see records they created.

**Admin users** bypass this filtering and see all records.

## Demo Users

The system includes two hardcoded demo users:

| Email | Password | User ID | Role |
|-------|----------|---------|------|
| admin@example.com | admin123 | admin-001 | admin |
| user@example.com | user123 | user-001 | user |

## Testing

### Login
```bash
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"user123"}'
```

### Fetch Data
```bash
# Replace TOKEN with the JWT from login response
curl http://localhost:8080/proxy/quotes/records \
  -H "Authorization: Bearer TOKEN"
```

## Project Structure

```
backend/
├── main.go              # Entry point, server setup
├── middleware/
│   ├── auth.go         # JWT validation
│   └── authorize.go    # Row-level filtering
├── proxy/
│   └── handler.go      # NocoDB proxy logic
├── utils/
│   └── jwt.go          # JWT generation and validation
├── go.mod              # Go dependencies
└── .env.example        # Environment template
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| PORT | Server port | 8080 |
| NOCODB_URL | NocoDB API base URL | http://localhost:8090/api/v3/data/project/ |
| NOCODB_TOKEN | NocoDB API token | secret123 |
| JWT_SECRET | Secret for JWT signing | myjwtsecret |

## Security Notes

- JWT tokens expire after 24 hours
- NocoDB token is never exposed to frontend
- Row-level filtering is applied at the proxy layer
- CORS is enabled for frontend access
