# Route Planner — Norway

Draw running and cycling routes that snap to real roads, cycleways and trails, with
distance, elevation gain, and a body-weight-based fueling plan.

It is a **single static HTML file**. No build step, no npm, no API keys, no server.

## Running it

Either double-click `index.html`, or serve the folder over HTTP:

```sh
python -m http.server 8000
```

Both work — the app is deliberately built without ES modules so the `file://` route
isn't blocked by CORS. Routing and elevation were verified working from both.

To host it, drop `index.html` on GitHub Pages, Netlify, Cloudflare Pages, or any static
host. That's the whole deployment.

## What it does

- **Snapped routing** for road running, trail running, road / gravel / MTB cycling and
  touring. Access rules are respected: no motorways, and nothing tagged `bicycle=no`
  or `foot=no` for the relevant mode.
- **Distance and elevation** gain/loss, using Norwegian 1 m lidar terrain data.
- **Elevation profile** chart, hovering it highlights that point on the map.
- **Transport segments** — mark any leg as a ferry, train or lift. It draws as a dashed
  line and is left out of distance, climb, duration and fueling, with its distance
  reported separately. Tick "Next segment is transport" before adding the far-side
  point, or toggle any existing segment in the list.
- **Fueling plan** — carbohydrate, fluid and sodium, scaled to body weight, estimated
  duration, temperature and effort.
- **GPX export** for Garmin / Wahoo / Strava.
- **Save routes** in the browser (see the caveat below).
- Click the map to add points, drag markers to reshape, click a marker to delete it.
- **Shape a route mid-way** — grab the route line anywhere and drag it onto the road you
  want; a waypoint is created at the drop point in one motion. Clicking the line without
  dragging inserts a point in place, which is also what a tap does on touch.
- **Undo** steps back through what you actually did — insert, move, delete, reverse,
  transport toggle, activity change, and clear.
- **Preview mode** — a figure travels the route, scrubbed with a slider or played back.
  It moves at modelled speed, so it visibly slows on climbs. The readout gives distance,
  elevation, gradient, elapsed and remaining time, and the fuel you should have taken in
  by that point; cue markers along the route show where each gel and bottle comes due.
  Editing is disabled while previewing, so scrubbing can't drop stray waypoints.

## Services it depends on

