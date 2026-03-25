# Technology Stack

## Overview

| Layer | Choice | Notes |
|---|---|---|
| Android | Kotlin (native) | Jetpack Compose for UI |
| iOS | Swift (native) | SwiftUI for UI |
| Backend | Node.js + TypeScript | REST + WebSocket |
| Database | PostgreSQL + PostGIS | Spatial queries for territory |
| Cache | Redis | Leaderboards, sessions, real-time state |
| Map Rendering | Mapbox GL | Both platforms + web |
| Real-time | Socket.IO | Territory updates, live competition |
| Cloud | AWS or GCP | Containerized (ECS/Cloud Run) |

---

## Mobile — Why Native Over Cross-Platform

This is the single most important stack decision in this project. Here's the honest breakdown.

### Why native is right for this app

1. **GPS tracking is the core product.** Background location tracking, battery optimization, and motion sensors behave very differently on Android vs iOS. Cross-platform abstractions (React Native, Flutter) leak badly here. You'll end up writing platform-specific code anyway for the most critical feature — at which point the "write once" benefit evaporates.

2. **Map rendering performance.** Territory overlay rendering with hundreds/thousands of polygons on a live map needs to be buttery smooth. Native gives you direct access to Metal (iOS) and Vulkan/OpenGL (Android) through Mapbox's native SDKs, which are significantly more performant than their React Native or Flutter wrappers.

3. **Background activity tracking.** Android and iOS have fundamentally different models for background execution. Android uses foreground services with persistent notifications. iOS uses background location modes with strict limits. Abstracting over these is a constant source of bugs in cross-platform apps — Strava, Nike Run Club, and every serious fitness app is native for this exact reason.

4. **Sensor fusion.** Combining GPS + accelerometer + gyroscope for accurate pace/distance (especially in urban canyons or tunnels) requires platform-specific APIs. Android's Fused Location Provider and iOS's Core Motion have no good cross-platform equivalent.

### What this costs you

- **Two codebases to maintain.** Every feature ships twice. Every bug potentially exists twice. For a small team / POC this is painful.
- **Slower iteration speed.** No hot reload across both platforms. Changes take longer to validate.
- **Need expertise in both ecosystems.** Kotlin + Jetpack Compose is a different mental model from Swift + SwiftUI.

### Mitigation strategies

- **Shared business logic via KMP (Kotlin Multiplatform).** You can write territory calculation logic, API clients, data models, and caching in Kotlin and compile to both Android (native) and iOS (via Kotlin/Native). This lets you keep a single source of truth for non-UI logic while keeping UI fully native. Worth evaluating seriously — companies like Netflix, Cash App, and Philips use this in production.
- **API-first design.** Push as much logic as possible to the backend. Territory ownership calculation, leaderboard computation, challenge validation — none of this should live on the client. The thinner the client, the less duplication hurts.
- **Feature parity tracking.** Maintain a shared feature matrix. Ship Android-first or iOS-first per feature (not both simultaneously) to reduce context switching.

### Verdict

Native is the correct call for a GPS-intensive, map-heavy, real-time fitness app. The tradeoff is development speed, which you offset with KMP for shared logic and a thick backend.

---

## Android — Kotlin

### UI: Jetpack Compose

- Declarative UI toolkit, modern, Google's recommended approach
- Excellent for reactive state management (map overlays updating in real-time as territory changes)
- Strong interop with existing Android Views (important because Mapbox SDK still uses Views)

### Key Libraries

| Purpose | Library | Why |
|---|---|---|
| Maps | Mapbox SDK for Android | Best customization for territory overlays. Google Maps works but Mapbox gives you full style control and vector tile customization |
| Location | Google Fused Location Provider | Best accuracy, battery-aware, handles GPS + WiFi + cell triangulation |
| Background tracking | Foreground Service + WorkManager | Android requires a persistent notification for background GPS. WorkManager for periodic syncs |
| Networking | Ktor Client or Retrofit | Ktor if using KMP (shared with iOS), Retrofit if Android-only |
| Local DB | Room | Offline run storage, territory cache |
| DI | Hilt | Standard for Android |
| State | Kotlin Coroutines + Flow | Reactive streams for real-time updates |

### What works well

- Kotlin is a mature, expressive language with excellent tooling
- Coroutines make async GPS tracking and network calls clean
- Foreground services give you reliable background tracking (unlike iOS which throttles aggressively)
- Android is dominant in India (your target emerging market) — ~95% market share

