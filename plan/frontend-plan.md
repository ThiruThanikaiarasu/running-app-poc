# Android Frontend Plan — Territory Running App

> This document is both an architecture plan and a learning guide. If you are new to Kotlin and Android development, every concept is explained from scratch. Nothing is assumed.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Why These Technologies](#2-why-these-technologies)
3. [Project Structure](#3-project-structure)
4. [Core Components Explained](#4-core-components-explained)
5. [Screen-by-Screen Breakdown](#5-screen-by-screen-breakdown)
6. [Data Flow Diagrams](#6-data-flow-diagrams)
7. [Local Data Storage](#7-local-data-storage)
8. [Key Libraries and Dependencies](#8-key-libraries-and-dependencies)
9. [Error Handling and Edge Cases](#9-error-handling-and-edge-cases)
10. [Build Phases](#10-build-phases)

---

## 1. Project Overview

### What we are building

An Android app that lets a runner open a map, start a run, and watch grid cells on the map light up in their colour as they run through them. Other players' claimed cells show in different colours. Everything updates in real time — if another runner steals one of your cells while you are both out running, you see it happen live.

### How the Android app talks to the backend

The app communicates with a Go backend in two ways:

```
Android App                          Go Backend
-----------                          ----------

  1. REST API (HTTP)  ──────────>    Handles: login, register, fetch run history,
     Request/response                get leaderboard, submit completed run
     (ask a question, get answer)

  2. WebSocket  <──────────────>     Handles: live territory updates during a run,
     Persistent two-way pipe         real-time cell ownership changes,
     (stays open, both sides         "someone just claimed your cell" notifications
      can send messages anytime)
```

**REST API** is like sending a letter — you write a request, send it, and wait for a reply. Used for things that are not time-sensitive (loading your profile, fetching your run history).

**WebSocket** is like a phone call — once connected, both sides can talk anytime without having to "dial" again. Used for real-time updates during a run, because you need territory changes to show up instantly, not after a request-response round trip.

The backend stores everything in **PostgreSQL with PostGIS** (a spatial database that understands geography). The Android app keeps a local copy of relevant data using **Room** (a local database on the phone) so the app works even when the network drops.

---

## 2. Why These Technologies

### Kotlin — What it is and why we use it for Android

**What it is:** Kotlin is a programming language made by JetBrains (the company behind IntelliJ IDEA). Google officially declared it the preferred language for Android development in 2019. Before Kotlin, Android apps were written in Java. Kotlin runs on the same platform as Java (the JVM — Java Virtual Machine) but has a cleaner, more modern syntax.

**Why Kotlin and not something else:**

| Option | Why not |
|---|---|
| Java | Verbose (more typing for the same result), no coroutines (harder async code), Kotlin is the future of Android |
| Flutter (Dart) | Cross-platform framework. GPS background tracking and Mapbox integration require heavy platform-specific code anyway, so the "write once" benefit breaks down for this app |
| React Native (JavaScript) | Same problem as Flutter — background location and map performance suffer behind abstraction layers |
| Kotlin | Modern, concise, null-safe (prevents a whole class of crashes), has coroutines for clean async code, first-class Android support from Google |

**Key Kotlin concepts you will encounter:**

- **Coroutines** — Kotlin's way of handling asynchronous work (like network calls or GPS updates) without freezing the UI. Think of them as lightweight threads that can pause and resume. When you see `suspend fun` in Kotlin, it means "this function might take a while, but it will not block everything else while it waits."
- **Flow** — A stream of data that arrives over time. Perfect for GPS coordinates that keep coming every second, or WebSocket messages that arrive unpredictably. You "collect" from a Flow the way you would listen to a radio station — data keeps arriving and you process each piece as it comes.
- **Null safety** — Kotlin forces you to explicitly handle cases where a value might be missing. `String` means "definitely a string." `String?` means "maybe a string, maybe nothing." This catches bugs at compile time instead of crashing at runtime.

### Jetpack Compose — the UI framework

**What it is:** Jetpack Compose is Android's modern UI toolkit. Instead of building screens by dragging and dropping widgets in an XML layout file (the old way), you write Kotlin functions that describe what the screen should look like. When data changes, Compose automatically redraws only the parts of the screen that changed.

**Why it matters for this app:** Territory ownership changes in real time. When a WebSocket message arrives saying "cell X is now owned by player B," Compose will automatically update the relevant part of the screen without you manually finding and updating UI elements.

**The Mapbox catch:** Mapbox's Android SDK does not have native Compose components yet. We wrap the Mapbox MapView inside a Compose `AndroidView` — this is a bridge that lets old-style Android Views live inside a Compose screen. It works fine; it just means the map code looks slightly different from the rest of the UI code.

### Mapbox GL — What it does and why not Google Maps

**What it is:** Mapbox GL is a map rendering library that draws maps using vector tiles (mathematical descriptions of shapes) instead of raster tiles (pre-drawn images). This means the map can be styled, rotated, tilted, and customized at every level — road colours, building shapes, label fonts, everything.

**Why Mapbox over Google Maps:**

| Need | Mapbox | Google Maps |
|---|---|---|
| Custom territory colours per player | Full style control — any colour, opacity, glow | Limited polygon overlay styling |
| Grid overlay (thousands of small squares) | Custom GeoJSON source layers, very performant | Basic overlay, performance degrades with many shapes |
| Dark/game-like map theme | Full custom style via Mapbox Studio | Limited dark mode, cannot fully restyle |
| Offline map support | Built-in offline packs | Hacky workarounds required |
| Cost at prototype scale | Free for 25,000 map loads/month | Free tier then per-load billing |
| 3D territory extrusion | Built-in fill-extrusion layer | Not available for custom polygons |

For a territory game, the map IS the game board. We need full visual control, and Mapbox gives that.

**How Mapbox works on Android (simplified):**

```
1. You add a MapView to your screen layout
2. Mapbox downloads vector tile data for the visible area from its servers
3. The GPU on the phone renders those tiles into the map you see
4. You add "sources" (data) and "layers" (visual rules) on top:
   - Source: a GeoJSON object describing all grid cells and their owners
   - Layer: a fill layer that colours each cell based on owner
5. When data changes (someone claims a cell), you update the source
   and Mapbox re-renders only what changed
```

### WebSocket — How it works from the client side

**What it is:** WebSocket is a communication protocol that upgrades a regular HTTP connection into a persistent, two-way channel. Once opened, both the client (your phone) and the server can send messages at any time without establishing a new connection each time.

**How it works step by step:**

```
1. App opens an HTTP connection to the server:
   GET /ws HTTP/1.1
   Upgrade: websocket

2. Server agrees to upgrade:
   HTTP/1.1 101 Switching Protocols

3. Now both sides have an open pipe:

   Phone  ----"I claimed cell (45, 120)"---->  Server
   Phone  <---"Player B claimed cell (44, 121)"----  Server
   Phone  <---"Player C claimed cell (45, 119)"----  Server
   Phone  ----"I claimed cell (45, 121)"---->  Server

   Messages flow freely in both directions.
   No need to "ask" for updates — they arrive when they happen.

4. When the run ends or the app backgrounds, the WebSocket closes.
```

**Why not just use REST for everything?** With REST, the phone would have to keep asking "any new territory updates?" every second (polling). That wastes battery, bandwidth, and adds latency. With WebSocket, updates arrive the instant they happen on the server.

---

## 3. Project Structure

### What is "clean architecture" and why bother?

Clean architecture is a way of organising code so that each piece has one job and knows as little as possible about the other pieces. The core idea is **layers**:

```
Outer layer:  UI (screens, buttons, maps)
              Depends on everything below.

Middle layer: Domain (business logic, rules)
              Knows nothing about the UI or the database.
              "A run is a list of GPS points with a start and end time."

Inner layer:  Data (network calls, database, GPS sensor)
              Knows how to fetch/store things but not how to display them.
```

**Why this matters for a beginner:** When something breaks, you know where to look. GPS giving bad data? Look in the data layer. Map not showing territory? Look in the UI layer. Cell ownership logic wrong? Look in the domain layer. Without this separation, everything bleeds together and a GPS bug can break your login screen.

**Why we are keeping it simple:** Full clean architecture in production Android apps uses dependency injection frameworks (like Dagger or Hilt), interface abstractions for every component, and multiple modules. For a prototype with 10-15 users, that is over-engineering. We will use the folder structure and naming conventions of clean architecture, but skip the framework overhead. If this goes to production, you would add Hilt for dependency injection and split into Gradle modules.

### Folder layout

```
app/
 src/
  main/
   java/com/territoryrun/app/
   |
   |-- TerritoryRunApp.kt              # Application class (app-wide setup)
   |
   |-- data/                           # DATA LAYER — talks to the outside world
   |   |-- api/                        # Network communication
   |   |   |-- ApiService.kt           # REST API definitions (Retrofit interface)
   |   |   |-- ApiClient.kt            # Creates and configures the HTTP client
   |   |   |-- WebSocketClient.kt      # Manages the WebSocket connection
   |   |   |-- models/                 # Data shapes for JSON responses
   |   |       |-- RunResponse.kt
   |   |       |-- TerritoryUpdate.kt
   |   |       |-- UserResponse.kt
   |   |       |-- LeaderboardResponse.kt
   |   |
   |   |-- local/                      # Local database (Room)
   |   |   |-- AppDatabase.kt          # Room database definition
   |   |   |-- dao/                    # Data Access Objects — SQL queries as Kotlin functions
   |   |   |   |-- RunDao.kt
   |   |   |   |-- TerritoryDao.kt
   |   |   |   |-- UserDao.kt
   |   |   |-- entity/                 # Database table definitions
   |   |       |-- RunEntity.kt
   |   |       |-- GpsPointEntity.kt
   |   |       |-- TerritoryCellEntity.kt
   |   |
   |   |-- location/                   # GPS tracking
   |   |   |-- LocationTracker.kt      # Wraps the Fused Location Provider
   |   |   |-- LocationService.kt      # Foreground service for background GPS
   |   |
   |   |-- repository/                 # Bridges between data sources and the rest of the app
   |       |-- RunRepository.kt        # Combines local DB + API for run data
   |       |-- TerritoryRepository.kt  # Combines local cache + WebSocket for territory
   |       |-- UserRepository.kt       # Auth, profile
   |
   |-- domain/                         # DOMAIN LAYER — business logic, no Android imports
   |   |-- model/                      # Core data types
   |   |   |-- Run.kt                  # What a "run" is (id, points, distance, duration)
   |   |   |-- GpsPoint.kt            # A single GPS coordinate with timestamp
   |   |   |-- GridCell.kt            # A single grid cell (row, col, owner)
   |   |   |-- User.kt                # A user (id, name, colour)
   |   |   |-- LeaderboardEntry.kt
   |   |
   |   |-- grid/                       # Grid math — convert GPS coords to cell IDs
   |   |   |-- GridCalculator.kt       # The core "which cell is this point in?" logic
   |   |
   |   |-- util/                       # Pure utility functions
   |       |-- DistanceCalculator.kt   # Haversine formula for GPS distance
   |       |-- GpsFilter.kt           # Filter out GPS drift/noise
   |
   |-- ui/                             # UI LAYER — screens and visual components
   |   |-- theme/                      # Colours, fonts, spacing
   |   |   |-- Theme.kt
   |   |   |-- Color.kt
   |   |   |-- Type.kt
   |   |
   |   |-- navigation/                 # Screen navigation setup
   |   |   |-- NavGraph.kt            # Defines which screens exist and how to move between them
   |   |   |-- Screen.kt              # Enum/sealed class of all screens
   |   |
   |   |-- screens/
   |   |   |-- map/                    # Main map screen
   |   |   |   |-- MapScreen.kt       # Compose UI
   |   |   |   |-- MapViewModel.kt    # State holder — connects UI to data
   |   |   |   |-- MapboxMapWrapper.kt # AndroidView bridge for Mapbox
   |   |   |
   |   |   |-- activerun/             # During a run
   |   |   |   |-- ActiveRunScreen.kt
   |   |   |   |-- ActiveRunViewModel.kt
   |   |   |
   |   |   |-- history/               # Past runs
   |   |   |   |-- HistoryScreen.kt
   |   |   |   |-- HistoryViewModel.kt
   |   |   |
   |   |   |-- leaderboard/           # Rankings
   |   |   |   |-- LeaderboardScreen.kt
   |   |   |   |-- LeaderboardViewModel.kt
   |   |   |
   |   |   |-- auth/                  # Login and register
   |   |       |-- LoginScreen.kt
   |   |       |-- RegisterScreen.kt
   |   |       |-- AuthViewModel.kt
   |   |
   |   |-- components/                 # Reusable UI pieces
   |       |-- TerritoryStatsBar.kt   # Shows cells owned, distance, etc.
   |       |-- RunTimer.kt            # Elapsed time display during a run
   |       |-- PlayerMarker.kt        # Other player dots on the map
   |
   |-- util/                           # App-wide utilities
       |-- PrefsManager.kt            # SharedPreferences wrapper (auth token, settings)
       |-- Constants.kt               # API URLs, grid size, magic numbers
       |-- NetworkMonitor.kt          # Watches for internet connectivity changes

   res/
    |-- values/
    |   |-- strings.xml                # All user-facing text
    |   |-- colors.xml
    |-- drawable/                      # Icons, images
    |-- raw/                           # Mapbox style JSON (if custom)
```

### What each folder does — the short version

| Folder | Job | Example |
|---|---|---|
| `data/api/` | Talks to the Go backend over HTTP and WebSocket | "Send my GPS points to the server" |
| `data/local/` | Stores data on the phone in a SQLite database (via Room) | "Save this run locally in case we lose network" |
| `data/location/` | Gets GPS coordinates from the phone's hardware | "Where am I right now?" |
| `data/repository/` | Decides whether to get data from the network or local DB | "Get territory — check cache first, then network" |
| `domain/model/` | Defines what things ARE (data shapes with no logic about how to store or display them) | "A Run has an id, a list of GpsPoints, a duration, and a distance" |
| `domain/grid/` | Pure math for the grid system | "GPS point (13.0827, 80.2707) falls in grid cell row=4521, col=8923" |
| `ui/screens/` | What the user sees and taps | "Show the map with coloured grid cells" |
| `ui/components/` | Small reusable UI pieces used across screens | "A stats bar showing '42 cells owned'" |

### What is a ViewModel?

You will see a `ViewModel` file paired with almost every screen. A ViewModel is a class that:

1. Holds the data the screen needs (the "state")
2. Survives screen rotations (when you rotate the phone, the screen is destroyed and recreated, but the ViewModel persists)
3. Talks to repositories to fetch/save data
4. Exposes data as `StateFlow` (a reactive stream the Compose screen observes)

```
User taps "Start Run"
       |
       v
ActiveRunScreen.kt  -->  ActiveRunViewModel.kt  -->  RunRepository.kt  -->  LocationTracker + API
   (UI layer)              (holds state)              (data layer)          (hardware + network)
```

The screen never talks directly to the network or database. It only talks to its ViewModel. This keeps the UI code simple and testable.

---

## 4. Core Components Explained

### 4.1 Location Tracking Service

#### How Android tracks GPS in the background

When your app is open and on screen (in the "foreground"), getting GPS updates is straightforward — you ask the system for location updates and they arrive. The problem is: runners put their phone in their pocket. The app goes to the background. And Android, to save battery, aggressively kills background apps.

#### What is a Foreground Service?

A Foreground Service is Android's way of saying "this app is doing something important right now — do not kill it." It requires showing a persistent notification (the one you see in fitness apps that says "Recording your run..."). This notification is not optional — Android demands it as a signal to the user that an app is actively running.

```
Without Foreground Service:
  User starts run → puts phone in pocket → Android says
  "this app is idle, kill it to save battery" → GPS tracking stops → run data lost

With Foreground Service:
  User starts run → Foreground Service starts → persistent notification appears →
  phone goes in pocket → Android sees the notification and says
  "this app declared it is doing active work, leave it alone" →
  GPS tracking continues → run data captured
```

#### How it works in our app

```kotlin
// Simplified — what LocationService.kt does:

class LocationService : Service() {

    // Called when the service starts
    override fun onStartCommand(...) {
        // 1. Show the persistent notification ("Recording your run...")
        startForeground(notificationId, notification)

        // 2. Start requesting GPS updates
        locationClient.requestLocationUpdates(
            interval = 2000,          // every 2 seconds
            priority = HIGH_ACCURACY  // use GPS satellite, not just WiFi/cell
        )

        // 3. Each GPS point that arrives gets:
        //    a) Saved to local Room database
        //    b) Fed to GridCalculator to determine which cell it is in
        //    c) Sent to server via WebSocket (if connected)
    }
}
```

#### Battery considerations

GPS is the single biggest battery drain on a phone. Strategies to manage it:

| Strategy | What it means | When to use |
|---|---|---|
| High accuracy, 2-second interval | Full GPS power, best tracking | During an active run |
| Reduced accuracy, 10-second interval | Less GPS power, rougher track | If we ever need passive tracking (not for POC) |
| Stop updates entirely | No GPS drain at all | When not running — turn it off completely |

For the prototype, we keep it simple: GPS is ON during a run (high accuracy, 2-second intervals) and OFF otherwise. No background tracking when the user is not actively running.

#### What happens when GPS is inaccurate

GPS drift is when your reported position jumps around even though you are standing still or running in a straight line. This happens near tall buildings, under trees, or with cheap phone GPS chips.

**How we handle it (GpsFilter.kt):**

```
Raw GPS points from phone:
  Point 1: (13.0827, 80.2707)  ← OK
  Point 2: (13.0828, 80.2708)  ← OK (moved ~15 meters, reasonable)
  Point 3: (13.0900, 80.2800)  ← SUSPICIOUS (jumped 900 meters in 2 seconds — impossible)
  Point 4: (13.0829, 80.2709)  ← OK (back to reasonable position)

Filter logic:
  - Calculate speed between consecutive points
  - Human max sprint speed ≈ 12 m/s (43 km/h)
  - If calculated speed > 15 m/s → discard the point as GPS noise
  - If reported accuracy from Android > 25 meters → mark as low confidence
  - Minimum displacement filter: ignore points < 3 meters from last accepted point
    (reduces jitter when standing still or running slowly)

Filtered output:
  Point 1: (13.0827, 80.2707)  ← kept
  Point 2: (13.0828, 80.2708)  ← kept
  Point 3: DISCARDED            ← impossible speed
  Point 4: (13.0829, 80.2709)  ← kept
```

This is a simple but effective approach for a prototype. Production apps use Kalman filters (a more sophisticated statistical smoothing algorithm) and snap-to-road (align GPS points to known roads using OpenStreetMap data). Those are out of scope for the POC.

---

### 4.2 Map Rendering

#### How Mapbox GL works on Android

Mapbox GL renders maps using the phone's GPU (graphics processing unit — the same chip that renders games). It downloads vector tiles (compact mathematical descriptions of roads, buildings, water, etc.) and draws them in real time. This is why you can rotate, tilt, and zoom the map smoothly — it is being rendered like a 3D scene, not just showing pre-drawn images.

#### How we render the grid overlay

The grid overlay is a collection of small square polygons drawn on top of the base map. Here is how it works:

```
Step 1: Mapbox loads and displays the base map (roads, buildings, etc.)

Step 2: We create a GeoJSON source containing grid cell data:
  {
    "type": "FeatureCollection",
    "features": [
      {
        "type": "Feature",
        "geometry": {
          "type": "Polygon",
          "coordinates": [[[80.270, 13.082], [80.271, 13.082],
                           [80.271, 13.083], [80.270, 13.083],
                           [80.270, 13.082]]]
        },
        "properties": {
          "owner": "player_a",
          "color": "#FF5733"
        }
      },
      ... more cells ...
    ]
  }

Step 3: We add a fill layer that reads the "color" property and fills each cell:
  - Owned by you → your colour (e.g., blue), semi-transparent
  - Owned by another player → their colour (e.g., red), semi-transparent
  - Unclaimed → not rendered (transparent, shows base map)

Step 4: When territory changes (via WebSocket), we update the GeoJSON source.
  Mapbox detects the change and re-renders only the affected cells.
```

#### How claimed territories get coloured

Each player has an assigned colour. The server sends this as part of user data. The map layer uses a data-driven style expression:

```
"fill-color": ["get", "color"]    // Read the "color" property from each cell's data
"fill-opacity": 0.4               // 40% transparent so the map is still visible underneath
"fill-outline-color": "#000000"   // Thin black border around each cell
```

#### How to show other players' territories

All territory data comes from the server. When the map loads, we fetch the current territory state for the visible area via REST API. During a run, the WebSocket pushes real-time updates. The GeoJSON source contains cells from ALL players — Mapbox colours them based on the `owner` property.

```
What the user sees on the map:
+---+---+---+---+---+---+
|   |   |BBB|BBB|   |   |     B = Blue (you)
+---+---+---+---+---+---+     R = Red (opponent)
|   |BBB|BBB|   |RRR|   |     empty = unclaimed
+---+---+---+---+---+---+
|   |BBB|   |   |RRR|RRR|
+---+---+---+---+---+---+
|   |   |   |RRR|RRR|   |
+---+---+---+---+---+---+
```

#### Performance: not rendering the entire world

We only load and render grid cells for the visible map area (the "viewport"), plus a small buffer around it. As the user pans the map, we fetch cells for the new area. This keeps memory usage manageable.

```
Phone screen shows a ~1km x 1km area.
Grid cells are ~50m x 50m.
That is about 20 x 20 = 400 cells visible at once.
400 small polygons is trivial for Mapbox GL to render.

If we tried to load ALL cells for an entire city: millions of polygons → phone crashes.
Load only what is visible → fast and smooth.
```

---

### 4.3 WebSocket Client

#### How the app maintains a persistent connection

The WebSocket connection is managed by `WebSocketClient.kt`. Here is the lifecycle:

```
1. User opens the app
   → REST API calls work (login, load initial data)
   → WebSocket NOT connected yet (saves battery)

2. User taps "Start Run"
   → WebSocket connects to ws://server/ws?token=<auth_token>
   → Server authenticates and joins the user to a geographic "room"
     (only players in the same area receive each other's updates)

3. During the run
   → Phone sends: cell claims (which cells the runner passed through)
   → Server sends: other players' cell claims, territory ownership changes

4. User taps "Stop Run"
   → WebSocket disconnects cleanly
   → Final run data sent via REST API

5. User browses the app (not running)
   → WebSocket stays disconnected
   → Territory data loaded via REST API (cached, not real-time)
```

#### What happens when the connection drops

Mobile networks are unreliable. The runner goes through a tunnel, signal drops, connection dies. Our strategy:

```
WebSocket connection drops:
  |
  v
Detect disconnection (OkHttp WebSocket listener fires onFailure/onClosed)
  |
  v
Start reconnection timer:
  Attempt 1: wait 1 second, try to reconnect
  Attempt 2: wait 2 seconds
  Attempt 3: wait 4 seconds
  Attempt 4: wait 8 seconds
  Attempt 5: wait 16 seconds
  ... (exponential backoff, max 30 seconds between attempts)
  |
  v
While disconnected:
  - GPS tracking CONTINUES (foreground service is independent of network)
  - GPS points are saved to local Room database
  - Cell claims are queued locally
  - UI shows a small "reconnecting..." indicator
  |
  v
When reconnected:
  - Send all queued cell claims to the server
  - Server responds with any territory changes we missed
  - Map updates to reflect current state
```

**Why exponential backoff?** If the server is down, hammering it with reconnection attempts every second makes the problem worse. By waiting longer between each attempt (1s, 2s, 4s, 8s...), we give the server time to recover without flooding it. The "exponential" part means the wait time doubles each time.

#### How real-time territory updates flow

```
Runner A claims cell (45, 120):

  Runner A's phone                    Server                    Runner B's phone
  ─────────────────                   ──────                    ─────────────────
  GPS point received
  GridCalculator: "this is cell (45,120)"
  Send via WebSocket:
    {"type":"claim","cell":[45,120]}
                          ───────>
                                      Validate claim
                                      Update database
                                      Broadcast to room:
                                        {"type":"territory_update",
                                         "cell":[45,120],
                                         "owner":"runner_a",
                                         "color":"#3366FF"}
                          <───────                    ───────>
  Update local state                                  Update local state
  (already blue — we                                  Cell (45,120) turns
   optimistically coloured                            blue on Runner B's map
   it when we claimed it)
```

**Optimistic updates:** When you run through a cell, we immediately colour it on your map (before the server confirms). This makes the app feel instant. If the server rejects the claim (rare), we revert the colour. This is called "optimistic UI" — assume success, correct on failure.

---

### 4.4 Run Recording

#### How we capture a run: start to stop

```
┌─────────────────────────────────────────────────────────┐
│                    RUN LIFECYCLE                         │
│                                                         │
│  IDLE ──[Tap Start]──> RUNNING ──[Tap Stop]──> SAVING   │
│   │                      │                       │      │
│   │                      │ Every 2 seconds:      │      │
│   │                      │  • Get GPS point      │      │
│   │                      │  • Filter for noise   │      │
│   │                      │  • Save to local DB   │      │
│   │                      │  • Calculate cell ID  │      │
│   │                      │  • Send via WebSocket  │      │
│   │                      │  • Update map UI      │      │
│   │                      │                       │      │
│   │                      │ Every 5 seconds:      │      │
│   │                      │  • Update distance    │      │
│   │                      │  • Update pace        │      │
│   │                      │  • Update timer       │      │
│   │                      │                       │      │
│   │                      v                       │      │
│   │                  [Tap Pause]                  │      │
│   │                      │                       │      │
│   │                      v                       │      │
│   │                   PAUSED                     │      │
│   │                      │                       │      │
│   │                  [Tap Resume]                 │      │
│   │                      │                       │      │
│   │                      v                       │      │
│   │                   RUNNING                    │      │
│   │                                              │      │
│   │                                              v      │
│   │                                      Upload run to  │
│   │                                      server (REST)  │
│   │                                              │      │
│   │                                      ┌───────┴──┐   │
│   │                                      │ Success?  │   │
│   │                                      └───┬───┬──┘   │
│   │                                      YES │   │ NO   │
│   │                                          v   v      │
│   │                                     DONE  QUEUED    │
│   │                                           (retry    │
│   │                                            later)   │
│   │                                              │      │
│   v                                              v      │
│  IDLE <──────────────────────────────────────  IDLE     │
└─────────────────────────────────────────────────────────┘
```

#### Storing GPS points locally

Every GPS point received during a run is immediately saved to the local Room database. This is the safety net: even if the app crashes, the network dies, or the phone reboots, the GPS data is persisted on disk.

```
Room database table: gps_points

| id  | run_id | latitude  | longitude | altitude | accuracy | timestamp         |
|-----|--------|-----------|-----------|----------|----------|-------------------|
| 1   | run_42 | 13.08270  | 80.27070  | 12.5     | 4.2      | 2026-03-26T06:00:00 |
| 2   | run_42 | 13.08280  | 80.27080  | 12.3     | 3.8      | 2026-03-26T06:00:02 |
| 3   | run_42 | 13.08290  | 80.27090  | 12.1     | 5.1      | 2026-03-26T06:00:04 |
```

#### Sending to server

During the run, cell claims are sent in real time via WebSocket. When the run ends, the complete run (all GPS points, duration, distance) is sent to the server via REST API as a single payload. The server uses this to:
- Validate the run (check for GPS spoofing, impossible speeds)
- Permanently record the run in the database
- Finalize territory ownership

#### What happens if network drops mid-run

```
Scenario: You are 15 minutes into a run. Network drops.

What KEEPS working:
  ✓ GPS tracking (foreground service, independent of network)
  ✓ Local GPS point storage (Room database, on-device)
  ✓ Grid cell calculation (pure math, no network needed)
  ✓ Map display (Mapbox caches tiles, visible area still renders)
  ✓ Your own territory colouring (optimistic updates from local data)

What STOPS working:
  ✗ Real-time updates from other players
  ✗ Cell claim messages to the server
  ✗ Seeing new territory changes from opponents

What we DO about it:
  1. Queue all cell claims locally
  2. Show "No connection" indicator on screen
  3. Keep running, keep tracking
  4. When network returns:
     - Reconnect WebSocket
     - Flush queued cell claims to server
     - Server sends back any updates we missed
  5. If network never returns during the run:
     - User stops the run
     - Complete run saved locally
     - Next time app has network: upload the run via REST API
     - Server processes it and awards territory retroactively
```

---

### 4.5 Grid System (Client Side)

#### How the client knows which grid cell a GPS coordinate falls into

The grid system divides the world into a uniform grid of square cells. Each cell has a fixed size (for the prototype, we will use approximately 50 meters x 50 meters). Given any GPS coordinate, we can instantly calculate which cell it belongs to using simple math.

**The concept:**

```
The Earth's surface (simplified to a flat plane for a small area):

  Latitude increases going north  ↑
  Longitude increases going east  →

  We define:
    - An origin point (e.g., 0.0 lat, 0.0 lng — or a local offset for our city)
    - A cell size in degrees (approximately 0.00045 degrees ≈ 50 meters at Chennai's latitude)

  Cell calculation:
    cell_row = floor(latitude / cell_size)
    cell_col = floor(longitude / cell_size)

  Example:
    GPS point: (13.08270, 80.27070)
    Cell size: 0.00045 degrees

    cell_row = floor(13.08270 / 0.00045) = floor(29072.67) = 29072
    cell_col = floor(80.27070 / 0.00045) = floor(178379.33) = 178379

    This point is in cell (29072, 178379).
```

**Why this works:** The floor function (round down to nearest integer) ensures that all points within the same 50m square map to the same cell ID. It is an O(1) operation — two divisions, done.

**The catch — cell size varies with latitude:** One degree of longitude covers different physical distances at different latitudes (because the Earth is a sphere, and lines of longitude converge at the poles). At the equator, one degree of longitude is about 111 km. At 60 degrees north (Helsinki), it is about 55 km. For our prototype focused on one city, this distortion is minimal and we can ignore it. For a production app, you would use a proper geospatial grid system like Uber's H3 (hexagonal cells that handle this correctly).

**GridCalculator.kt — what it does:**

```kotlin
// Simplified
object GridCalculator {

    // Cell size in degrees (≈50 meters at typical latitudes in India)
    private const val CELL_SIZE = 0.00045

    fun cellForPoint(lat: Double, lng: Double): GridCell {
        val row = floor(lat / CELL_SIZE).toInt()
        val col = floor(lng / CELL_SIZE).toInt()
        return GridCell(row, col)
    }

    fun cellBounds(cell: GridCell): CellBounds {
        // Returns the four corners of a cell — needed for rendering on the map
        val south = cell.row * CELL_SIZE
        val west = cell.col * CELL_SIZE
        val north = south + CELL_SIZE
        val east = west + CELL_SIZE
        return CellBounds(south, west, north, east)
    }
}
```

#### How to render the grid efficiently on the map

We do NOT draw a grid for the entire visible area. We only draw cells that have an owner (claimed cells). Unclaimed cells are invisible — the base map shows through.

```
Approach 1 (BAD): Draw 10,000 empty cells with faint borders
  → Slow rendering, cluttered visually, wastes GPU

Approach 2 (GOOD): Only draw claimed cells
  → Maybe 50-100 cells visible at any time
  → Clean look, fast rendering
  → Unclaimed area is just the normal map
```

When the server sends territory data, it includes only claimed cells. The client converts each cell to a GeoJSON polygon (using `cellBounds`) and adds it to the Mapbox source layer.

---

## 5. Screen-by-Screen Breakdown

### 5.1 Map Screen (Main Screen)

This is the home screen of the app. It fills the entire screen with a Mapbox map showing the user's current area with territory overlays.

**What the user sees:**
```
┌─────────────────────────────────────┐
│  [Profile icon]    Territory Run  🔍 │  ← Top bar (minimal)
├─────────────────────────────────────┤
│                                     │
│         ┌───┬───┐                   │
│         │ B │ B │     R = Red       │
│     ┌───┼───┼───┤     B = Blue     │  ← Mapbox map with
│     │ B │ B │   │                   │     coloured grid cells
│     ├───┼───┘   │                   │     overlaid on streets
│     │   │   ┌───┤                   │
│     │   │   │ R │                   │
│     └───┘   ├───┤                   │
│             │ R │                   │
│    [you]    └───┘                   │  ← Blue dot = your location
│       *                             │
│                                     │
├─────────────────────────────────────┤
│  Cells: 12    Area: 0.03 km²       │  ← Stats bar
├─────────────────────────────────────┤
│                                     │
│        [ START RUN ]                │  ← Big start button
│                                     │
├────────┬──────────┬─────────────────┤
│  Map   │ History  │  Leaderboard    │  ← Bottom navigation
└────────┴──────────┴─────────────────┘
```

**Key UI components:**
- **Mapbox MapView** — the map itself, wrapped in `AndroidView` for Compose compatibility
- **Territory overlay** — GeoJSON fill layer showing claimed cells
- **Your location dot** — Mapbox's built-in location puck (blue dot showing where you are)
- **Stats bar** — your total cells owned, total area
- **Start Run button** — large, prominent, begins a run session
- **Bottom navigation** — switches between Map, History, and Leaderboard tabs

**Data this screen needs:**
- Current GPS position (for centering the map)
- Territory cells for the visible area (from server, cached locally)
- User's total stats (cells owned, area)

---

### 5.2 Active Run Screen (During a Run)

This screen replaces the main map screen while a run is in progress. The map is still there, but the UI overlays change to show run stats.

**What the user sees:**
```
┌─────────────────────────────────────┐
│  ● RECORDING          [Pause] [Stop]│  ← Red dot = recording indicator
├─────────────────────────────────────┤
│                                     │
│         ┌───┬───┐                   │
│         │ B │ B │  ← cells you just │
│     ┌───┼───┼───┤     claimed turn  │  ← Map (same as before but
│     │ B │ B │ * │     blue as you   │     camera follows your
│     ├───┼───┘ | │     run through   │     position)
│     │   │     | │     them          │
│     │   │     | │                   │
│     └───┘     | │                   │
│               * ← your position     │
│         (moving dot)                │
│                                     │
├─────────────────────────────────────┤
│  0:14:32          2.3 km            │  ← Duration    Distance
│  5'42"/km         +4 cells          │  ← Pace        Cells claimed this run
├─────────────────────────────────────┤
│  ⚡ Connection OK                    │  ← Network status
│  (or: ⚠ Reconnecting...)            │     (important for user confidence)
└─────────────────────────────────────┘
```

**Key UI components:**
- **Recording indicator** — pulsing red dot so the user knows GPS is active
- **Map with auto-follow** — camera tracks the runner's position, cells light up as they are claimed
- **Run stats overlay** — duration (live timer), distance, pace, cells claimed this run
- **Pause/Stop buttons** — pause temporarily or end the run
- **Network status** — small indicator showing if the WebSocket is connected. Critical for user trust: "yes, your territory claims are being sent"

**Data this screen needs:**
- Live GPS stream (from Foreground Service)
- Run state (duration, distance, pace — computed from GPS points)
- Cells claimed this run (from GridCalculator)
- WebSocket connection status
- Other players' territory updates (from WebSocket)

---

### 5.3 Run History / Stats

A scrollable list of past runs with summary stats.

**What the user sees:**
```
┌─────────────────────────────────────┐
│  Run History                        │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐    │
│  │ Today, 6:02 AM              │    │
│  │ 3.2 km  •  28:14  •  12 cells│   │  ← Each run is a card
│  │ [mini map of route]         │    │     with key stats
│  └─────────────────────────────┘    │
│                                     │
│  ┌─────────────────────────────┐    │
│  │ Yesterday, 6:15 AM          │    │
│  │ 5.1 km  •  42:08  •  23 cells│   │
│  │ [mini map of route]         │    │
│  └─────────────────────────────┘    │
│                                     │
│  ┌─────────────────────────────┐    │
│  │ Mar 24, 5:45 AM             │    │
│  │ 4.7 km  •  39:55  •  18 cells│   │
│  └─────────────────────────────┘    │
│                                     │
├─────────────────────────────────────┤
│  Total: 42 runs  •  156 km         │  ← Summary stats at bottom
│         268 cells  •  0.67 km²     │
├────────┬──────────┬─────────────────┤
│  Map   │ History  │  Leaderboard    │
└────────┴──────────┴─────────────────┘
```

**Key UI components:**
- **Run cards** — each past run as a card showing date, distance, duration, cells claimed
- **Mini route map** — small static map showing the GPS trace for that run (optional for v1, can be added in later phases)
- **Summary stats** — totals at the bottom
- **Tap to expand** — tapping a run card opens a detail view with the full route on a map

**Data this screen needs:**
- List of runs (from local Room database, synced from server)
- Stats per run (distance, duration, cells)
- Aggregate stats (totals)

---

### 5.4 Leaderboard

Shows who owns the most territory in the area.

**What the user sees:**
```
┌─────────────────────────────────────┐
│  Leaderboard                        │
├─────────────────────────────────────┤
│                                     │
│  🥇  1.  RunnerKing      89 cells   │
│          ████████████████████ 2.2km² │
│                                     │
│  🥈  2.  SpeedDemon       72 cells   │
│          ████████████████ 1.8km²     │
│                                     │
│  🥉  3.  TrailBlazer      65 cells   │
│          ██████████████ 1.6km²       │
│                                     │
│      4.  You (Thiru)      42 cells   │  ← Your row highlighted
│          ██████████ 1.1km²           │
│                                     │
│      5.  NightRunner      38 cells   │
│          █████████ 0.95km²           │
│                                     │
│      ...                             │
│                                     │
├─────────────────────────────────────┤
│  Your rank: #4 of 12 players        │
├────────┬──────────┬─────────────────┤
│  Map   │ History  │  Leaderboard    │
└────────┴──────────┴─────────────────┘
```

**Key UI components:**
- **Ranked list** — players ordered by cells owned (or area, configurable)
- **Progress bars** — visual representation of relative territory size
- **Your row highlighted** — easy to spot yourself in the list
- **Player colours** — each player's name shown in their territory colour (matches the map)

**Data this screen needs:**
- Leaderboard data (from server REST API, can be cached briefly)
- Player colours and names

---

### 5.5 Auth Screens (Login / Register)

Simple authentication screens. For the prototype, email + password is sufficient.

**Login screen:**
```
┌─────────────────────────────────────┐
│                                     │
│         Territory Run               │
│         [App logo/icon]             │
│                                     │
│  ┌─────────────────────────────┐    │
│  │ Email                       │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │ Password                    │    │
│  └─────────────────────────────┘    │
│                                     │
│         [ LOG IN ]                  │
│                                     │
│     Don't have an account?          │
│         Register here               │
│                                     │
└─────────────────────────────────────┘
```

**Register screen:**
```
┌─────────────────────────────────────┐
│                                     │
│         Create Account              │
│                                     │
│  ┌─────────────────────────────┐    │
│  │ Display Name                │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │ Email                       │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │ Password                    │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │ Confirm Password            │    │
│  └─────────────────────────────┘    │
│                                     │
│         [ REGISTER ]                │
│                                     │
│     Already have an account?        │
│         Log in here                 │
│                                     │
└─────────────────────────────────────┘
```

**Key details:**
- Auth token (JWT) stored securely in Android's EncryptedSharedPreferences
- Token sent with every REST API call in the `Authorization` header
- Token sent as a query parameter when opening the WebSocket connection
- If the token expires, the app redirects to the login screen
- For POC: no OAuth (Google/Apple sign-in). Just email + password. OAuth adds complexity that does not help validate the core concept.

---

## 6. Data Flow Diagrams

### 6.1 GPS Tracking to Territory Claim Flow

```
┌──────────────┐
│  Phone GPS   │
│  Hardware    │
└──────┬───────┘
       │ Raw location (lat, lng, accuracy, timestamp)
       v
┌──────────────────┐
│  Fused Location  │   Android's location API — combines GPS, WiFi,
│  Provider        │   cell tower data for best accuracy
└──────┬───────────┘
       │ Location object
       v
┌──────────────────┐
│  LocationService │   Foreground Service — keeps tracking alive
│  (Foreground)    │   when phone is in pocket
└──────┬───────────┘
       │ Location passed to app logic
       v
┌──────────────────┐
│  GpsFilter       │   Discard impossible points (speed > 15m/s,
│                  │   accuracy > 25m)
└──────┬───────────┘
       │ Validated GPS point
       │
       ├──────────────────────────┐
       │                          │
       v                          v
┌──────────────┐          ┌──────────────────┐
│  Room DB     │          │  GridCalculator   │
│  (save point │          │  (which cell is   │
│   locally)   │          │   this point in?) │
└──────────────┘          └──────┬────────────┘
                                 │ GridCell(row, col)
                                 │
                          ┌──────┴────────────┐
                          │                   │
                          v                   v
                   ┌─────────────┐    ┌───────────────┐
                   │  Update Map │    │  WebSocket     │
                   │  (colour    │    │  Send claim:   │
                   │   cell on   │    │  {cell, user}  │
                   │   map UI)   │    └───────┬────────┘
                   └─────────────┘            │
                                              v
                                       ┌──────────────┐
                                       │  Go Backend  │
                                       │  - Validate  │
                                       │  - Store     │
                                       │  - Broadcast │
                                       └──────────────┘
```

### 6.2 WebSocket Message Flow (Server to Client)

```
┌─────────────────┐              ┌──────────────────┐
│  Runner A       │              │    Go Backend     │
│  (Android App)  │              │                   │
└────────┬────────┘              └────────┬──────────┘
         │                                │
         │  ── connect ──────────────>    │
         │  <── connection_ack ────────   │
         │                                │
         │  ── auth {token:"xyz"} ───>    │  Server validates token,
         │  <── auth_ok {user_id} ─────   │  joins user to geo-room
         │                                │
         │                                │         ┌─────────────────┐
         │                                │         │  Runner B       │
         │                                │         │  (Android App)  │
         │                                │         └────────┬────────┘
         │                                │                  │
         │                                │  <── claim {cell:(45,120)} ──
         │                                │                  │
         │                                │  Server validates,
         │                                │  updates DB,
         │                                │  broadcasts to room:
         │                                │
         │  <── territory_update ──────   │  ── territory_update ───>
         │      {cell:(45,120),           │      {cell:(45,120),
         │       owner:"runner_b",        │       owner:"runner_b",
         │       color:"#FF3333"}         │       color:"#FF3333"}
         │                                │
         │  App updates map:              │
         │  cell (45,120) turns red       │
         │                                │

Message types (client -> server):
  claim          – "I just passed through this cell"
  ping           – keepalive heartbeat

Message types (server -> client):
  territory_update  – "cell X is now owned by player Y"
  run_validated     – "your completed run has been processed"
  error             – "claim rejected" (e.g., GPS spoofing detected)
  pong              – heartbeat response
```

### 6.3 Offline / Reconnection Flow

```
Timeline:
────────────────────────────────────────────────────────────>

  CONNECTED        DISCONNECTED              RECONNECTED
  |                |                         |
  v                v                         v
  ├────────────────┤─────────────────────────┤──────────────>

  Normal flow:     Network drops:            Network returns:
  - GPS tracking   - GPS tracking CONTINUES  - WebSocket reconnects
  - Claims sent    - Points saved locally    - Queued claims flushed
    via WebSocket  - Claims queued in memory - Server sends missed
  - Updates        - Map still works           updates
    received         (cached tiles)          - Map refreshes with
                   - UI shows "offline"        current state
                     indicator


Detail of reconnection:

  ┌──────────────────┐
  │ WebSocket drops  │
  └────────┬─────────┘
           │
           v
  ┌──────────────────┐       ┌──────────────────────┐
  │ Start claim      │       │ Show "Reconnecting"  │
  │ queue (in-memory │       │ banner in UI         │
  │ + Room DB)       │       └──────────────────────┘
  └────────┬─────────┘
           │
           v
  ┌──────────────────────────────┐
  │ Reconnection loop:           │
  │   Attempt after: 1s          │
  │   Attempt after: 2s          │──── If still failing
  │   Attempt after: 4s          │
  │   Attempt after: 8s          │
  │   ...max 30s between tries   │
  └────────┬─────────────────────┘
           │ Connected!
           v
  ┌──────────────────────────────────┐
  │ 1. Send queued claims            │
  │ 2. Request territory sync        │
  │    (server sends current state   │
  │     for your visible area)       │
  │ 3. Update local state + map      │
  │ 4. Hide "Reconnecting" banner    │
  └──────────────────────────────────┘
```

---

## 7. Local Data Storage

### What gets stored on device and why

We use **Room** as the local database. Room is a library from Google that provides a clean Kotlin API over SQLite (the tiny SQL database engine built into every Android phone). Every Android phone has SQLite — Room just makes it easier to use.

**Why store data locally at all?**

1. **Offline resilience** — if the network drops during a run, GPS points are not lost
2. **Faster loading** — territory data for the last-viewed area loads instantly from cache instead of waiting for a network call
3. **Reduced API calls** — run history, user profile, and territory cache reduce server load and mobile data usage
4. **Crash recovery** — if the app or phone crashes mid-run, the GPS points already saved to Room survive

### Database tables

```
┌─────────────────────────────────────────────────────────────┐
│  Table: runs                                                │
│  Stores completed and in-progress runs                      │
├──────────────┬──────────┬───────────────────────────────────┤
│ Column       │ Type     │ Purpose                           │
├──────────────┼──────────┼───────────────────────────────────┤
│ id           │ TEXT (PK)│ UUID, matches server run ID        │
│ status       │ TEXT     │ "recording", "completed", "synced"│
│ start_time   │ INTEGER  │ Unix timestamp (milliseconds)     │
│ end_time     │ INTEGER  │ Null if still recording           │
│ distance_m   │ REAL     │ Total distance in meters          │
│ cells_claimed│ INTEGER  │ Number of cells claimed this run  │
│ synced       │ INTEGER  │ 0 = not yet uploaded, 1 = synced  │
└──────────────┴──────────┴───────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Table: gps_points                                          │
│  Every GPS point for every run                              │
├──────────────┬──────────┬───────────────────────────────────┤
│ Column       │ Type     │ Purpose                           │
├──────────────┼──────────┼───────────────────────────────────┤
│ id           │ INTEGER  │ Auto-increment primary key        │
│ run_id       │ TEXT (FK)│ Which run this point belongs to   │
│ latitude     │ REAL     │ GPS latitude                      │
│ longitude    │ REAL     │ GPS longitude                     │
│ altitude     │ REAL     │ Meters above sea level            │
│ accuracy     │ REAL     │ GPS accuracy in meters            │
│ timestamp    │ INTEGER  │ Unix timestamp (milliseconds)     │
└──────────────┴──────────┴───────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Table: territory_cache                                     │
│  Cached grid cell ownership for the visible map area        │
├──────────────┬──────────┬───────────────────────────────────┤
│ Column       │ Type     │ Purpose                           │
├──────────────┼──────────┼───────────────────────────────────┤
│ cell_row     │ INTEGER  │ Grid cell row (composite PK)      │
│ cell_col     │ INTEGER  │ Grid cell column (composite PK)   │
│ owner_id     │ TEXT     │ User ID of the owner              │
│ owner_color  │ TEXT     │ Hex colour code (e.g., "#3366FF") │
│ updated_at   │ INTEGER  │ When this was last confirmed      │
└──────────────┴──────────┴───────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Table: pending_claims                                      │
│  Cell claims queued while offline                           │
├──────────────┬──────────┬───────────────────────────────────┤
│ Column       │ Type     │ Purpose                           │
├──────────────┼──────────┼───────────────────────────────────┤
│ id           │ INTEGER  │ Auto-increment primary key        │
│ run_id       │ TEXT     │ Which run this claim is from      │
│ cell_row     │ INTEGER  │ Grid cell row                     │
│ cell_col     │ INTEGER  │ Grid cell column                  │
│ timestamp    │ INTEGER  │ When the claim was made           │
│ sent         │ INTEGER  │ 0 = queued, 1 = sent to server   │
└──────────────┴──────────┴───────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Table: user                                                │
│  Current logged-in user (always 0 or 1 rows)                │
├──────────────┬──────────┬───────────────────────────────────┤
│ Column       │ Type     │ Purpose                           │
├──────────────┼──────────┼───────────────────────────────────┤
│ id           │ TEXT (PK)│ User ID from server               │
│ email        │ TEXT     │ User's email                      │
│ display_name │ TEXT     │ Shown on leaderboard and map      │
│ color        │ TEXT     │ Assigned territory colour         │
│ total_cells  │ INTEGER  │ Total cells owned                 │
│ total_runs   │ INTEGER  │ Total runs completed              │
└──────────────┴──────────┴───────────────────────────────────┘
```

### Offline support strategy

For the prototype, offline support is simple and focused on one critical scenario: **not losing a run if the network drops**.

| Scenario | Strategy |
|---|---|
| Network drops mid-run | GPS points saved locally, claims queued, uploaded when reconnected |
| App opened with no internet | Show cached territory from `territory_cache`, show cached run history, disable "Start Run" if no internet (runs require auth validation) |
| Run completed but upload fails | Run marked as `synced = 0`, retried next time the app has network |
| Territory data is stale | Cache has `updated_at` timestamps. On next network availability, re-fetch territory for visible area. Stale cache is better than blank map. |

**What we are NOT doing for the prototype:**
- Full offline-first architecture (too complex)
- Conflict resolution for offline territory claims (server is the source of truth — if two people claim the same cell offline, whoever the server processes first wins)
- Offline map tile caching (Mapbox can do this, but it adds complexity we do not need for 10-15 users who will be in areas with reasonable connectivity)

---

## 8. Key Libraries and Dependencies

Every library below is something you would add to your `build.gradle.kts` (Kotlin-flavoured Gradle build file) in the `dependencies` block.

### Core Android

| Library | What it does | Why we need it |
|---|---|---|
| `androidx.core:core-ktx` | Kotlin extensions for core Android APIs — makes common operations shorter and more readable | Standard in every Kotlin Android project |
| `androidx.lifecycle:lifecycle-runtime-ktx` | Lifecycle-aware coroutine scopes — automatically cancels async work when a screen is destroyed | Prevents memory leaks and crashes when navigating away from a screen |
| `androidx.lifecycle:lifecycle-viewmodel-compose` | ViewModel integration with Jetpack Compose — lets Compose screens access ViewModels | Every screen needs a ViewModel; this connects them |
| `androidx.activity:activity-compose` | Bridge between Android Activities and Compose — sets up Compose as the UI for the app | Required entry point for a Compose-based app |

### Jetpack Compose (UI)

| Library | What it does | Why we need it |
|---|---|---|
| `androidx.compose.ui:ui` | Core Compose UI toolkit — the foundation for building screens | Cannot build a Compose UI without this |
| `androidx.compose.material3:material3` | Material Design 3 components — buttons, cards, text fields, bottom navigation | Pre-built UI components so we do not build everything from scratch |
| `androidx.compose.ui:ui-tooling-preview` | Live preview of Compose screens inside Android Studio | See UI changes without running the app on a device every time |
| `androidx.navigation:navigation-compose` | Screen navigation for Compose — handles moving between screens, back button, deep links | Manages the map/history/leaderboard/auth screen flow |

### Maps

| Library | What it does | Why we need it |
|---|---|---|
| `com.mapbox.maps:android` | Mapbox Maps SDK for Android — renders the interactive map, handles gestures, tile loading | The entire game board — cannot display territory without it |
| (Mapbox GeoJSON bundled) | GeoJSON parsing and creation | Building the territory overlay data that Mapbox renders |

### Location

| Library | What it does | Why we need it |
|---|---|---|
| `com.google.android.gms:play-services-location` | Google's Fused Location Provider — combines GPS, WiFi, and cell tower signals for best possible location accuracy with minimum battery drain | More accurate and battery-efficient than using the raw Android GPS API directly |

### Networking

| Library | What it does | Why we need it |
|---|---|---|
| `com.squareup.retrofit2:retrofit` | Type-safe HTTP client — you define API calls as Kotlin interfaces and Retrofit generates the implementation | Clean, readable REST API code instead of manual HTTP connection management |
| `com.squareup.retrofit2:converter-gson` | Converts JSON responses to Kotlin objects automatically | The server sends JSON; we need Kotlin objects |
| `com.squareup.okhttp3:okhttp` | HTTP and WebSocket client — the engine under Retrofit, also handles WebSocket connections directly | Retrofit uses it for REST; we also use it directly for WebSocket |
| `com.squareup.okhttp3:logging-interceptor` | Logs HTTP requests and responses in development — see exactly what data is being sent and received | Essential for debugging "why is the server returning an error?" |
| `com.google.code.gson:gson` | JSON serialization/deserialization library | Parsing WebSocket messages (which are JSON strings) into Kotlin objects |

### Local Database

| Library | What it does | Why we need it |
|---|---|---|
| `androidx.room:room-runtime` | Room database — clean Kotlin API over SQLite for storing structured data on the phone | Stores runs, GPS points, territory cache, pending claims |
| `androidx.room:room-ktx` | Kotlin coroutine support for Room — database queries return `Flow` streams and `suspend` functions | Allows non-blocking database reads that automatically update the UI when data changes |
| `androidx.room:room-compiler` (kapt) | Generates Room implementation code at compile time from your annotations | Room uses code generation — this is the generator |

### Security

| Library | What it does | Why we need it |
|---|---|---|
| `androidx.security:security-crypto` | EncryptedSharedPreferences — stores key-value data encrypted on device | Secure storage for the auth token (JWT). Plain SharedPreferences stores in readable XML — bad for tokens. |

### Coroutines

| Library | What it does | Why we need it |
|---|---|---|
| `org.jetbrains.kotlinx:kotlinx-coroutines-android` | Coroutines with Android Main dispatcher — run async work without blocking the UI thread | Every async operation (network, database, GPS) uses coroutines |

### What we are deliberately NOT using (and why)

| Library | What it does | Why we are skipping it |
|---|---|---|
| **Hilt / Dagger** | Dependency injection framework — automatically creates and provides class instances | Adds significant boilerplate and a steep learning curve. For a prototype with 10-15 users, manually creating objects is fine. Add Hilt when the codebase grows beyond ~30 classes that need injection. |
| **Kotlin Multiplatform (KMP)** | Share Kotlin code between Android and iOS | No iOS app in the POC phase. If/when iOS is added, evaluate KMP for sharing domain logic. |
| **DataStore** | Modern replacement for SharedPreferences | We only store one or two preferences (auth token, settings). EncryptedSharedPreferences is sufficient. DataStore is better for complex preference hierarchies. |
| **WorkManager** | Reliable background task scheduling | Used for deferred, guaranteed work (like "sync data when internet is available"). For the POC, we handle sync manually when the app opens. Add WorkManager when reliability of background sync becomes critical. |
| **Paging 3** | Library for loading large lists in pages | Run history for 10-15 users will be a small list. Paging adds complexity only justified for thousands of items. |

---

## 9. Error Handling and Edge Cases

### 9.1 GPS Permission Denied

**When it happens:** The user taps "Start Run" but has not granted location permission, or previously denied it.

**How Android permissions work (for a beginner):** Android requires the user to explicitly grant permission before an app can access sensitive hardware like GPS. There are three states:
- **Not yet asked** — first time, Android shows a dialog
- **Granted** — user said yes
- **Denied** — user said no (can ask again, but not forever)
- **Permanently denied** — user said "Don't ask again" (can only fix in phone Settings)

**How we handle it:**

```
User taps "Start Run"
  |
  v
Check: do we have location permission?
  |
  ├── YES → Start the run
  |
  └── NO → Has the user been asked before?
        |
        ├── Never asked → Show Android permission dialog
        |     |
        |     ├── User grants → Start the run
        |     └── User denies → Show explanation screen:
        |           "We need GPS to track your run and claim territory.
        |            Without it, the app cannot work."
        |           [Grant Permission] button → re-ask
        |
        └── Previously denied / permanently denied →
              Show screen with:
              "Location permission is required.
               Please enable it in Settings."
              [Open Settings] button → opens Android Settings for this app
```

**Important:** We need both `ACCESS_FINE_LOCATION` (GPS-level accuracy) and for the foreground service, `FOREGROUND_SERVICE_LOCATION` (declared in the manifest). On Android 10+, we also need the notification permission for the foreground service notification. Handle each permission gap gracefully.

### 9.2 Poor GPS Accuracy (Buildings, Urban Canyons)

**When it happens:** The user is running between tall buildings, under dense tree cover, or near reflective surfaces. GPS signals bounce off buildings ("multipath"), and reported accuracy degrades from 3-5 meters to 20-50 meters.

**How we handle it:**

```
Each GPS point from Android includes an "accuracy" value in meters.
This is the radius of the circle within which the phone believes you are.

  accuracy = 4m   → Very confident. Trust this point.
  accuracy = 15m  → Moderate. Accept but flag as uncertain.
  accuracy = 40m  → Poor. Point could be anywhere in a 40m radius.
  accuracy = 100m → Useless. Discard entirely.

Our approach:
  - Accept points with accuracy <= 25m
  - Discard points with accuracy > 25m
  - When accuracy is between 15-25m, accept but do NOT claim grid cells
    (the point might be in the wrong cell). Only use for route drawing.
  - Show a subtle UI indicator: "GPS signal weak" when multiple
    consecutive points have accuracy > 15m
  - If GPS accuracy is consistently terrible (e.g., running indoors),
    show a warning: "GPS signal too weak for territory tracking"
```

### 9.3 Network Drops Mid-Run

Covered in detail in Section 4.4 (Run Recording) and Section 6.3 (Offline/Reconnection Flow). Summary:

- GPS tracking and local storage continue uninterrupted
- Cell claims are queued locally
- WebSocket reconnects with exponential backoff
- On reconnection, queued claims are flushed and missed updates are received
- If network never returns, run is saved locally and synced later

### 9.4 App Killed by OS During a Run

**When it happens:** Even with a foreground service, some Android manufacturers (Xiaomi, Oppo, Vivo, Samsung) have aggressive battery optimization that kills foreground services. This is a known, painful problem for fitness apps on Android, especially on budget phones common in India.

**How we handle it:**

```
Prevention (reduce the chance of being killed):
  1. Foreground service with a visible, ongoing notification
     (Android is less likely to kill services with active notifications)
  2. Request the user to disable battery optimization for our app:
     - On first run, show a one-time dialog explaining why
     - Link to the "Battery optimization" settings page
     - Reference: dontkillmyapp.com for device-specific instructions
  3. Use START_STICKY for the service — tells Android to restart the
     service if it is killed (though GPS data during the gap is lost)

Recovery (when it happens anyway):
  1. All GPS points up to the moment of death are in Room DB (saved immediately)
  2. When the app restarts, check Room for a run with status = "recording"
  3. If found:
     - Show a dialog: "It looks like your last run was interrupted.
       Do you want to resume or save what we have?"
     - "Resume" → restart the foreground service, continue the same run
     - "Save" → mark the run as completed with the data we have
  4. The partial run data is preserved — the user does not lose everything
```

**The honest truth:** There is no perfect solution for OEM battery killing on Android. Even Strava struggles with this. The best we can do is minimize the chance (foreground service, battery optimization exemption) and recover gracefully when it happens (save data immediately, offer to resume). For the prototype with 10-15 testers, we can also simply tell testers "disable battery optimization for this app" in the onboarding instructions.

### 9.5 WebSocket Disconnection

Covered in Section 4.3. Key points:

- Exponential backoff reconnection (1s, 2s, 4s, 8s... max 30s)
- Clear UI indicator ("Reconnecting..." banner)
- Claims queued locally during disconnection
- On reconnection: flush queue, request state sync
- The run never stops because of a WebSocket disconnection — GPS tracking is independent

### 9.6 Battery Optimization Killing Background Tracking

This is related to 9.4 but specifically about the systematic problem of Chinese-brand Android phones (which dominate the Indian market) killing background processes.

**Device-specific problems:**

| Manufacturer | Problem | Workaround |
|---|---|---|
| Xiaomi (MIUI) | "Battery saver" kills background services aggressively, even foreground services sometimes | User must add app to "Autostart" list and lock app in recent apps |
| Oppo (ColorOS) | Background restrictions are ON by default | User must disable "Restrict background activity" for the app |
| Vivo (FunTouchOS) | Similar to Oppo — kills services, restricts background | User must whitelist the app in battery management |
| Samsung (One UI) | "Sleeping apps" feature puts unused apps to deep sleep | User must exclude app from "Sleeping apps" list |
| Stock Android (Pixel, etc.) | Generally well-behaved. Respects foreground services. | Usually no workaround needed |

**Our approach for the prototype:**

1. On first launch, detect the device manufacturer
2. Show a one-time setup guide specific to that manufacturer: "To ensure your runs are tracked reliably, please follow these steps..."
3. Include screenshots for the top 3-4 manufacturers (Xiaomi, Samsung, Oppo, Vivo)
4. Link to dontkillmyapp.com as a fallback for other manufacturers
5. Add a "Test tracking" button in settings that starts a 30-second background tracking test and reports if any points were lost

---

## 10. Build Phases

### Phase 1 — See a Map, Track Your Location (Week 1-2)

**Goal:** The absolute minimum to prove the technical foundation works. At the end of this phase, you can open the app, see a Mapbox map, and see your blue dot moving on it.

**What gets built:**
- Android project scaffolded with Compose
- Mapbox map displayed full-screen
- Location permission request flow
- Fused Location Provider showing your current position on the map
- Basic project structure (folders created, empty placeholders)

**What it looks like:**
```
┌─────────────────────────────────────┐
│                                     │
│                                     │
│      (Mapbox map fills screen)      │
│                                     │
│              * ← your blue dot      │
│                                     │
│                                     │
│                                     │
│                                     │
└─────────────────────────────────────┘
```

**Why this matters:** If you cannot get Mapbox rendering and GPS working, nothing else matters. This phase validates the two hardest integrations first.

**Key technical milestones:**
- Mapbox API key configured and map loads
- `AndroidView` wrapper for MapView works inside Compose
- Location permission granted and blue dot appears
- Location updates arrive every 2 seconds when the app is in the foreground

---

### Phase 2 — Record a Run, See Grid Cells (Week 3-5)

**Goal:** Run tracking works end-to-end. You can start a run, see your route drawn on the map, see grid cells light up as you pass through them, and stop the run. All data saved locally.

**What gets built:**
- Foreground service for background GPS tracking
- GPS noise filtering (GpsFilter.kt)
- GridCalculator — GPS point to grid cell conversion
- Room database with runs, gps_points, territory_cache tables
- Active run screen with timer, distance, pace, cells claimed
- Territory overlay on the map (grid cells coloured when claimed)
- Start/pause/stop run flow
- Run history screen (list of past runs from Room DB)

**What it looks like:**
```
┌─────────────────────────────────────┐
│  ● RECORDING              [Stop]    │
│                                     │
│         ┌───┬───┐                   │
│         │ B │ B │                   │
│     ┌───┼───┼───┘                   │
│     │ B │ B │    * ← you are here   │
│     └───┼───┘                       │
│         │                           │
│                                     │
├─────────────────────────────────────┤
│  0:08:12     1.4 km    4 cells      │
└─────────────────────────────────────┘
```

**Why this matters:** This is the core product experience in single-player mode. If this does not feel good, the multiplayer features built on top will not save it.

**Key technical milestones:**
- Foreground service survives phone lock and pocket mode
- GPS points arrive reliably in background
- Grid cells appear on the map in real time as you walk/run
- Run data persists in Room and survives app restart
- Route line drawn on the map from GPS points

---

### Phase 3 — Backend Integration, Multiplayer (Week 6-8)

**Goal:** Connect to the Go backend. Authentication, real-time territory updates, see other players' territories, leaderboard.

**What gets built:**
- Auth screens (login/register) with JWT
- REST API client (Retrofit) — login, register, submit run, fetch territory, fetch leaderboard
- WebSocket client (OkHttp) — real-time territory updates
- Reconnection logic with exponential backoff
- Offline claim queue (pending_claims table + flush on reconnect)
- Territory loading from server on map open
- Other players' territories shown on map
- Leaderboard screen
- Network status indicator during run

**What it looks like:**
```
The same app, but now:
- You log in first
- Other people's cells appear in different colours
- When someone claims a cell near you during a run, you see it happen live
- Leaderboard shows rankings
```

**Why this matters:** This is what makes it a game, not just a GPS tracker. Real-time multiplayer territory is the core differentiator.

**Key technical milestones:**
- Auth flow works (register, login, token stored, API calls authenticated)
- WebSocket connects during run, receives other players' updates
- Territory loads from server and renders correctly
- Offline queue works — run without network, data syncs later
- Leaderboard loads and displays

---

### Phase 4 — Polish and Prototype Testing (Week 9-10)

**Goal:** Make it usable by real testers. Fix edge cases, improve the UI, handle error states gracefully.

**What gets built:**
- Error handling for all edge cases in Section 9
- Device-specific battery optimization guide (for testers)
- Proper loading states (skeleton screens, spinners where appropriate)
- Empty states ("No runs yet — start your first run!")
- Pull-to-refresh on leaderboard and history
- Proper colour assignment per player
- Run summary screen (shown after completing a run)
- Minor UI polish (animations, transitions, spacing)
- Basic analytics/crash reporting (Firebase Crashlytics)

**Why this matters:** A prototype that crashes or loses data will give false negatives — testers will reject the concept because of bugs, not because the concept is bad. Enough polish to trust the results.

**Key technical milestones:**
- No data loss during normal usage
- Graceful handling of permission denial, network loss, GPS issues
- Testers can install, register, run, and see territory without developer help
- Crash reporting captures any issues during testing

---

### What comes after the prototype (not in scope, but worth knowing)

If the prototype validates the concept, the next things to add would be:

| Feature | Why it was deferred |
|---|---|
| Hilt dependency injection | Not needed at prototype scale. Add when the codebase exceeds ~30 injectable classes |
| Kotlin Multiplatform (KMP) | Only relevant when building an iOS app |
| WorkManager for sync | Manual sync is fine for 10-15 users. WorkManager guarantees delivery at scale |
| Offline map tile caching | Mapbox supports this, but adds storage and complexity |
| Territory animations (capture effects) | Important for game feel, but not for validating the core concept |
| Push notifications | "Someone is stealing your territory!" — requires Firebase Cloud Messaging setup |
| Snap-to-road GPS correction | Uses OpenStreetMap road data to align GPS points to actual roads. Improves accuracy significantly but requires road data infrastructure |
| Anti-cheat (Play Integrity) | Validates the app is running on a real device. Essential for public launch, overkill for 10 trusted testers |
| Kalman filter for GPS smoothing | More sophisticated than our simple speed/accuracy filter. Better accuracy but more complex math |
| H3 hex grid | Uber's hexagonal grid system handles latitude distortion correctly. Better than our simple square grid for global coverage, but adds a dependency and conceptual complexity |

---

## Summary

This plan gives you a working territory-mapping running app in 10 weeks, built in four phases that each produce something runnable:

1. **Map + GPS** — prove the hardest integrations work
2. **Run tracking + grid** — the core single-player experience
3. **Backend + multiplayer** — the game comes alive
4. **Polish + testing** — make it trustworthy for real testers

The architecture is intentionally simple. No dependency injection framework, no complex state management, no Kotlin Multiplatform. Just clean folder structure, ViewModels, Room for local storage, Retrofit for REST, OkHttp for WebSocket, and Mapbox for the map. Every decision prioritizes clarity and debuggability over sophistication — because for a prototype, understanding what went wrong matters more than having elegant abstractions.
