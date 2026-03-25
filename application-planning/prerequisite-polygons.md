# Polygons — Shapes on Maps

## One-Liner

A polygon is a closed shape made by connecting coordinate points — it's how software represents areas like parks, buildings, and territories on a map.

## Prerequisites

- GPS tracking and lat/long coordinates — See: [prerequisite-gps-tracking.md](prerequisite-gps-tracking.md)

## What Is It?

A polygon is a closed shape with straight edges. In map software, you define a polygon by listing its corner points (vertices) as lat/long coordinates. The system connects the dots in order and closes the shape by linking the last point back to the first. Any area you see highlighted on Google Maps — a park boundary, a building footprint, a city border — is a polygon.

## Real-World Analogy

Think of a polygon like a fence around a property. You hammer posts (coordinate points) into the ground at each corner, then run wire (edges) between each post in order. The last post connects back to the first one, and now you have a closed boundary. Everything inside the fence is "your area." The shape of your property depends entirely on where you placed the posts.

## Lines vs Polygons — Open vs Closed

```
=== LINE (Open Shape) ===

    A --> B --> C --> D              Just a path. No area.
                                    Your GPS trace during a run
    *----*----*----*                 is a line — it has length
                                    but doesn't enclose anything.


=== POLYGON (Closed Shape) ===

    A --> B --> C --> D --> back to A    A closed boundary. Has area.

    *----*                              The fence closes, so now
    |    |                              there's an "inside" and
    |    |                              an "outside."
    *----*

    Key difference: a polygon's last point connects back to
    the first point, creating an enclosed area.
```

## How a Polygon Is Stored in Software

```
A rectangle park on a map:

  Point A (13.08, 80.27) -------- Point B (13.08, 80.28)
         |                                |
         |          PARK AREA             |
         |                                |
  Point D (13.07, 80.27) -------- Point C (13.07, 80.28)

Stored as a list of coordinates:
polygon = [
    (13.08, 80.27),    // Point A (top-left)
    (13.08, 80.28),    // Point B (top-right)
    (13.07, 80.28),    // Point C (bottom-right)
    (13.07, 80.27),    // Point D (bottom-left)
    (13.08, 80.27)     // Back to A — closes the shape
]

That's it. A polygon is just a list of coordinates where
the last point = the first point.
```

## Polygon Operations — The Important Ones

These operations are why polygons matter (and why they're hard) for a territory app.

```
=== 1. AREA — How big is this shape? ===

    Given the polygon's coordinates, calculate the
    area inside it. This tells you how much territory
    a player owns.

    +-------+
    |       |  Area = width x height (for a rectangle)
    |  42   |  For complex shapes, the math is harder
    | sq km |  but the database (PostGIS) handles it.
    +-------+


=== 2. CONTAINS — Is this point inside the shape? ===

    "Did this runner pass through my territory?"

    +-------+
    |       |
    |   * <----- YES, this point is inside
    |       |
    +-------+
                  * <-- NO, this point is outside

    This is a common operation: checking if a GPS point
    falls inside a territory polygon.


=== 3. INTERSECTION — Where do two shapes overlap? ===

    "Where does my territory overlap with yours?"

    +-------+
    |   A   |
    |    +--+----+
    |    |//|    |     The shaded //  area is the
    +----+--+    |     INTERSECTION — the overlap
         |   B   |     between polygon A and polygon B.
         +-------+

    This is EXPENSIVE to compute. With thousands of
    players, checking every polygon against every other
    polygon gets slow fast.


=== 4. UNION — Merge two shapes into one ===

    "Combine all my run territories into one big shape"

    +-------+
    |   A   |
    |    +--+----+          +------------+
    |    |  |    |    =>    |            |
    +----+--+    |          |   MERGED   |
         |   B   |          |            |
         +-------+          +------------+

    Takes two overlapping polygons and produces one
    combined polygon. Even more expensive than
    intersection because the result is a new, more
    complex polygon.


=== 5. DIFFERENCE — Subtract one shape from another ===

    "Someone stole part of my territory"

    +-------+                +---+
    |   A   |                | A |
    |    +--+----+    =>     |   +-------+
    |    |  | B  |           |           |
    +----+--+    |           +---+       |
         |      |                |   B   |
         +------+                +-------+

    A minus the overlap = what's left of A after B takes over.
```

## Why This Is Hard for a Territory Running App

```
Scenario: 500 runners in Chennai, each with a unique territory polygon.

New runner finishes a run. Need to check:
- Does the new polygon INTERSECT with any of the 500 existing ones?
- If yes, compute the INTERSECTION for each overlap
- Compute the UNION of the new runner's territories
- Update the AREA calculations for affected territories
- Do all of this in real-time so the map updates live

500 polygons = manageable
50,000 polygons = slow
500,000 polygons = breaks

This is exactly why grid-based systems exist — they avoid
these expensive polygon operations entirely by using simple
cell ID lookups instead.
```

## Diagram — How a Run Becomes a Territory Polygon

```
Step 1: You run a loop (GPS trace = line)

    Start/End
        *
       / \
      *   *
      |   |
      *   *
       \ /
        *

Step 2: The app detects it's a closed loop

    "First point and last point are close together"
    -> Convert the line into a polygon (close the shape)

Step 3: The polygon is your territory

        *
       /|\
      * | *
      | | |        The area INSIDE the loop
      * | *        is now "yours" on the map
       \|/
        *

Step 4: Problem — what if your run isn't a loop?

    You ran from A to B in a straight line:
    *----*----*----*

    That's a line, not a polygon. No enclosed area.

    Solutions:
    a) Only count loop runs (limiting)
    b) Add a "buffer" — inflate the line into a thin polygon:

       +--------------+
       |*----*----*---|*    <- thin strip around your path
       +--------------+

    c) Use a GRID system — no polygons needed at all,
       just mark which cells you passed through
```

## Where & How to Use

- Every territory on the map is either a polygon (arbitrary shape) or a collection of grid cells (which are themselves small polygons)
- Understanding polygon operations (intersection, union) is key to understanding why grid-based systems are faster
- PostGIS (the database extension) handles all these polygon operations — you don't write the math yourself

## What to Learn Next

- **Grid-based vs arbitrary polygon territory systems** — now that you understand both GPS and polygons, you can fully grasp the tradeoffs. See: [grid-vs-polygons.md](grid-vs-polygons.md)
