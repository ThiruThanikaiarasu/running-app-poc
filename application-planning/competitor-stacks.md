# Competitor Tech Stacks & Architecture

> Most territory running apps don't publicly disclose their tech stacks. What's below is a mix of **confirmed info** (from job postings, GitHub repos, wikis, app analysis) and **educated inference** (from app behavior, platform patterns, and industry norms). Strava is included as a reference — it's not a direct competitor but it's the gold standard for GPS fitness app architecture.

---

## 1. INTVL (Australia)

### What we know

- **Platforms:** Native iOS + Android (separate apps on App Store and Play Store)
- **App size:** ~182 MB (suggests native, not a thin cross-platform wrapper)
- **Map rendering:** Custom map with territory overlays — likely Mapbox (the level of map customization suggests it's not vanilla Google Maps)
- **Strava integration:** OAuth-based sync for importing runs
- **Features:** Real-time territory capture ("Terra mode"), personalized training plans, smartwatch support (Apple Watch)
- **Recent updates:** "New map rendering system with instant map loading" (v3.3.7, March 2026)

### Inferred stack

| Layer | Likely choice | Reasoning |
|---|---|---|
| Mobile | Native (Swift / Kotlin) or React Native | App size and smartwatch integration suggest native. Could be React Native with native modules |
| Maps | Mapbox GL | Custom territory styling, color overlays, the "new map rendering system" language aligns with Mapbox upgrades |
| Backend | Node.js or Python | Small Australian startup, fast iteration. Likely a modern backend framework |
| Database | PostgreSQL + PostGIS | Territory capture requires spatial queries. No real alternative at this scale |
| Real-time | WebSocket (Socket.IO or similar) | "Real-time takeovers" require push-based updates |
| Cloud | AWS or GCP | Standard for Australian startups |
| Territory model | Grid-based (1km² zones) | Confirmed — they use fixed 1km² zones, not arbitrary polygons |

### Architecture takeaway

INTVL uses a **grid-based territory system with 1km² fixed zones**. This is a deliberate choice — large zones mean fewer cells to manage, simpler computation, and territory feels meaningful (you own a whole square kilometer). The tradeoff is granularity — two runners on the same street both claim the same huge zone.

---

## 2. Stride / Run an Empire (UK)

### What we know

- **Platforms:** iOS + Android
- **Origin:** Started as "Run an Empire" (2018), rebranded to "Stride"
- **Territory model:** Hexagonal tile grid — confirmed
- **Integrations:** Strava, Apple Health, Garmin
- **Gameplay:** Cumulative ownership — the person who has passed through a hex the most times owns it

### Inferred stack

| Layer | Likely choice | Reasoning |
|---|---|---|
| Mobile | Native or Unity | Early versions had a game-like UI that could suggest Unity. Later versions look more native |
| Maps | Mapbox or custom tile renderer | Hex grid overlay requires custom map layer rendering |
| Backend | Ruby on Rails or Node.js | UK startup, 2018 era — Rails was common for this type of app |
| Database | PostgreSQL + PostGIS | Hex grid still needs spatial indexing for "which hex does this GPS point fall in?" |
| Territory model | Hex grid | Confirmed — hexagonal tiles, cumulative ownership model |

### Architecture takeaway

Stride's **hex grid with cumulative ownership** is the most interesting model among competitors. Ownership isn't binary (you ran here once, it's yours) — it's based on who has run through a hex the most times. This means the database needs to track **run counts per hex per user**, not just current ownership. Schema likely looks like:

```
hex_visits: (user_id, hex_id, visit_count, last_visit)
hex_ownership: (hex_id, owner_id)  -- derived from max(visit_count)
```

This model rewards consistency over single big runs — architecturally simpler but requires frequent aggregation queries.

---

## 3. Turf (Sweden — Andrimon)

### What we know (most documented of all competitors)

- **Platforms:** Android (launched 2010), iOS (added later)
- **Developer:** Andrimon AB, Stockholm
- **Server:** MongoDB for persistent data storage — **confirmed** via Turf wiki and community docs
- **API:** REST-based services — **confirmed** (public Turf API documented at wiki.turfgame.com)
- **Architecture:** Server maintains in-memory state + MongoDB for persistence — **confirmed**
- **GitHub:** github.com/turf (Andrimon's org) — limited public repos
- **Community tools:** Third-party Java tools built against Turf API (github.com/hoddmimes/turf-game)
- **Players:** ~315,000+ registered

### Confirmed stack

| Layer | Choice | Source |
|---|---|---|
| Android | Java (originally), likely Kotlin now | Launched 2010 — pre-Kotlin era. Play Store listing |
| iOS | Objective-C originally, likely Swift now | Added later, wiki mentions "restarted iPhone development" |
| Backend | Custom server (likely Java/JVM) | Swedish dev culture + 2010 era + REST API patterns suggest Java |
| Database | **MongoDB** | Confirmed via Turf wiki — "MongoDB database for persisting long-living data and states" |
| API | REST + HTML endpoints | Confirmed — public API documentation exists |
| State management | In-memory server state | Confirmed — server holds live game state in RAM, persists to MongoDB |
| Territory model | Pre-placed virtual zones | Not a grid — zones are manually/community-placed at specific GPS coordinates |

### Architecture takeaway

Turf is the **oldest app in this space** (2010) and its architecture reflects that era. Key insights:

- **MongoDB, not PostGIS.** Their zones are pre-placed points with a capture radius, not complex polygons or grid cells. MongoDB's basic geospatial queries (`$near`, `$geoWithin`) are sufficient for this simpler model. They don't need polygon operations.
- **In-memory state.** Zone ownership is held in RAM for fast lookups, with MongoDB as the persistence layer. This works because their zone count is finite and manageable (~tens of thousands worldwide, not millions of grid cells).
- **Zone placement is community-driven.** Players suggest new zones, moderators approve them. This avoids the "pre-divide the entire world into cells" problem but limits coverage to where players actively request zones.
- **Monthly resets.** All zone ownership resets each month. This is an architectural simplification — no need for complex historical state, conflict resolution, or territory decay algorithms.

### What they got right

- Simple zone model (point + radius) keeps computation trivial
- Monthly resets keep the database clean and gameplay fresh
- Public API enabled a community ecosystem of third-party tools

### What they got wrong

- MongoDB limits their spatial query capabilities — can't do polygon union/intersection if they wanted to
- In-memory state doesn't scale horizontally without careful coordination
- Zone placement depends on community — cold-start problem in new regions

---

## 4. Motera

### What we know

- **Platforms:** iOS only
- **Territory model:** Loop-based — run a closed loop to claim the area inside it
- **Features:** Route generator, points redeemable for gear/perks
- **Status:** Newer, less established

### Inferred stack

| Layer | Likely choice | Reasoning |
|---|---|---|
| Mobile | Swift (native iOS) | iOS-only app, likely a small team going native |
| Maps | Mapbox or MapKit | Loop visualization needs custom polygon rendering |
| Backend | Node.js or Python (Django/FastAPI) | Small startup, likely chose for speed |
| Database | PostgreSQL + PostGIS | Loop-based claiming = arbitrary polygon territory. PostGIS is almost mandatory here |
| Territory model | Arbitrary polygons from run loops | Confirmed — "run in loops to claim zones, bigger loop = more territory" |

### Architecture takeaway

Motera is the **only competitor using true arbitrary polygon territory** (not a grid). This is the most visually appealing approach but the hardest to compute:

- Every run loop creates a polygon
- Overlapping polygons between users need intersection computation
- "Bigger loop = more territory" means area calculation on every run
- Scaling this to thousands of users in a city would require heavy PostGIS optimization or a computation queue

Their iOS-only approach may be partly driven by this complexity — managing one platform while solving the polygon computation problem is hard enough.

---

## 5. Squadrats (Strava Extension)

### What we know

- **Type:** Web app + browser extension (not a mobile app)
- **Integration:** Strava and Garmin Connect (OAuth)
- **Territory model:** Square grid — world divided into 1-mile squares ("Squadrats"), each subdivided into 64 smaller squares ("Squadratinhos")
- **Browser extensions:** Chrome, Firefox
- **Route planning:** Plugin overlays on Strava, Komoot, RideWithGPS, Garmin Connect

### Inferred stack

| Layer | Likely choice | Reasoning |
|---|---|---|
| Frontend | JavaScript/TypeScript web app | Browser-based, Chrome extension |
| Maps | Leaflet or Mapbox GL JS | Web-based map rendering with custom grid overlay |
| Backend | Likely a lightweight server (Node.js or Python) | Pulls data from Strava API, processes grid membership |
| Database | PostgreSQL or even SQLite | Grid cell ownership is simple key-value (cell_id -> user_id) |
| Territory model | Square grid (1-mile cells, subdivided into 64 sub-cells) | Confirmed |

### Architecture takeaway

Squadrats is the **simplest architecture** of all competitors:

- No real-time tracking (reads from Strava after the fact)
- No mobile app to build
- Grid is trivially simple — divide lat/long by cell size, get cell ID
- Two-tier grid (Squadrats + Squadratinhos) gives both macro and micro views of coverage
- No competition/ownership — purely personal exploration tracking

This validates that a **grid-based system can be built with minimal infrastructure**. The two-tier grid concept (large cells + subdivisions) is worth borrowing.

---

## Reference: Strava's Architecture

Strava isn't a competitor but is the benchmark for GPS fitness app architecture. Their stack is well-documented.

### Confirmed stack

| Layer | Choice | Source |
|---|---|---|
| Backend (original) | Ruby on Rails (monolith) | StackShare, engineering blog |
| Backend (current) | Microservices in Scala, Ruby, and Go | Strava engineering blog |
| Database | PostgreSQL | StackShare — "solid ACID-compliant database" |
| Cache | Redis (AWS ElastiCache) | StackShare — used for background workers, caching |
| Deployment | Mesos, Marathon, Docker (~100 services) | Strava engineering blog |
| Cloud | AWS | Confirmed across multiple sources |
| API | REST (public API at developers.strava.com) | Public documentation |

### Architecture evolution

```
2009-2014: Monolithic Ruby on Rails app
    |
    v
2014-2018: Gradually extracted services (Scala for performance-critical paths)
    |
    v
2018-present: ~100 microservices (Scala, Ruby, Go)
              Mesos/Marathon/Docker orchestration
              Event-driven architecture
              PostgreSQL + Redis + likely time-series DB for activity data
```

### Key lessons from Strava

1. **Started monolith, went microservices only when needed.** They didn't start with 100 services — they extracted them over years as pain points emerged. This validates starting with a monolith for POC.

2. **Scala for hot paths.** CPU-intensive work (route matching, segment detection, leaderboard computation) moved to Scala for performance. Equivalent for us: if Node.js chokes on spatial computation, extract those paths to Go.

3. **PostgreSQL, not a fancy spatial DB.** Strava handles billions of GPS points with PostgreSQL. PostGIS extension likely used but not confirmed — they may do spatial computation in application code.

4. **Redis is critical infrastructure.** Not just a cache — used for background job queues, real-time leaderboards, and session management.

---

## Summary — What We Can Learn

| Decision | Competitors' choice | Lesson for us |
|---|---|---|
| Territory model | Grid (INTVL, Stride, Squadrats, Turf zones) vs Polygons (Motera only) | Grid is dominant for a reason — it scales. Only Motera attempted polygons, and they're iOS-only and small |
| Database | MongoDB (Turf) vs PostgreSQL (everyone else, implied) | PostgreSQL + PostGIS is the industry standard. Turf's MongoDB works only because their zone model is simple point-and-radius |
| Mobile approach | Native across the board | No major competitor uses Flutter or React Native. GPS + maps + background tracking pushes everyone native |
| Grid size | 1km² (INTVL), hex tiles (Stride), 1-mile squares (Squadrats), point-radius (Turf) | Smaller cells = more granularity but more data. H3 hex system gives you adjustable resolution |
| Real-time | Only INTVL and Turf have real-time mechanics | Most competitors are async (run, sync, see results later). Real-time is a differentiator |
| Backend | Varies, but all lightweight for their scale | No one in this niche is at Strava-scale yet. Simple backends work fine at this stage |

---

Sources:
- [Strava Tech Stack — StackShare](https://stackshare.io/strava/strava)
- [Strava Engineering Blog](https://medium.com/strava-engineering)
- [Strava Labs — Infrastructure Posts](https://labs.strava.com/blog/tags/infrastructure/)
- [Turf Wiki](https://wiki.turfgame.com/en/index.php?title=Turf)
- [Turf — Wikipedia](https://en.wikipedia.org/wiki/Turf_(video_game))
- [Turf Game GitHub (community tools)](https://github.com/hoddmimes/turf-game)
- [INTVL App Review — SportsSteps](https://sportssteps.com/intvl-app-turning-every-run-into-a-game-of-territory-and-triumph/)
- [Squadrats](https://squadrats.com/)
- [Squadrats Review — PQRS](https://pqrs.in/blog/squadrats-review)
- [Stride / Run an Empire](https://www.runanempire.com/stride/)