### What doesn't work well / watch out for

- **Battery drain.** Continuous GPS on Android will kill battery. You need adaptive polling — high frequency during active runs, low frequency otherwise. Users will uninstall if you drain their battery.
- **GPS accuracy on cheap Android devices.** India-first means you'll encounter devices with poor GPS chipsets. Need to handle GPS drift aggressively (Kalman filtering, snap-to-road, minimum speed thresholds to discard noise).
- **Mapbox View in Compose.** Mapbox doesn't have a first-class Compose component yet. You'll wrap it in an `AndroidView` composable — works fine but feels janky to develop.
- **Background kill behavior varies wildly by OEM.** Xiaomi, Oppo, Vivo, Samsung all have aggressive battery optimization that kills background services. You'll need device-specific workarounds (dontkillmyapp.com is your reference). This is a real problem for an India-first app.

---

## iOS — Swift

### UI: SwiftUI

- Apple's declarative UI framework
- Excellent for animations and transitions (territory capture celebrations, map interactions)
- Lifecycle-aware, works well with background state changes

### Key Libraries

| Purpose | Library | Why |
|---|---|---|
| Maps | Mapbox Maps SDK for iOS | Same reason as Android — full style control for territory rendering |
| Location | Core Location | Apple's location framework, supports background modes |
| Motion | Core Motion | Accelerometer + gyroscope for pace accuracy |
| Background tracking | Background Location Updates mode | Must declare in entitlements, Apple reviews this strictly |
| Networking | URLSession or Ktor (if KMP) | Native URLSession is solid, Ktor if sharing with Android |
| Local DB | SwiftData or Core Data | Run caching, offline territory state |
| State | Combine + async/await | Reactive streams for real-time updates |

### What works well

- Core Location is excellent — accurate, battery-efficient, well-documented
- Apple's background location mode is strict but reliable once approved
- SwiftUI animations will make territory capture feel satisfying
- iOS users tend to spend more on in-app purchases (important for monetization)

### What doesn't work well / watch out for

- **App Store review for background location.** Apple will reject your app if you don't clearly justify why you need "always on" location. You need a compelling description and must show the blue status bar indicator. Plan for review rejections and resubmissions.
- **iOS throttles background GPS.** Unlike Android's foreground service model, iOS will reduce location update frequency when backgrounded. You get significant location updates (SLC) but they're less frequent. Active run tracking needs the `allowsBackgroundLocationUpdates` flag on your location manager — and the user must keep the app in the "active tracking" state.
- **MapKit vs Mapbox.** MapKit is free and deeply integrated, but lacks the custom styling needed for territory overlays. Mapbox costs money at scale but gives you full control. For this app, Mapbox is worth the cost.
- **Smaller market share in India.** iOS is ~3-5% in India. If you're going India-first, iOS can be a fast-follow, not the lead platform.

---

## Backend — Node.js + TypeScript

### Framework: Fastify or Express

| Option | Recommendation |
|---|---|
| **Fastify** | Preferred. Faster than Express, built-in schema validation, better TypeScript support, plugin architecture |
| **Express** | Fine if you're more comfortable with it. Massive ecosystem. Slightly more boilerplate |

### What the backend is responsible for

1. **User auth and profiles** — JWT-based auth, OAuth (Google/Apple sign-in)
2. **Run ingestion** — receive GPS trace data from clients, process into territory claims
3. **Territory engine** — the core: calculate who owns what based on run data. This is the hardest part.
4. **Leaderboards** — real-time ranking using Redis sorted sets
5. **Real-time updates** — WebSocket (Socket.IO) for live territory changes during runs
6. **Challenge/game mode logic** — 1v1 duels, weekly challenges, crew battles
7. **Notifications** — push notifications via FCM (Android) and APNs (iOS)
8. **Admin/moderation** — anti-cheat, GPS spoofing detection

### What works well

- TypeScript on the backend gives you type safety and shared type definitions with the API contract
- Node's event loop is excellent for handling many concurrent WebSocket connections (real-time territory updates)
- Massive npm ecosystem for everything you need
- Easy to deploy, easy to scale horizontally
- Fast iteration speed — important for a POC

### What doesn't work well / watch out for