| Purpose | Service | Terms |
|---|---|---|
| Routing | [BRouter](https://brouter.de/) public server | Free, no API key |
| Map tiles | [Kartverket](https://www.kartverket.no/) WMTS + OpenStreetMap | CC BY 4.0 / ODbL |
| Elevation | [Kartverket høydedata](https://ws.geonorge.no/hoydedata/v1/) | CC BY 4.0, no auth |

Attribution is rendered in the sidebar footer, as those licences require.

## Caveats worth knowing

**The routing server is volunteer-run.** `brouter.de` has no SLA and no published rate
limit. During development its rate limiter was triggered by rapid automated requests
("operation killed by thread-priority-watchdog") — normal human clicking is very
unlikely to hit this, but it can happen. The app handles it by showing the error rather
than failing silently, caching each leg so nothing is re-fetched needlessly, and storing
saved routes with their geometry so reloading one makes no network calls at all.

If you ever need to move off the public server, every routing request goes through the
single `routeLeg()` function in `index.html` — point it at a self-hosted BRouter
(available as a Docker image) or another engine and nothing else has to change.

**Saved routes live only in this browser, on this device.** They use `localStorage`.
Clearing site data deletes them. Export a GPX for anything you want to keep.

**The fueling numbers are population guidelines, not individual advice.** Sweat rate,
sodium loss and gut tolerance vary roughly two-fold between people of the same weight.
Use the output as a starting point and test it in training.

## How the numbers are calculated

**Elevation gain** is computed from the merged coordinate array for the whole route,
not by summing each leg's figure — a per-leg sum would drift depending on how the route
happened to be split. A 2 m hysteresis threshold suppresses data jitter, so tiny
wobbles don't inflate the total. A closed loop correctly returns identical ascent and
descent.

**Elevation data** comes from Kartverket's `dtm1` (1 m lidar), fetched in batches of 50
points. Where coverage runs out, the app falls back to BRouter's SRTM values per point
and tells you. Note that Kartverket's coverage extends somewhat past the national
border, so routes near it usually still get the good data.

**Water is handled specially.** Over sea, Kartverket returns the *seabed depth*, not a
surface height — Sognefjord comes back as −1080 m. Taken at face value that sends any
fjord crossing to the bottom of the sea and back up. Those points are detected by their
`datakilde: "dybdekurver"` / `terreng: "Havflate"` markers and their elevation is
interpolated across from the land on either side. This matters for **bridges** as much
as ferries: simply falling back to SRTM would read ~0 m over water and make a high
bridge look like a deep dip. On the real Moss–Horten crossing this took the figures
from 185 m climb / −167 m low point down to 8 m climb / 0 m.

In testing, Kartverket read about 3% higher than SRTM over a 7 km Nordmarka route
(339 m vs 328 m). It is the authoritative national dataset and costs about 400 ms, so
it is worth using — but BRouter's built-in European elevation data is already decent,
and you should not expect dramatically different totals.

**Running duration** uses Minetti's energy cost of running as a function of gradient —
the same curve behind "grade adjusted pace". The raw energy curve implies you would run
~40% faster on a -10% descent, which nobody sustains, so downhill is damped: never
faster than 0.8× flat pace, and very steep descents get slower again.

**Cycling duration** solves a power balance per segment. Your stated flat speed
determines your power (28 km/h ≈ 164 W for a 75 kg rider), and each segment's speed is
solved from that against gravity, rolling resistance and drag. Descent speed is capped
by activity — 60 km/h road, 45 gravel, 32 MTB — since descents are limited by
confidence and surface, not physics. On a capped descent the rider is treated as
coasting and charged no energy.

**Preview pacing** reuses the same per-segment model rather than a second copy:
`estimateEffort()` emits cumulative time at every point, and the preview drives position
from it. On a Nordmarka test route that gives 10.9 min/km on climbs, 6.6 on the flat
(against a stated 6.5 flat pace) and 5.2 on descents.

Preview keeps three separate clocks, because they measure different things: distance
*including* transport (what the scrubber travels), moving seconds *excluding* transport
(what the readout reports, matching the stats), and a playback timeline that gives a
ferry of unknown real duration a short fixed slice so the figure visibly crosses it.

**Fueling** is duration-banded for carbohydrate (nothing under ~75 min, rising to
80–100 g/h beyond 3 h, reduced ~10% for running since the gut tolerates less while
running). Fluid starts from ~7 ml/kg/h and is adjusted for temperature and effort,
clamped to 300–1000 ml/h. Sodium is 500 mg/L, or 900 mg/L for salty sweaters.

## Verification performed

- Browser CORS confirmed for BRouter and Kartverket, over both `http://` and `file://`.
- Snapping checked through Nordmarka forest — the route follows marked trails.
- Access rules checked Oslo→Lillestrøm: the car profile used 15 motorway and 43 trunk
  segments; the bike and run profiles used **zero** motorway, zero `bicycle=no` and
  zero `foot=no` respectively.
- Leg caching: adding a waypoint costs 1 routing request, dragging a middle waypoint
  costs exactly 2.
- Closed loop returns ascent == descent exactly.
- GPX output parses as valid XML with elevation on every point, and escapes XML
  metacharacters in the route name.
- Saved route reloads with **0 network requests**, and editing it afterwards correctly
  rebuilds only the two affected legs.
- Failure modes: network down, server error, and Kartverket unavailable all behave
  correctly — the first two show a clear message, the third silently falls back to SRTM
  and still produces a route.
- Sea depth: the Moss–Horten ferry crossing reads 8 m climb / 0 m low point, against
  185 m / −167 m before the fix.
- Long routes: a 155 km route exports all 5847 points, and reconstructing distance from
  the downloaded GPX gives back 155.4 km. A synthetic 400 km / 40 000-point route exports
  complete at 2.66 MB. Saving a 150 km route uses ~0.15 MB of `localStorage`, so roughly
  30 of them fit the browser quota.
- Preview: scrubbing to 50% lands at exactly half the path distance; the figure
  interpolates in ~14 m steps rather than snapping between nodes; playback at 500x runs
  13.28 s against a predicted 13.25 s and takes exactly 2.5x longer at 200x; climbs are
  measurably slower per metre than flat ground; cue counts match `computeFueling`
  exactly; a sub-75-minute route produces no cues without dividing by zero; crossing a
  ferry shows a boat and freezes elapsed time; and five enter/exit cycles leak no layers.
- Mid-route insertion: dragging the line moves the route (3.62 → 3.91 km) and places the
  waypoint at the drop point; clicking without dragging inserts on the line with distance
  unchanged to 3 decimals. Both cost exactly 2 routing requests, neither appends a stray
  end point, and the trailing click a drag produces does not insert a second time.
  Releasing outside the map aborts cleanly and leaves the map draggable.
- Undo: 10 mixed edits (append, insert, drag, delete, reverse, transport toggle, activity
  change, clear) undo back to an empty route with **zero network calls**, since legCache
  serves every replayed leg.
- Splitting a transport leg yields two transport halves, with ride and transport distance
  both unchanged.
- Transport segments: excluding the Moss–Horten leg moves a 25.25 km route to 12.79 km
  ridden plus 10.17 km transport, and the time estimate from 1h00m to 0h33m. GPX splits
  into separate `trkseg`s, saved routes keep their flags across a reload with zero
  network calls, an all-transport route reports zeroes without NaN, and deleting a
  waypoint beside a ferry merges conservatively to a ride rather than silently
  swallowing the crossing.
