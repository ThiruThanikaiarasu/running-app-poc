# Grid-Based Territory vs Arbitrary Polygons

## One-Liner

Two different ways to divide the map into "ownable" chunks — pre-cut tiles vs freeform shapes traced by your actual run.

## Prerequisites

- Basic understanding of GPS tracking (your phone records your location as a series of lat/long points)
- What a polygon is (a closed shape with straight edges)

## What Is It?

**Grid-based:** The entire map is pre-divided into fixed cells (hexagons or squares) before anyone runs. When you run through a cell, you claim it. Every cell is the same size, same shape, pre-defined.

**Arbitrary polygons:** There are no pre-defined cells. Instead, the shape of your actual run route becomes your territory. Run a loop around a park? That exact loop shape is now your territory. The shapes are unique to each run.

## Real-World Analogy

**Grid-based** is like a chessboard. The board is already divided into 64 squares. You place a piece on a square, you control that square. Simple. Fast. Everyone understands the boundaries.

**Arbitrary polygons** is like drawing with a marker on a blank map. You draw any shape you want. Your territory is exactly what you drew — unique, personal, but now figuring out "does your drawing overlap mine?" gets complicated. Imagine two people drawing freeform shapes on the same paper and trying to calculate who owns more area — that's the computational problem.

## Example — How Territory Claiming Works

### Grid-based (hex cells)

```
You run from point A to point B.
Your GPS trace passes through cells: H1, H2, H3, H5, H8, H9

Backend logic:
  for each cell your GPS trace touches:
    mark cell as owned by you        // Simple lookup: cell_id -> owner

Total cells owned: 6
Territory area: 6 x cell_size        // Exact, instant calculation
```

### Arbitrary polygons

```
You run a loop: A -> B -> C -> D -> A (back to start)
Your GPS trace forms a closed polygon with vertices at A, B, C, D

Backend logic:
  convert GPS points to a polygon geometry
  compute polygon area using ST_Area()                    // Moderate cost
  check overlap with EVERY existing polygon: ST_Intersection()  // Expensive
  compute union of overlapping territories: ST_Union()          // Very expensive
  store the resulting merged polygon

Territory area: computed from the actual polygon geometry  // Exact but costly
```

## Diagram

```
=== GRID-BASED (Hex) ===

    The map is pre-divided. Your run claims cells it passes through.

      ___     ___     ___
     / 1 \___/ 2 \___/ 3 \
     \___/ 4 \___/ 5 \___/
     / 6 \___/ 7 \___/ 8 \
     \___/ 9 \___/10 \___/

    Your run path: ~~~~~~~~~~~>
      ___     ___     ___
     / 1 \___/[2]\___/ 3 \          [X] = claimed cell
     \___/[4]\___/[5]\___/
     / 6 \___/[7]\___/ 8 \
     \___/ 9 \___/10 \___/

    Ownership check: "Is cell 4 mine?" -> YES (instant lookup)


=== ARBITRARY POLYGONS ===

    No pre-divided cells. Your run loop becomes the territory shape.

    Before your run:
    +----------------------------------+
    |                                  |
    |          (empty map)             |
    |                                  |
    +----------------------------------+

    After your run (you ran a loop):
    +----------------------------------+
    |                                  |
    |       .----.                     |
    |      /      \    <- your actual  |
    |     |  YOUR  |      run shape    |
    |      \      /                    |
    |       '----'                     |
    +----------------------------------+

    Now someone else runs an overlapping loop:
    +----------------------------------+
    |                                  |
    |       .----.                     |
    |      / YOUR \_.----.            |
    |     |    :////OVER  \           |
    |      \  :/// LAP   /            |
    |       '----''-----'             |
    |              ^                   |
    |              THEIR run shape     |
    +----------------------------------+

    Who owns the overlap? -> requires polygon intersection math
```

## Tradeoffs

```
+---------------------+-------------------+----------------------+
|                     | GRID-BASED        | ARBITRARY POLYGONS   |
+---------------------+-------------------+----------------------+
| Claiming logic      | Cell ID lookup    | Polygon geometry     |
|                     | O(1) per cell     | O(n) intersections   |
+---------------------+-------------------+----------------------+
| "Who owns this      | Instant           | Compute intersection |
|  spot?" query       | (hash map lookup) | against all polygons |
+---------------------+-------------------+----------------------+
| Storage             | cell_id + owner   | Full polygon coords  |
|                     | (tiny)            | (can be large)       |
+---------------------+-------------------+----------------------+
| Visual result       | Blocky, tiled     | Smooth, organic      |
|                     | (Minecraft-like)  | (looks hand-drawn)   |
+---------------------+-------------------+----------------------+
| Real-time updates   | Fast — flip a     | Slow — recompute     |
|                     | cell's owner      | merged polygon shape |
+---------------------+-------------------+----------------------+
| Low-end devices     | Light rendering   | Heavy rendering      |
|                     | (fixed shapes)    | (complex shapes)     |
+---------------------+-------------------+----------------------+
| Multiplayer overlap | Trivial — same    | Hard — polygon union |
|                     | cell, new owner   | and intersection     |
+---------------------+-------------------+----------------------+
| "Feels like a game" | Yes — clear tiles | Less gamey, more     |
|                     | to conquer        | personal/artistic    |
+---------------------+-------------------+----------------------+
| Works without loops | Yes — any run     | Needs a closed loop  |
|                     | claims cells it   | to form a polygon    |
|                     | passes through    | (or buffer around    |
|                     |                   | the path)            |
+---------------------+-------------------+----------------------+
```

## Which Is Better — By Scenario

| Scenario | Winner | Why |
|---|---|---|
| **Urban, dense city** | Grid | Many overlapping runners — polygon intersection becomes a nightmare at scale |
| **Rural, open areas** | Either | Low player density means overlap computation is rare |
| **Low-end Android devices (India-first)** | Grid | Less rendering work, less memory, less battery drain |
| **Real-time multiplayer (1v1 duels)** | Grid | Instant ownership flips, no geometry recomputation |
| **"Show off my epic run"** | Polygons | The exact shape of your marathon loop is more personal and shareable |
| **Leaderboards ("who owns the most area")** | Grid | Count cells. Done. No area calculation needed |
| **Anti-cheat validation** | Grid | Easier to validate "did you actually pass through this cell?" than "is this polygon shape legitimate?" |

## The Hybrid Option

You don't have to pick one exclusively:

```
Backend: Grid-based (hex cells via Uber's H3 system)
  -> Fast ownership, fast queries, fast leaderboards

Frontend: Render territories as smoothed polygons
  -> Take the grid cells a player owns
  -> Merge adjacent cells into a smooth visual boundary
  -> Display as an organic shape on the map

Result: Grid performance + polygon aesthetics
```

This is probably the best of both worlds for this app.

## What to Learn Next

- **Uber H3 hex grid system** — the industry-standard library for dividing the world into hex cells at multiple resolutions, which directly implements the grid approach
- **PostGIS spatial functions** — even with a grid, you'll use PostGIS for "find cells near this point" and rendering the smoothed boundaries
