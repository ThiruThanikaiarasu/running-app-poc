# GPS Tracking — How Your Phone Knows Where You Are

## One-Liner

Your phone listens to signals from satellites in space, does some math on the time those signals took to arrive, and figures out your exact spot on Earth — recorded as two numbers: latitude and longitude.

## Prerequisites

- None. Starting from zero.

## What Is It?

GPS (Global Positioning System) is a network of ~30 satellites orbiting Earth. Your phone has a GPS chip that picks up signals from these satellites. By comparing signals from at least 3-4 satellites, it calculates your position on Earth. That position is stored as two numbers — **latitude** (how far north or south you are) and **longitude** (how far east or west you are). When you go for a run with a tracking app, it records your lat/long every few seconds, creating a trail of points — your **GPS trace**.

## Real-World Analogy

Imagine you're blindfolded in a huge field, and three friends are standing at known positions. Each friend shouts your name at the same time. You hear Friend A first (they're closest), then Friend B, then Friend C. From the time difference between hearing each voice, you can figure out how far you are from each friend — and from that, exactly where you're standing.

GPS works the same way, except your "friends" are satellites, and instead of sound, they send radio signals. Your phone measures the tiny time differences between signals to calculate your position.

## What Are Latitude and Longitude?

Think of Earth wrapped in a grid of invisible lines.

- **Latitude** = horizontal lines. They tell you how far **north or south** you are from the equator. Ranges from -90 (South Pole) to +90 (North Pole). Chennai is roughly 13.0 N.
- **Longitude** = vertical lines. They tell you how far **east or west** you are from a line through London (the Prime Meridian). Ranges from -180 to +180. Chennai is roughly 80.2 E.

Together, (13.0, 80.2) pinpoints Chennai on the planet. Every location on Earth has a unique lat/long pair.

## Example — A GPS Trace From a Run

```
You start a run at 6:00 AM. Your app records your position every second.

Time        Latitude    Longitude     What's happening
------      --------    ---------     ----------------
6:00:00     13.0827     80.2707       Standing at start (Marina Beach)
6:00:01     13.0827     80.2708       Started moving east
6:00:02     13.0828     80.2709       Picking up pace
6:00:03     13.0829     80.2710       Running along the beach
6:00:04     13.0830     80.2710       Turned slightly north
...
6:30:00     13.0827     80.2707       Back at start

That's 1,800 points (30 min x 60 seconds).
Connect the dots and you get your run route on the map.
```

## Diagram

```
=== HOW GPS POSITIONING WORKS ===

        Satellite A              Satellite B             Satellite C
           /\                       /\                      /\
          /  \                     /  \                    /  \
         / sig \                  / sig \                 / sig \
        / nal   \                / nal   \              / nal    \
       /         \              /         \            /          \
      v           v            v           v          v            v
      |--120ms----|            |--95ms-----|          |--110ms-----|
      |           |            |           |          |            |
      |           +------------+-----------+----------+            |
      |                        |                                   |
      |                    YOUR PHONE                              |
      |               (calculates position                         |
      |                from time differences)                      |
      |                        |                                   |
      |                        v                                   |
      |                  Lat: 13.0827                               |
      |                  Lng: 80.2707                               |


=== WHAT A GPS TRACE LOOKS LIKE ===

    Your phone records a point every second as you run:

    Start
      *
       \
        *---*
             \
              *---*---*
                       \
                        *
                       /
              *---*---*
             /
        *---*
       /
      *
    End (back at start)

    Each * is one GPS point (lat, lng, timestamp).
    Connect them = your run route.


=== LATITUDE AND LONGITUDE GRID ===

                     Longitude (East-West)
                 -180        0        +180
                   |         |         |
            +90  --+---------+---------+-- North Pole
                   |         |         |
                   |    * <- |London   |
    Latitude       |         |         |
    (North-South)  | Chennai |-> *     |
                   |         |         |
            -90  --+---------+---------+-- South Pole
                              ^
                        Prime Meridian
```

## GPS Accuracy and Drift — Why It Matters

GPS isn't perfect. Your phone's position can jump around even when you're standing still.

- **Good conditions** (open sky, clear weather): accurate to ~3-5 meters
- **Urban areas** (tall buildings): signals bounce off buildings, accuracy drops to 10-30 meters. This is called the "urban canyon" effect.
- **Indoors/tunnels**: GPS barely works. Can be off by 50+ meters.
- **Cheap phone GPS chips**: lower accuracy than flagship phones

**Why this matters for a running app:**

```
Reality:       You ran in a straight line down a road
GPS recorded:  A wobbly, zigzag path (because of drift)

    Actual path:     *----*----*----*----*
    GPS recorded:    *--*   *--*  *---*--*
                        \  /      \  /
                         *         *     <- "drift" points

This means:
- Your distance might show 5.2km when you actually ran 5.0km
- Territory claimed might include areas you didn't actually run through
- On low-end devices, drift is worse — a real problem for India-first
```

Apps like Strava handle this by filtering out impossible jumps (you can't teleport 100 meters in 1 second) and snapping your route to known roads.

## Where & How to Use

- Understanding GPS traces is fundamental to the territory app — every feature depends on knowing where the user ran
- GPS accuracy directly impacts territory fairness — need drift filtering and validation
- Background GPS tracking (recording while phone is in pocket) is the core technical challenge on both Android and iOS

## What to Learn Next

- **Polygons on maps** — now that you know GPS gives you points, learn how those points form shapes (territories) on a map. See: [prerequisite-polygons.md](prerequisite-polygons.md)
- **Grid vs Polygon territory systems** — the two approaches to turning GPS traces into owned territory. See: [grid-vs-polygons.md](grid-vs-polygons.md)
