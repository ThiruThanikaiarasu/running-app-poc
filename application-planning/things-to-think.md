# Things to Think About

Decisions that need to be made before moving to architecture.

---

- [ ] **Grid-based territory vs arbitrary polygons from run traces?**
  - Hex grid (like Stride) is simpler and more performant — each cell is an O(1) lookup
  - Arbitrary polygons from run traces are more visually interesting but expensive to compute (polygon union/intersection)
  - Hybrid option: snap runs to a fine-grained grid for ownership, but render the visual territory as smoothed polygons for a better look
  - This decision impacts database schema, PostGIS query complexity, real-time update performance, and map rendering approach

- [ ] **Go vs Node.js + TypeScript for backend?**
  - **Node + TS:** Faster iteration, huge npm ecosystem, easy WebSocket handling (Socket.IO), shared TypeScript types with API contracts, great for POC speed
  - **Go:** Better raw performance, built-in concurrency (goroutines), lower memory footprint, single binary deployment, stronger for CPU-heavy spatial computation
  - Go handles concurrent WebSocket connections more efficiently and won't choke on CPU-intensive territory calculations the way Node's single thread can
  - Node is faster to prototype with but may need hot paths rewritten later if the app scales
  - Consider: team familiarity, hiring pool, and how much spatial computation lives in PostGIS (if most heavy lifting is in the DB, Node's CPU weakness matters less)

- [ ] **PostgreSQL + PostGIS vs a specialized database?**
  - **PostgreSQL + PostGIS:** Battle-tested for geospatial (used by Strava, Uber, OpenStreetMap), rich spatial functions (ST_Contains, ST_Union, ST_Area), GiST spatial indexes, handles both relational data and spatial queries in one DB
  - **MongoDB + GeoJSON:** Native geospatial indexes, flexible schema, but limited spatial operations compared to PostGIS (no polygon union/intersection, no area calculation) — fine for "find near me" but weak for territory computation
  - **TileDB / H3 (Uber's hex grid system):** Purpose-built for spatial tiling, could pair with any DB as a computation layer rather than replacing the DB entirely
  - **CockroachDB / Yugabyte (distributed SQL with spatial):** PostGIS-compatible but distributed — overkill for POC, relevant at serious scale
  - Key question: how much spatial computation happens in the DB vs in application code? If territory logic is DB-heavy, PostGIS is hard to beat. If you go grid-based (H3 hex cells), spatial queries simplify to cell ID lookups and almost any DB works
  - PostgreSQL is the safe, proven choice — only consider alternatives if a specific scaling or data model need demands it
