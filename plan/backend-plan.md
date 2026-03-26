# Backend Architecture Plan — Territory Running App Prototype

## Table of Contents

- [1. What We're Building](#1-what-were-building)
- [2. Why These Technologies](#2-why-these-technologies)
- [3. How the Pieces Fit Together](#3-how-the-pieces-fit-together)
- [4. Project Structure](#4-project-structure)
- [5. Database Design](#5-database-design)
- [6. The Grid System — How Territory Works](#6-the-grid-system--how-territory-works)
- [7. API Design](#7-api-design)
- [8. WebSocket — Real-Time Communication](#8-websocket--real-time-communication)
- [9. Run Processing Pipeline](#9-run-processing-pipeline)
- [10. Authentication](#10-authentication)
- [11. Error Handling & What Can Go Wrong](#11-error-handling--what-can-go-wrong)
- [12. Configuration & Environment](#12-configuration--environment)
- [13. Build Phases](#13-build-phases)

---

## 1. What We're Building

A backend server that does three core things:

1. **Accepts GPS data** from the Android app while users are running
2. **Calculates which grid cells** a runner passed through and marks them as "claimed"
3. **Broadcasts territory changes** in real-time to all connected users

Think of it like a game server. The Android app sends your location every few seconds. The server figures out which squares on the map you ran through, saves them as yours, and tells everyone else "hey, this area just got claimed."

### Request Lifecycle (what happens when the app talks to the server)

```
Android App
    │
    ▼
┌──────────────────────────────┐
│         Go HTTP Server       │   ← Receives HTTP requests and WebSocket connections
│  (net/http + gorilla/mux)    │
├──────────────────────────────┤
│       Route Handler          │   ← Decides what to do based on the URL path
│  e.g. POST /api/runs/start   │
├──────────────────────────────┤
│      Business Logic          │   ← The actual work: validate data, compute grids,
│   (services / domain)        │      update ownership, calculate scores
├──────────────────────────────┤
│      Database Layer          │   ← Talks to PostgreSQL. Reads/writes data.
│   (repository pattern)       │      Uses PostGIS for geographic queries.
├──────────────────────────────┤
│      WebSocket Hub           │   ← Keeps track of all connected clients.
│                              │      Broadcasts territory updates in real-time.
└──────────────────────────────┘
           │
           ▼
┌──────────────────────────────┐
│   PostgreSQL + PostGIS       │   ← Stores everything: users, runs, GPS points,
│                              │      grid cells, territory ownership
└──────────────────────────────┘
```

---

## 2. Why These Technologies

### Go (Golang)

**What it is:** A programming language created by Google in 2009. It compiles to a single binary (one file you can just run — no "install Node" or "install Python" needed on the server).

**Why we're using it:**

- **Concurrency built in.** Go has "goroutines" — lightweight threads that let you handle thousands of simultaneous connections without complex code. This matters because we need WebSocket connections open for every active user, and we need to process GPS data from multiple runners at the same time.
- **Simple language.** Go intentionally has fewer features than languages like Java or Rust. No classes, no inheritance, no generics magic. This means there's less to learn, and code tends to look the same no matter who writes it.
- **Fast compilation.** Your code compiles in seconds, not minutes. This means fast feedback while developing.
- **Single binary deployment.** `go build` gives you one file. Copy it to a server, run it. No runtime dependencies.

**Key Go concepts you'll encounter:**

```
Package     → Like a folder/module. All .go files in a folder share a package.
Struct      → Go's version of a class (but no inheritance). Just a collection of fields.
Interface   → A contract that says "anything with these methods counts as this type."
Goroutine   → A lightweight thread. Start one with: go myFunction()
Channel     → A pipe that goroutines use to send data to each other safely.
Error       → Go doesn't have exceptions. Functions return errors as values: result, err := doThing()
```

### PostgreSQL + PostGIS

**PostgreSQL** is a relational database (like MySQL, but more powerful). It stores your data in tables with rows and columns.

**PostGIS** is an extension (a plugin) for PostgreSQL that adds geographic superpowers. Without PostGIS, your database understands numbers and text. With PostGIS, it understands **locations, areas, distances, and shapes**.

**Why we need PostGIS specifically:**

- "Which grid cells does this GPS track pass through?" → PostGIS can answer this with a single SQL query using `ST_Intersects`
- "How much total area does this user own?" → PostGIS can calculate this
- "Which territories are within the visible map area?" → PostGIS handles bounding-box queries efficiently

**Without PostGIS**, you'd have to write all this geographic math yourself in Go. It would be slower, buggier, and way more code.

**Key PostGIS concepts:**

```
POINT(lng, lat)      → A single location (e.g., where you are right now)
LINESTRING(...)      → A series of connected points (e.g., your running route)
POLYGON(...)         → A closed shape (e.g., a grid cell boundary)
SRID 4326            → The coordinate system GPS uses (latitude/longitude)
ST_Intersects(a, b)  → Returns true if shape A overlaps with shape B
ST_MakeEnvelope(...) → Creates a rectangle from corner coordinates
```

### WebSocket

**What it is:** A protocol that keeps a connection open between the client and server. Unlike normal HTTP (client asks → server responds → connection closes), WebSocket stays open so both sides can send messages at any time.

**Why we need it:** When Player A claims a territory, Player B's map should update immediately. Without WebSocket, Player B would have to keep asking "did anything change?" every few seconds (called "polling"), which wastes bandwidth and battery. With WebSocket, the server just pushes the update the instant it happens.

**How it works in our app:**

```
1. App opens WebSocket connection: ws://server/ws
2. Server adds this connection to the "hub" (list of connected clients)
3. When someone claims territory:
   Server → sends JSON message to ALL connections in the hub
4. App receives message → updates map immediately
5. If connection drops → app reconnects automatically
```

### Mapbox GL (Backend Relevance)

Mapbox GL is primarily a frontend technology (renders maps on the device), but the backend needs to know about it because:

- We use the same coordinate system (WGS84 / SRID 4326)
- Grid cells need to align with what Mapbox renders
- The backend serves territory data that Mapbox will display

---

## 3. How the Pieces Fit Together

```
                    ┌─────────────────────────────────────────────────┐
                    │                  Go Server                      │
                    │                                                 │
 Android App ──────►│  ┌─────────┐    ┌───────────┐    ┌──────────┐ │
  (HTTP REST)       │  │  Router  │───►│  Handlers │───►│ Services │ │
                    │  └─────────┘    └───────────┘    └────┬─────┘ │
                    │                                       │       │
                    │                                       ▼       │
 Android App ◄─────►│  ┌──────────────┐              ┌───────────┐ │
  (WebSocket)       │  │ WebSocket Hub│◄─────────────│Repository │ │
                    │  │              │  (notifies    │  (DB ops) │ │
                    │  │ • track conns│   on change)  └─────┬─────┘ │
                    │  │ • broadcast  │                     │       │
                    │  └──────────────┘                     │       │
                    └───────────────────────────────────────┼───────┘
                                                            │
                                                            ▼
                                                    ┌──────────────┐
                                                    │  PostgreSQL  │
                                                    │  + PostGIS   │
                                                    └──────────────┘
```

**The flow when someone is running:**

1. Android app sends GPS coordinates every 3-5 seconds via HTTP POST
2. Handler receives the request, validates it
3. Service layer calculates which grid cells the GPS points fall into
4. Repository saves the GPS points and updates grid cell ownership in PostgreSQL
5. Service notifies the WebSocket Hub: "cells X, Y, Z are now owned by user A"
6. WebSocket Hub broadcasts this to all connected clients
7. Everyone's map updates in real-time

---

## 4. Project Structure

```
running-app-backend/
│
├── cmd/                        # Application entry points
│   └── server/
│       └── main.go             # Starts the server. This is where everything begins.
│
├── internal/                   # Private application code (Go convention: code in "internal"
│   │                           # can only be imported by code in the parent directory)
│   │
│   ├── config/
│   │   └── config.go           # Loads configuration (DB connection string, port, etc.)
│   │                           # from environment variables or a .env file
│   │
│   ├── models/                 # Data structures — the "shapes" of your data
│   │   ├── user.go             # User struct: id, username, email, password hash
│   │   ├── run.go              # Run struct: id, user_id, start_time, end_time, status
│   │   ├── gps_point.go        # GPSPoint struct: lat, lng, timestamp, accuracy
│   │   └── grid_cell.go        # GridCell struct: row, col, owner_id, claimed_at
│   │
│   ├── handler/                # HTTP handlers — receive requests, return responses
│   │   ├── auth.go             # Login, register, token refresh
│   │   ├── run.go              # Start run, send GPS points, end run
│   │   ├── territory.go        # Get territories for a map area
│   │   ├── leaderboard.go      # Get rankings
│   │   └── websocket.go        # Upgrade HTTP connection to WebSocket
│   │
│   ├── service/                # Business logic — the "brain" of the app
│   │   ├── auth_service.go     # Password hashing, token generation, validation
│   │   ├── run_service.go      # Run state management, GPS processing
│   │   ├── grid_service.go     # GPS-to-grid conversion, territory calculations
│   │   └── leaderboard_service.go
│   │
│   ├── repository/             # Database operations — SQL queries live here
│   │   ├── user_repo.go        # CRUD operations for users
│   │   ├── run_repo.go         # CRUD operations for runs and GPS points
│   │   ├── grid_repo.go        # Grid cell queries (claim, get by area, etc.)
│   │   └── leaderboard_repo.go
│   │
│   ├── websocket/              # Real-time communication
│   │   ├── hub.go              # Manages all active connections, broadcasts messages
│   │   └── client.go           # Represents one connected user
│   │
│   ├── middleware/              # Code that runs BEFORE your handlers
│   │   ├── auth.go             # Checks JWT token on protected routes
│   │   └── logging.go          # Logs every request (method, path, duration)
│   │
│   └── migration/              # Database migrations — version-controlled schema changes
│       ├── 001_create_users.sql
│       ├── 002_create_runs.sql
│       ├── 003_create_gps_points.sql
│       └── 004_create_grid_cells.sql
│
├── go.mod                      # Go module file — lists dependencies (like package.json)
├── go.sum                      # Lock file — exact versions of dependencies
├── .env.example                # Template for environment variables
└── Makefile                    # Shortcut commands: make run, make migrate, make build
```

**Why this structure?**

This follows a common Go project layout. The key idea is **separation of concerns**:

- **handler/** knows about HTTP (requests, responses, status codes) but NOT about database queries
- **service/** knows about business rules (how grids work, who owns what) but NOT about HTTP or SQL
- **repository/** knows about SQL and the database but NOT about HTTP or business rules

This means if you want to change how grid claiming works, you only touch `service/grid_service.go`. If you want to change a SQL query, you only touch the repository. Nothing else breaks.

**Why `internal/`?** It's a Go convention. Any code inside `internal/` can only be imported by code within this project. It prevents other projects from depending on your internal code. For a prototype, it's just good hygiene.

---

## 5. Database Design

### Entity Relationship Diagram

```
┌──────────────┐       ┌──────────────────┐       ┌──────────────────┐
│    users     │       │      runs        │       │   gps_points     │
├──────────────┤       ├──────────────────┤       ├──────────────────┤
│ id (PK)      │──┐    │ id (PK)          │──┐    │ id (PK)          │
│ username     │  │    │ user_id (FK)     │  │    │ run_id (FK)      │
│ email        │  ├───►│ started_at       │  ├───►│ latitude         │
│ password_hash│  │    │ ended_at         │  │    │ longitude        │
│ created_at   │  │    │ status           │  │    │ recorded_at      │
└──────────────┘  │    │ distance_meters  │  │    │ accuracy_meters  │
                  │    │ duration_seconds │  │    │ location (POINT) │
                  │    └──────────────────┘  │    └──────────────────┘
                  │                          │
                  │    ┌──────────────────┐  │    ┌──────────────────┐
                  │    │   grid_cells     │  │    │ cell_claims      │
                  │    ├──────────────────┤  │    ├──────────────────┤
                  │    │ id (PK)          │  │    │ id (PK)          │
                  └───►│ owner_id (FK)    │  └───►│ run_id (FK)      │
                       │ grid_row         │       │ cell_id (FK)     │
                       │ grid_col         │       │ claimed_at       │
                       │ claimed_at       │       └──────────────────┘
                       │ bounds (POLYGON) │
                       └──────────────────┘
```

### Table Details

#### `users` — Who's playing

```sql
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username      VARCHAR(50) UNIQUE NOT NULL,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**Why UUID instead of auto-increment integer?**
UUIDs (like `550e8400-e29b-41d4-a716-446655440000`) are globally unique. If you ever merge databases, add replication, or expose IDs in URLs, integers cause problems (user 1 on server A ≠ user 1 on server B). UUIDs avoid this. For a prototype it's mild overkill, but it costs us nothing and avoids a painful migration later.

**Why `password_hash` not `password`?**
Never store plain passwords. We'll use bcrypt to hash them. Even if someone steals the database, they can't recover the passwords.

#### `runs` — Individual running sessions

```sql
CREATE TABLE runs (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID NOT NULL REFERENCES users(id),
    started_at       TIMESTAMP WITH TIME ZONE NOT NULL,
    ended_at         TIMESTAMP WITH TIME ZONE,
    status           VARCHAR(20) NOT NULL DEFAULT 'active',
                     -- 'active', 'completed', 'discarded'
    distance_meters  DOUBLE PRECISION DEFAULT 0,
    duration_seconds INTEGER DEFAULT 0,
    created_at       TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_runs_user_id ON runs(user_id);
CREATE INDEX idx_runs_status ON runs(status);
```

**What's an INDEX?**
Think of it like a book's index. Without it, the database reads every single row to find what you want (slow). With it, it can jump directly to the right rows. We index `user_id` because we'll often ask "give me all runs for this user" and `status` because we'll ask "are there any active runs?"

**Why `status` as a string instead of boolean `is_active`?**
A run can be active, completed, or discarded (user cancelled). That's 3 states — a boolean only handles 2.

#### `gps_points` — Raw GPS breadcrumbs

```sql
CREATE TABLE gps_points (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(id),
    latitude        DOUBLE PRECISION NOT NULL,
    longitude       DOUBLE PRECISION NOT NULL,
    recorded_at     TIMESTAMP WITH TIME ZONE NOT NULL,
    accuracy_meters DOUBLE PRECISION,
    location        GEOGRAPHY(POINT, 4326) NOT NULL
);

CREATE INDEX idx_gps_points_run_id ON gps_points(run_id);
CREATE INDEX idx_gps_points_location ON gps_points USING GIST(location);
```

**What's `GEOGRAPHY(POINT, 4326)`?**
This is PostGIS's way of storing a geographic point. `4326` is the SRID (Spatial Reference ID) — it means "standard GPS coordinates (latitude/longitude on Earth's surface)." By storing the location as a PostGIS geography type, we can use spatial functions like "find all points within 100 meters of X" directly in SQL.

**What's a GIST index?**
Regular indexes work for exact matches and ranges (numbers, text). Geographic data needs a different indexing strategy. GIST (Generalized Search Tree) indexes understand spatial relationships — "is this point inside this polygon?" — and make those queries fast.

**Why store latitude/longitude AND location?**
The `latitude`/`longitude` columns are for easy reading and debugging. The `location` column is for PostGIS queries. It's slight duplication, but it makes both use cases simple.

#### `grid_cells` — The territory grid

```sql
CREATE TABLE grid_cells (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grid_row   INTEGER NOT NULL,
    grid_col   INTEGER NOT NULL,
    owner_id   UUID REFERENCES users(id),
    claimed_at TIMESTAMP WITH TIME ZONE,
    bounds     GEOGRAPHY(POLYGON, 4326) NOT NULL,

    UNIQUE(grid_row, grid_col)  -- Each cell position can only exist once
);

CREATE INDEX idx_grid_cells_owner ON grid_cells(owner_id);
CREATE INDEX idx_grid_cells_bounds ON grid_cells USING GIST(bounds);
CREATE INDEX idx_grid_cells_row_col ON grid_cells(grid_row, grid_col);
```

**Why `UNIQUE(grid_row, grid_col)`?**
Grid cell (5, 10) should only exist once in the database. Without this constraint, a bug could create duplicate cells. The database enforces this so we don't have to worry about it in code.

**Do we pre-create all grid cells?**
No. We create them lazily — a cell row is only inserted when someone first claims it. For 10-15 users, the number of cells will be small. Pre-creating them for the entire map would be millions of rows for no reason.

#### `cell_claims` — History of who claimed what during which run

```sql
CREATE TABLE cell_claims (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id     UUID NOT NULL REFERENCES runs(id),
    cell_id    UUID NOT NULL REFERENCES grid_cells(id),
    claimed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_cell_claims_run_id ON cell_claims(run_id);
```

**Why a separate claims table?**
`grid_cells` only tracks the *current* owner. `cell_claims` tracks *history* — "during this run, these cells were claimed." This lets us show stats like "you claimed 15 cells on your run today" and also supports a future feature where cells can be re-claimed (stolen) by other runners.

---

## 6. The Grid System — How Territory Works

This is the heart of the app. Let's break it down completely.

### What is the grid?

Imagine laying a sheet of graph paper over the entire map. Each small square is a "grid cell." When you run through a cell, it becomes yours and gets colored in your color on everyone's map.

### Grid Parameters

```
GRID_CELL_SIZE = 0.0005 degrees ≈ 55 meters at the equator

Why 55 meters?
- Small enough that running down a street claims a meaningful path
- Large enough that GPS inaccuracy (typically 5-15m) doesn't cause
  chaotic claiming of cells you didn't actually run through
- At 10-15 users, this creates a manageable number of cells
```

**Important note about degrees vs meters:** 1 degree of latitude ≈ 111 km everywhere on Earth. 1 degree of longitude ≈ 111 km at the equator but shrinks as you go toward the poles (it's 0 km at the North/South Pole). For a prototype in a single city, this approximation is fine. For production, you'd use a proper projection.

### How GPS Points Map to Grid Cells

```
Step 1: Receive GPS point (lat: 12.9716, lng: 77.5946)

Step 2: Calculate grid row and column
   grid_row = floor(lat / CELL_SIZE) = floor(12.9716 / 0.0005) = 25943
   grid_col = floor(lng / CELL_SIZE) = floor(77.5946 / 0.0005) = 155189

Step 3: Calculate cell boundaries
   min_lat = grid_row * CELL_SIZE = 25943 * 0.0005 = 12.9715
   max_lat = min_lat + CELL_SIZE  = 12.9720
   min_lng = grid_col * CELL_SIZE = 155189 * 0.0005 = 77.5945
   max_lng = min_lng + CELL_SIZE  = 77.5950

Step 4: Cell (25943, 155189) → check if exists in DB
   If not exists → INSERT with owner = current user
   If exists but owned by someone else → UPDATE owner to current user (takeover!)
   If exists and already owned by current user → skip (no change)
```

### Handling the GPS Path Between Points

The app sends GPS points every 3-5 seconds. If someone is running fast, there might be 20-50 meters between consecutive points. We need to claim ALL cells between two points, not just the cells where we got a GPS reading.

```
Point A (GPS reading at T=0)          Point B (GPS reading at T=3s)
    ●─────────────────────────────────────●
    │                                     │
    ▼                                     ▼
 Cell(5,10)                          Cell(5,13)

Without interpolation: Only cells (5,10) and (5,13) get claimed
With interpolation:    Cells (5,10), (5,11), (5,12), (5,13) all get claimed
```

**How we interpolate:** Draw a straight line between Point A and Point B. Find every cell that this line passes through. This is a classic algorithm called "grid traversal" or "line rasterization." PostGIS can do this for us:

```sql
-- Find all grid cells that intersect with a line between two GPS points
SELECT gc.id, gc.grid_row, gc.grid_col
FROM grid_cells gc
WHERE ST_Intersects(
    gc.bounds,
    ST_MakeLine(
        ST_MakePoint(lng1, lat1)::geography,
        ST_MakePoint(lng2, lat2)::geography
    )
);
```

Or we can do it in Go with simple math (faster for this prototype since we're already computing the grid row/col):

```go
// Bresenham-like line algorithm for grid cells
func CellsBetween(lat1, lng1, lat2, lng2, cellSize float64) []GridCell {
    row1 := int(math.Floor(lat1 / cellSize))
    col1 := int(math.Floor(lng1 / cellSize))
    row2 := int(math.Floor(lat2 / cellSize))
    col2 := int(math.Floor(lng2 / cellSize))

    // Walk from (row1,col1) to (row2,col2), collecting every cell we cross
    // ... (Bresenham's line algorithm)
}
```

**For the prototype, we'll do the grid math in Go** (it's simpler and avoids extra DB queries per GPS point). PostGIS is reserved for spatial queries like "get all cells in this map viewport."

### Territory Takeover Rules (Prototype)

Keep it simple:

- **Last runner wins.** If you run through someone else's cell, it becomes yours.
- **No cooldown.** Cells can change hands immediately.
- **No team system.** Every player is independent.

These can be made more complex later (cooldown timers, team territories, "fortification" by running the same cell multiple times).

---

## 7. API Design

### REST Endpoints

All endpoints return JSON. Protected endpoints require a JWT token in the `Authorization` header.

#### Authentication

```
POST /api/auth/register
  Body: { "username": "thiru", "email": "thiru@email.com", "password": "..." }
  Response: { "user": { "id": "...", "username": "thiru" }, "token": "jwt-string" }

  What happens: Creates user, hashes password with bcrypt, generates JWT, returns both.

POST /api/auth/login
  Body: { "email": "thiru@email.com", "password": "..." }
  Response: { "user": { "id": "...", "username": "thiru" }, "token": "jwt-string" }

  What happens: Finds user by email, verifies password against hash, generates JWT.
```

#### Runs (Protected)

```
POST /api/runs/start
  Body: { "started_at": "2026-03-26T10:00:00Z" }
  Response: { "run_id": "uuid-here", "status": "active" }

  What happens: Creates a new run record. Only one active run per user allowed.
  Error case: If user already has an active run → 409 Conflict.

POST /api/runs/:id/gps
  Body: {
    "points": [
      { "lat": 12.9716, "lng": 77.5946, "recorded_at": "...", "accuracy": 8.5 },
      { "lat": 12.9718, "lng": 77.5948, "recorded_at": "...", "accuracy": 6.2 }
    ]
  }
  Response: {
    "cells_claimed": [
      { "row": 25943, "col": 155189, "was_taken_from": null },
      { "row": 25943, "col": 155190, "was_taken_from": "other-user-id" }
    ]
  }

  What happens:
  1. Validates the run exists and is active
  2. Filters out points with accuracy > 30m (unreliable GPS)
  3. For each consecutive pair of points, calculates all grid cells on the path
  4. Claims/takes over those cells in the database
  5. Broadcasts territory changes via WebSocket
  6. Returns which cells were claimed

  Why batch points? The app collects GPS points locally and sends them in batches
  (every 5-10 seconds) instead of one at a time. This reduces network requests
  and handles brief connectivity gaps.

POST /api/runs/:id/end
  Body: { "ended_at": "2026-03-26T10:35:00Z" }
  Response: {
    "run_id": "...",
    "status": "completed",
    "summary": {
      "distance_meters": 4250,
      "duration_seconds": 2100,
      "cells_claimed": 42,
      "cells_taken_from_others": 7
    }
  }

  What happens: Marks run as completed, calculates final stats.

GET /api/runs
  Response: { "runs": [ { run summary objects } ] }

  What happens: Returns all runs for the authenticated user, newest first.
```

#### Territory (Protected)

```
GET /api/territory?min_lat=12.95&max_lat=13.00&min_lng=77.55&max_lng=77.60
  Response: {
    "cells": [
      { "row": 25943, "col": 155189, "owner_id": "...", "owner_username": "thiru", "color": "#FF5733" },
      ...
    ]
  }

  What happens: Returns all claimed grid cells within the given map viewport.
  This is what the app calls when you pan/zoom the map to load visible territories.

  The PostGIS query:
  SELECT gc.*, u.username
  FROM grid_cells gc
  JOIN users u ON gc.owner_id = u.id
  WHERE ST_Intersects(
      gc.bounds,
      ST_MakeEnvelope(min_lng, min_lat, max_lng, max_lat, 4326)::geography
  );
```

#### Leaderboard (Protected)

```
GET /api/leaderboard
  Response: {
    "rankings": [
      { "rank": 1, "user_id": "...", "username": "thiru", "cells_owned": 156, "total_distance": 42000 },
      ...
    ]
  }

  What happens: Counts cells per owner, sorts descending. With 10-15 users,
  this is a trivial query — no caching needed.
```

### Why REST for GPS Points Instead of WebSocket?

You might think: "We already have WebSocket open, why not send GPS data through it?"

Good question. REST for GPS data because:
1. **Reliability.** HTTP has built-in request/response. You know if the server received your data (200 OK) or not (error). WebSocket is fire-and-forget — you'd need to build acknowledgment yourself.
2. **Retries.** If the HTTP request fails, the app retries the same batch. With WebSocket, you'd need to track which messages were acknowledged.
3. **Simplicity.** The GPS endpoint does real work (processes cells, updates DB) and returns a response (which cells were claimed). This is naturally a request/response pattern.

WebSocket is used for one-way broadcasts: server → all clients. That's its strength.

---

## 8. WebSocket — Real-Time Communication

### Architecture

```
                        ┌─────────────────────────────┐
                        │         WebSocket Hub        │
                        │                              │
                        │  clients map:                │
                        │    conn1 → user_id_A         │
                        │    conn2 → user_id_B         │
                        │    conn3 → user_id_C         │
                        │                              │
                        │  broadcast channel ◄─────────│──── Territory Service
                        │       │                      │     (sends updates)
                        │       ▼                      │
                        │  for each client:            │
                        │    send(message)             │
                        └──────┬──────┬──────┬─────────┘
                               │      │      │
                               ▼      ▼      ▼
                           App A   App B   App C
```

### Message Types (Server → Client)

```json
// Territory update — sent when cells change ownership
{
  "type": "territory_update",
  "data": {
    "cells": [
      {
        "row": 25943,
        "col": 155189,
        "owner_id": "user-uuid",
        "owner_username": "thiru",
        "color": "#FF5733"
      }
    ],
    "triggered_by": "user-uuid"
  }
}

// Player location — sent periodically so you can see other runners on the map
{
  "type": "player_location",
  "data": {
    "user_id": "user-uuid",
    "username": "thiru",
    "lat": 12.9716,
    "lng": 77.5946,
    "is_running": true
  }
}

// Run started/ended — so the leaderboard can update
{
  "type": "run_event",
  "data": {
    "user_id": "user-uuid",
    "event": "started"  // or "completed"
  }
}
```

### Connection Lifecycle

```
Client                                    Server
  │                                         │
  │──── GET /ws (upgrade request) ────────►│
  │     + JWT token in query param          │
  │                                         │
  │◄─── 101 Switching Protocols ───────────│
  │                                         │
  │     (connection is now WebSocket)       │
  │                                         │
  │◄─── territory_update messages ─────────│
  │◄─── player_location messages ──────────│
  │                                         │
  │──── ping ─────────────────────────────►│  (every 30s, keepalive)
  │◄─── pong ──────────────────────────────│
  │                                         │
  │──── connection closes ────────────────►│
  │     (or drops unexpectedly)             │
  │                                         │
  │     Hub removes client from map         │
  │                                         │
  │──── reconnect (new GET /ws) ──────────►│
  │     Hub adds client back                │
```

### Hub Implementation (Conceptual)

```go
type Hub struct {
    clients    map[*Client]bool    // All connected clients
    broadcast  chan []byte          // Messages to send to everyone
    register   chan *Client         // New connections
    unregister chan *Client         // Disconnections
}

func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true

        case client := <-h.unregister:
            delete(h.clients, client)
            close(client.send)

        case message := <-h.broadcast:
            for client := range h.clients {
                client.send <- message
                // If send fails, client is dead → unregister
            }
        }
    }
}
```

**What's `select`?** It's Go's way of waiting on multiple channels at once. Like a `switch` statement, but for channels. Whichever channel receives a value first, that case runs.

---

## 9. Run Processing Pipeline

This is the step-by-step flow when GPS points come in during a run:

```
GPS Points Arrive (POST /api/runs/:id/gps)
         │
         ▼
┌─────────────────────┐
│  1. Validate Input   │
│  • Run exists?       │
│  • Run is active?    │
│  • Points have valid │
│    lat/lng/time?     │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  2. Filter by        │
│     Accuracy         │
│  • Drop points with  │
│    accuracy > 30m    │
│  • Why? Bad GPS      │
│    would claim wrong │
│    cells             │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  3. Save GPS Points  │
│  • Bulk INSERT into  │
│    gps_points table  │
│  • Store raw data    │
│    for later analysis│
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  4. Calculate Grid   │
│     Cells            │
│  • For each pair of  │
│    consecutive points│
│  • Find all cells    │
│    the line crosses  │
│  • Deduplicate       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  5. Claim Cells      │
│  • For each cell:    │
│    ┌─ Not in DB?     │
│    │  INSERT new cell │
│    ├─ Owned by other?│
│    │  UPDATE owner    │
│    └─ Already mine?  │
│       Skip           │
│  • Record in         │
│    cell_claims table │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  6. Update Run Stats │
│  • Add distance      │
│  • Update duration   │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  7. Broadcast        │
│  • Send changed      │
│    cells to          │
│    WebSocket Hub     │
│  • All connected     │
│    users see the     │
│    update            │
└────────┬────────────┘
         │
         ▼
    Return Response
    (cells claimed)
```

### Transaction Safety

Steps 3-6 should happen inside a **database transaction**.

**What's a transaction?** It's a way to say "either ALL of these database operations succeed, or NONE of them do." If step 5 fails halfway through (say, the database crashes), the transaction rolls back and steps 3-4 are undone too. Without a transaction, you could end up with GPS points saved but cells not claimed — inconsistent state.

```go
tx, err := db.Begin()  // Start transaction
// ... do steps 3-6 ...
if err != nil {
    tx.Rollback()      // Undo everything
    return err
}
tx.Commit()            // All good, save everything
```

---

## 10. Authentication

### How JWT Works

JWT (JSON Web Token) is a way to authenticate users without storing sessions on the server.

```
Registration/Login Flow:
┌──────┐                          ┌──────┐                    ┌────┐
│ App  │                          │Server│                    │ DB │
└──┬───┘                          └──┬───┘                    └─┬──┘
   │  POST /auth/login               │                          │
   │  { email, password }            │                          │
   │────────────────────────────────►│                          │
   │                                 │  Find user by email      │
   │                                 │─────────────────────────►│
   │                                 │◄─────────────────────────│
   │                                 │                          │
   │                                 │  Compare password with   │
   │                                 │  stored bcrypt hash      │
   │                                 │                          │
   │                                 │  Generate JWT containing:│
   │                                 │  { user_id, username,    │
   │                                 │    exp: now + 7 days }   │
   │                                 │  Sign with SECRET_KEY    │
   │                                 │                          │
   │  { token: "eyJhbG..." }        │                          │
   │◄────────────────────────────────│                          │

Subsequent requests:
   │  GET /api/runs                  │
   │  Authorization: Bearer eyJhbG.. │
   │────────────────────────────────►│
   │                                 │  Decode JWT with SECRET  │
   │                                 │  Extract user_id         │
   │                                 │  Check exp > now         │
   │                                 │                          │
   │                                 │  If valid → proceed      │
   │                                 │  If invalid → 401        │
```

**Why JWT and not sessions?**
Sessions require the server to store session data (in memory or a database like Redis). JWT is self-contained — the token itself carries the user info. For a prototype with 10-15 users, either works, but JWT is simpler (no Redis to set up).

**Token expiry:** 7 days for prototype. The app stores the token locally. After 7 days, the user logs in again. For production, you'd add refresh tokens, but that's unnecessary complexity now.

### Password Hashing

```go
import "golang.org/x/crypto/bcrypt"

// When registering:
hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
// Store hash in DB. DefaultCost = 10 rounds. Takes ~100ms to hash.

// When logging in:
err := bcrypt.CompareHashAndPassword([]byte(storedHash), []byte(password))
// err == nil means match. err != nil means wrong password.
```

**Why bcrypt?** It's intentionally slow (~100ms per hash). This means even if someone steals your database, they can only try ~10 passwords per second per CPU core to brute-force a hash. With a plain hash like SHA256, they could try billions per second.

---

## 11. Error Handling & What Can Go Wrong

### Go Error Handling Pattern

Go doesn't have try/catch. Every function that can fail returns an error:

```go
user, err := repo.FindUserByEmail(email)
if err != nil {
    // Handle the error
    return fmt.Errorf("finding user: %w", err)  // Wrap with context
}
// user is valid here, continue
```

**The `%w` verb** wraps the original error so callers can inspect it later. Always wrap errors with context so when something fails, the error message tells you WHERE it failed: `"claiming cell: updating grid: database connection refused"` is way more useful than just `"connection refused"`.

### Specific Failure Scenarios

#### Database connection drops mid-run

```
Problem: User is running, sending GPS data every 5 seconds, and the DB goes down.

What happens:
1. POST /api/runs/:id/gps arrives
2. Service tries to write to DB → error
3. Handler returns 500 Internal Server Error to the app
4. App holds onto the GPS points (doesn't discard them)
5. App retries the same batch on next cycle (5 seconds later)
6. If DB is back → points are saved, run continues
7. If still down → app keeps accumulating points locally

Why this works: GPS points have timestamps. Even if they arrive late, the server
processes them in order. The run isn't "lost" — just delayed.
```

#### Two users claim the same cell simultaneously

```
Problem: User A and User B run through the same cell at the exact same time.

What happens without protection:
1. Both read: cell is unclaimed
2. Both try to INSERT → one gets a unique constraint violation
3. One request fails silently

Solution: Use INSERT ... ON CONFLICT (upsert):

INSERT INTO grid_cells (grid_row, grid_col, owner_id, claimed_at, bounds)
VALUES ($1, $2, $3, NOW(), $4)
ON CONFLICT (grid_row, grid_col)
DO UPDATE SET owner_id = $3, claimed_at = NOW();

This means: "Insert the cell. If it already exists (conflict on row+col),
update the owner instead." Last write wins. No errors, no race condition.
```

#### WebSocket connection drops

```
Problem: User's phone briefly loses connection. WebSocket dies.

What happens:
1. Hub detects dead connection (ping timeout or write error)
2. Hub removes client from the map
3. App detects disconnection, waits 1-2 seconds, reconnects
4. On reconnect, app calls GET /api/territory to refresh all visible cells
5. App is now back in sync

Why it's fine: Territory data is in the database. WebSocket is just a
"notification shortcut." If you miss a notification, you can always
reload from the REST API. For 10-15 users, the amount of missed
updates during a brief disconnection is tiny.
```

#### Server crashes and restarts

```
Problem: Go server process dies (bug, OOM, machine reboot).

What happens:
1. All WebSocket connections drop
2. All in-flight HTTP requests fail
3. Server restarts (systemd, Docker restart policy, etc.)
4. Apps reconnect WebSocket, retry failed GPS batches
5. Database is fine — it's a separate process with its own durability

What we lose: Nothing permanent. GPS points not yet saved are still on the
user's phone. The phone re-sends them. Active runs remain marked as "active"
in the DB — they continue when points start flowing again.

Safeguard: Add a "stale run cleanup" job. If a run has been "active" for
more than 4 hours with no GPS points, mark it as "discarded." This handles
the case where the app crashes and never sends the "end run" request.
```

#### GPS accuracy is terrible

```
Problem: User is indoors or in a dense urban area. GPS accuracy is 50-100m.

What happens without protection: Cells get claimed randomly around the
user's actual position. Feels broken.

Solution (already built into the pipeline):
- Filter out GPS points where accuracy > 30m
- The app should also warn the user: "GPS signal is weak"
- If ALL points in a batch are filtered out, return a response indicating
  no cells were claimed (don't error — the run is still active)
```

---

## 12. Configuration & Environment

### Environment Variables

```bash
# .env.example

# Server
PORT=8080                    # HTTP server port
ENV=development              # development | production

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=running_app
DB_PASSWORD=your_password    # Change in production!
DB_NAME=running_app
DB_SSLMODE=disable           # Use "require" in production

# Auth
JWT_SECRET=your-secret-key-change-this   # Used to sign/verify JWT tokens
JWT_EXPIRY=168h                           # 7 days (168 hours)

# Grid
GRID_CELL_SIZE=0.0005        # Degrees. ~55m at equator.
GPS_ACCURACY_THRESHOLD=30    # Meters. Points worse than this are dropped.

# WebSocket
WS_PING_INTERVAL=30s         # How often to ping clients
WS_PONG_TIMEOUT=10s          # How long to wait for pong before disconnecting
```

### Loading Config in Go

```go
// internal/config/config.go

type Config struct {
    Port               string
    DatabaseURL        string
    JWTSecret          string
    JWTExpiry          time.Duration
    GridCellSize       float64
    GPSAccuracyThreshold float64
}

func Load() (*Config, error) {
    // Read from environment variables
    // Use a library like "github.com/joho/godotenv" to load .env file
    // in development
}
```

---

## 13. Build Phases

### Phase 1 — Walking Skeleton (Week 1-2)

**Goal:** A running server that accepts GPS points and stores them. No grid, no WebSocket yet.

```
What you build:
  ✓ Go project setup with the folder structure above
  ✓ PostgreSQL + PostGIS running locally (Docker recommended)
  ✓ Database migrations for users and runs tables
  ✓ POST /api/auth/register and POST /api/auth/login
  ✓ JWT middleware
  ✓ POST /api/runs/start, POST /api/runs/:id/gps, POST /api/runs/:id/end
  ✓ GPS points stored in database
  ✓ Basic request logging middleware

What you can test:
  - Register a user via curl/Postman
  - Start a run
  - Send fake GPS points
  - End the run
  - See data in the database

Key learning:
  - Go project structure and packages
  - HTTP handlers in Go
  - Connecting Go to PostgreSQL
  - Writing SQL migrations
  - JWT authentication flow
```

### Phase 2 — Grid Territory System (Week 2-3)

**Goal:** GPS points now claim grid cells. Territory is queryable.

```
What you build:
  ✓ grid_cells and cell_claims tables
  ✓ Grid calculation service (GPS point → grid row/col)
  ✓ Line interpolation between consecutive GPS points
  ✓ Cell claiming logic with upsert (handles conflicts)
  ✓ GET /api/territory endpoint (cells in a viewport)
  ✓ GPS accuracy filtering
  ✓ Run summary stats (distance, cells claimed)

What you can test:
  - Send GPS points and verify correct cells are claimed
  - Query territory for a map area
  - Verify takeover works (claim a cell owned by someone else)
  - Check that inaccurate GPS points are filtered

Key learning:
  - PostGIS spatial queries
  - Grid math and line interpolation
  - Database transactions in Go
  - Upsert pattern (INSERT ON CONFLICT)
```

### Phase 3 — Real-Time with WebSocket (Week 3-4)

**Goal:** Connected users see territory changes in real-time.

```
What you build:
  ✓ WebSocket Hub (manage connections, broadcast messages)
  ✓ WebSocket endpoint (GET /ws with JWT auth)
  ✓ Territory update broadcasts when cells are claimed
  ✓ Player location broadcasts (optional — shows other runners)
  ✓ Ping/pong keepalive
  ✓ Connection cleanup on disconnect

What you can test:
  - Connect via WebSocket (use a tool like websocat or a simple HTML page)
  - Send GPS points via REST and see territory updates on WebSocket
  - Disconnect and reconnect — verify cleanup and re-sync

Key learning:
  - WebSocket protocol
  - Go channels and goroutines in practice
  - Concurrent connection management
  - Message broadcasting patterns
```

### Phase 4 — Polish & Leaderboard (Week 4-5)

**Goal:** Leaderboard, stale run cleanup, hardening.

```
What you build:
  ✓ GET /api/leaderboard
  ✓ GET /api/runs (run history for a user)
  ✓ Stale run cleanup (background goroutine or cron)
  ✓ Better error responses (consistent JSON error format)
  ✓ Request rate limiting (simple in-memory, e.g., 100 req/min per user)
  ✓ Graceful server shutdown (finish in-flight requests before stopping)
  ✓ Health check endpoint (GET /health)

What you can test:
  - Leaderboard shows correct rankings
  - Stale runs get cleaned up
  - Server shuts down gracefully
  - Health check works (useful for monitoring)

Key learning:
  - Background workers in Go
  - Graceful shutdown pattern
  - Rate limiting basics
  - API design best practices
```

---

## Key Libraries

| Library | What it does | Why we need it |
|---------|-------------|----------------|
| `net/http` (stdlib) | HTTP server | Built into Go. No framework needed for this scale. |
| `github.com/gorilla/mux` | HTTP router | Adds URL parameters (`/runs/:id`), method matching. stdlib router is too basic. |
| `github.com/gorilla/websocket` | WebSocket support | The standard Go WebSocket library. Handles upgrade, ping/pong, message framing. |
| `github.com/lib/pq` | PostgreSQL driver | Lets Go talk to PostgreSQL. |
| `github.com/golang-jwt/jwt/v5` | JWT tokens | Create and verify JWT tokens for auth. |
| `golang.org/x/crypto/bcrypt` | Password hashing | Secure password hashing. Part of Go's extended stdlib. |
| `github.com/joho/godotenv` | .env file loader | Reads `.env` file into environment variables. Dev convenience. |
| `github.com/rs/zerolog` | Structured logging | Fast JSON logger. Makes debugging easier than `fmt.Println`. |

**Why no web framework (like Gin or Echo)?**
For 7-8 endpoints and 10-15 users, a framework adds complexity without benefit. `gorilla/mux` + `net/http` is plenty. You'll learn more about how Go HTTP actually works this way, which helps when you eventually use a framework.

---

## Quick Reference: Key Decisions Summary

| Decision | Choice | Why |
|----------|--------|-----|
| Architecture | Monolith (single binary) | 10-15 users. Microservices would be absurd overkill. |
| Database | Single PostgreSQL instance | Simple. PostGIS needs PostgreSQL anyway. |
| Grid math | Computed in Go code | Faster than querying PostGIS for every GPS point. PostGIS for viewport queries only. |
| GPS transport | REST (not WebSocket) | Need request/response semantics and retry reliability. |
| Territory updates | WebSocket broadcast | Server pushes to all clients. No polling. |
| Auth | JWT (no sessions) | No Redis/session store needed. Self-contained tokens. |
| Grid cells | Lazy creation | Only create rows when claimed. Don't pre-populate. |
| Conflict resolution | Last writer wins (upsert) | Simple. No race conditions. Fair for a game mechanic. |
| ORM | None (raw SQL) | For a beginner, understanding the actual SQL is more valuable. ORMs add a layer of abstraction that hides what's happening. |
| Caching | None | 10-15 users. Database handles this load trivially. |