- **CPU-intensive spatial calculations.** Node is single-threaded. Territory calculation (point-in-polygon, polygon union/intersection, area computation) is CPU-heavy. Options:
  - Use worker threads for heavy spatial computation
  - Offload spatial queries to PostGIS (preferred — let the database do what it's good at)
  - For really heavy stuff, consider a spatial computation microservice in Rust or Go
- **Memory usage at scale.** Node can be memory-hungry with many concurrent WebSocket connections. Monitor and plan for horizontal scaling early.
- **Not the fastest for raw throughput.** Fine for POC and early scale. If you hit millions of users, you may want to rewrite hot paths (territory engine) in a more performant language. But that's a good problem to have.

---

## Database — PostgreSQL + PostGIS

This is arguably the most important backend choice. PostGIS turns PostgreSQL into a spatial database.

### Why PostGIS

- **Territory is fundamentally a geospatial problem.** "Does this GPS trace overlap with this polygon?" "What's the area of the union of these two runs?" "Find all territories within 5km of this point." PostGIS handles all of this natively with spatial indexes.
- Spatial index (GiST) makes geographic queries fast — O(log n) instead of scanning every polygon
- Functions like `ST_Contains`, `ST_Intersection`, `ST_Union`, `ST_Area` are exactly what the territory engine needs
- Well-proven — used by OpenStreetMap, Strava, Uber, and every serious geo app

### Schema concepts

```
users           — profiles, auth
runs            — GPS traces stored as LineString geometries
territories     — polygons with owner references
territory_grid  — pre-divided hex/square cells for efficient claiming
leaderboards    — materialized views or Redis-backed
```

### What works well

- PostGIS spatial queries are fast and correct — no need to implement your own geometry library
- PostgreSQL is rock-solid for transactional data
- JSONB columns for flexible metadata (run stats, device info)
- Excellent tooling (pgAdmin, PostGIS viewers)

### What doesn't work well / watch out for

- **Complex polygon operations at write-heavy scale.** Computing the union of thousands of overlapping run polygons in real-time can be expensive. Solution: use a grid-based system (hex tiles like Stride, or square cells) instead of arbitrary polygons. Grid cells are O(1) lookups, arbitrary polygons are O(n) intersections.
- **Schema migrations with spatial data are tricky.** Use a migration tool (Prisma, Knex, or node-pg-migrate) from day one.
- **PostGIS adds operational complexity.** You need the extension installed, spatial indexes configured correctly, and SRID consistency (use EPSG:4326 everywhere — it's the GPS standard).

---

## Cache & Real-time State — Redis

### Use cases

| Use case | Redis feature |
|---|---|
| Leaderboards | Sorted sets (`ZADD`, `ZRANK`, `ZRANGE`) |
| Active run sessions | Hashes with TTL |
| Territory ownership cache | Geo sets + key-value |
| Rate limiting | Sliding window counters |
| Real-time pub/sub | Pub/Sub for cross-server WebSocket broadcasting |
| Session management | Key-value with TTL |

### What works well

- Sorted sets are purpose-built for leaderboards — O(log n) insert and rank lookup
- Pub/Sub enables scaling WebSocket servers horizontally (multiple Node instances)
- Sub-millisecond reads for territory state during active runs

### What doesn't work well

- **Not a database.** Don't store anything in Redis that you can't afford to lose. It's a cache and real-time state layer. Source of truth stays in PostgreSQL.
- **Memory-bound.** If you cache every territory cell for every city, memory usage grows fast. Be selective about what you cache.

---

## Map Rendering — Mapbox GL

### Why Mapbox over Google Maps

| Factor | Mapbox | Google Maps |
|---|---|---|
| Custom styling | Full control (Mapbox Studio) | Limited |
| Territory overlays | Custom layers, fill-extrusion, animations | Basic polygon overlay |
| Offline maps | Built-in | Requires workarounds |
| Pricing | Free tier then per-load | Per-load, gets expensive |
| Open source | SDK is open source | Proprietary |

For a territory game, **visual identity of the map is everything.** Players need to see their territory as distinctly theirs — custom colors, glow effects, animations on capture. Mapbox lets you build this. Google Maps doesn't.

### What works well

- Vector tiles render fast and look sharp at any zoom
- Custom map styles — you can make the map feel like your game, not like Google Maps
- GeoJSON source layers update in real-time — perfect for territory that changes during a run
- 3D extrusion — you can make owned territory "rise up" from the map for visual impact
- Offline map packs for areas with poor connectivity (important for India)

### What doesn't work well

- **Cost at scale.** Free tier is 25,000 map loads/month. After that it's $5 per 1,000 loads. Budget for this.
- **SDK complexity.** More complex to set up than Google Maps. Steeper learning curve.
- **Mapbox GL Native maintenance.** The native SDKs have had periods of slow maintenance. MapLibre (open-source fork) is a fallback option if Mapbox becomes problematic.

---

## Real-time — Socket.IO

### Why Socket.IO

- Handles WebSocket with automatic fallback to long-polling
- Room-based broadcasting — put all runners in a city into a "room" and broadcast territory changes only to them
- Built-in reconnection handling — critical for runners on mobile networks
- Works with Redis adapter for horizontal scaling

### Use cases

- Live territory updates during a run (you see someone stealing your zone in real-time)
- Active runner count in your area
- 1v1 duel state synchronization
- Push-based leaderboard updates

### What doesn't work well

- **Battery and data usage on mobile.** Persistent WebSocket connections drain battery and use mobile data. Only connect during active runs or when the app is foregrounded. Don't keep a socket open 24/7.
- **Scaling complexity.** With Redis adapter it scales, but it's another thing to operate. For POC, single server is fine.

---

## Infrastructure (POC Phase)

Keep it simple for now. Don't over-engineer infra for a POC.

```
POC Setup:
- 1x VPS or small cloud instance (e.g., AWS t3.medium, DigitalOcean droplet)
- PostgreSQL + PostGIS (managed: AWS RDS or Supabase)
- Redis (managed: AWS ElastiCache or Upstash)
- Node.js backend (PM2 or Docker)
- Mapbox account (free tier)
- Firebase for push notifications (free tier)
```

### Scale-ready path (when needed)

```
Production Setup:
- Container orchestration (ECS, Cloud Run, or k8s)
- Load balancer for WebSocket sticky sessions
- Read replicas for PostgreSQL
- CDN for map tile caching
- Separate spatial computation workers
- CI/CD pipeline
```

---

## Anti-Cheat Considerations (Stack Impact)

GPS spoofing is the #1 threat to this app. Stack choices that help:

- **Server-side validation.** Never trust the client. Validate GPS traces on the backend — check speed limits (humans can't run 100km/h), check continuity (no teleporting), cross-reference with known road/path networks.
- **Device attestation.** Android SafetyNet/Play Integrity API, iOS DeviceCheck — verify the app is running on a real device, not an emulator.
- **Sensor data cross-reference.** If the accelerometer shows no movement but GPS shows running, flag it. This requires collecting motion data alongside GPS (already planned via Core Motion / Fused Location Provider).
- **Statistical anomaly detection.** Backend service that flags accounts with inhuman running patterns. Can start simple (rule-based) and evolve to ML later.

---

## Summary — What I'd Prioritize for POC

| Priority | Component | Why |
|---|---|---|
| 1 | Backend + PostGIS + territory grid engine | Core gameplay logic, everything depends on this |
| 2 | Android app (Kotlin) with Mapbox | India-first = Android-first. Map + GPS tracking |
| 3 | Real-time WebSocket layer | Territory feels alive only if updates are instant |
| 4 | iOS app (Swift) with Mapbox | Fast-follow once Android is stable |
| 5 | Redis leaderboards + caching | Makes the competitive layer snappy |
| 6 | Anti-cheat baseline | Don't wait until cheaters ruin the game |

---

## Open Questions for Discussion

1. **Grid system vs arbitrary polygons?** Hex grid (like Stride) is simpler and more performant. Arbitrary polygons from run traces are more visually interesting but harder to compute. Could do hybrid — snap runs to a fine grid.

2. **KMP for shared logic — yes or no?** Adds complexity upfront but saves duplication long-term. Worth it if the team has Kotlin expertise.

3. **Mapbox vs MapLibre?** MapLibre is free and open source (forked from Mapbox GL JS v1). Lacks some features but eliminates per-load costs. Worth considering for cost-sensitive scaling.

4. **Monolith vs microservices for backend?** For POC: monolith. Split out the territory engine and real-time services only when you hit scaling pain.

5. **Where to host?** AWS gives the most flexibility. Supabase (PostgreSQL + PostGIS + auth) could accelerate the POC significantly. DigitalOcean is cheapest for a simple setup.
