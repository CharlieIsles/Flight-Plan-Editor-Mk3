# Flight Plan Format Reference

This document describes the JSON flight-plan format used by this project, what
it represents in the real world, and how the numbers translate into physical
drone motion. It is written so that another tool (or an AI agent) can generate
new, valid flight plans without needing to reverse-engineer the viewer code.

The format itself is reverse-engineered from sample files and the
`flight-plan-viewer3.html` animation/timing code in this repo — it is not an
official spec from the drone/flight-controller vendor. Fields the viewer
doesn't use are marked as such below; treat them as pass-through metadata for
whatever real flight-control software ultimately consumes the file.

## 1. What this actually is

A flight plan describes a **drone camera flying a fixed, repeatable path
around a stationary car** (or similar object) that sits parked at a fixed
location — e.g. for automotive marketing photography/video, dealership
listings, or product walk-arounds. The car does not move; the drone flies a
smooth 3D path around it, pausing at points of interest to take photos or
record video clips (front 3/4 shot, wheel close-up, roofline pass, high
overhead reveal, etc.), then continues to the next shot.

The whole coordinate system is **relative to the car**, not to GPS/compass
coordinates. That means the same flight plan file can, in principle, be
"replayed" around any car parked at any real-world location/heading, as long
as the flight controller re-anchors the car-relative path to the car's actual
position and heading at flight time.

## 2. Top-level file structure

```json
{
  "version": 6,
  "description": "Human-readable name for this plan",
  "waypoints": [ { ... }, { ... }, ... ]
}
```

- `version` — integer schema version. This document covers **`version: 6`**
  only. Not otherwise interpreted by the viewer.
- `description` — free-text name/label for the plan, shown as the title in
  the viewer.
- `waypoints` — ordered array of waypoint objects (see §4). The drone visits
  them in array order. A valid plan needs at least 2 waypoints.

## 3. Coordinate system: anchor points + offsets

Rather than one global XY origin, each waypoint is defined relative to the
**nearest corner of a bounding rectangle drawn around the car**. This keeps
waypoint numbers small and roughly means "near the front-left of the car,
a bit further out and up" instead of large absolute coordinates.

Picture the car from directly above, nose pointing toward the "front"
direction. Its bounding rectangle has 4 corners, each given a one-letter
anchor code:

```
        FRONT
   a ─────────── d
   │      ▲      │
   │   (car,     │      a = front-left
   │   nose up)  │      b = rear-left
   │             │      c = rear-right
   b ─────────── c      d = front-right
        REAR
```

Using a local 2D axis system centred on the car (X = left(–)/right(+),
Y = front(–)/rear(+)), with `W` = half the rectangle's width and `L` = half
its length:

| anchor | meaning     | local (x, y)   |
|--------|-------------|----------------|
| `a`    | front-left  | (−W, −L)       |
| `b`    | rear-left   | (−W, +L)       |
| `c`    | rear-right  | (+W, +L)       |
| `d`    | front-right | (+W, −L)       |

Each waypoint then gives an `x-offset` and `y-offset` from its anchor. The
actual position of the waypoint is:

```
waypoint.x = anchor.x − x-offset
waypoint.y = anchor.y + y-offset
```

Important nuance: `x-offset` is subtracted using the **same global left/right
sign** for every anchor (it is not mirrored per-corner). In practice this
means:
- From anchors `a`/`b` (car's left side, x = −W), a positive `x-offset` moves
  the waypoint further left (outward, away from the car).
- From anchors `c`/`d` (car's right side, x = +W), a positive `x-offset`
  moves the waypoint left too — i.e. *inward*, back across toward the car's
  centreline/left side.

`y-offset` always moves the same way regardless of anchor: positive
`y-offset` moves the waypoint toward the rear, negative toward the front.
This axis is the "depth along the car" — front-to-rear position — and is
independent of left/right.

Altitude is a separate, third axis (see §4) — it's not part of the anchor
math at all.

**Real-world scale**: `x-offset` and `y-offset` are in **centimetres**. So a
half-width of 100 units = 1 m and a half-length of 220 units = 2.2 m — a
rough sedan-sized footprint (≈2 m wide × ≈4.4 m long overall).

Note that the viewer's own animation/timing code uses a slightly different
approximation internally (it treats a half-width of 100 units as ≈0.9 m,
giving ≈0.9 cm/unit instead of the true 1 cm/unit) purely to drive its own
on-screen playback speed. That approximation is a viewer implementation
detail, not the authoritative unit — treat **centimetres** as the real,
confirmed unit for `x-offset`/`y-offset` when generating or interpreting
plans, and scale accordingly (1 unit = 1 cm = 0.01 m).

Altitude's units aren't pinned down by the viewer (it never converts or
scales altitude — it just displays/uses the raw number). Given the value
ranges seen in samples (roughly 50–350) and that it shares the same offset
system, it is most likely centimetres too — but treat this as an assumption
unless confirmed.

## 4. Waypoint fields

Every waypoint is a flat JSON object. Fields observed across sample plans in
this repo:

| Field | Type | Unit / range | Meaning |
|---|---|---|---|
| `anchor-point` | string | `a` \| `b` \| `c` \| `d` | Which corner of the car's bounding rectangle this waypoint is offset from. See §3. |
| `x-offset` | number | centimetres (see §3) | Left/right offset from the anchor. |
| `y-offset` | number | centimetres (see §3) | Front/rear ("depth along the car") offset from the anchor. Unaffected by anchor choice. |
| `altitude` | number | likely cm (assumption) | Height of the waypoint. Higher = drone flies higher. |
| `speed` | number | m/s | Travel speed *of the leg departing this waypoint* (i.e. the speed used to fly from this waypoint to the next one). The speed on the very last waypoint is unused (nothing to fly to next). |
| `hover-time` | number | milliseconds | How long the drone pauses/hovers at this waypoint before continuing — this is when photos/video are actually captured. `0` means fly through without stopping. |
| `radius` | number | centimetres | The size of the waypoint in space — effectively a capture-trigger/arrival tolerance sphere around the waypoint's exact position, rather than a path-blending radius. **Not used by this viewer** — it draws a full Catmull-Rom spline through every waypoint's exact position regardless. Preserve it for the real flight controller. |
| `yaw-mode` | string | e.g. `local-heading` | How to interpret `yaw-value` — relative to the flight path / local frame rather than absolute compass heading. Every sample file uses `local-heading`. |
| `yaw-value` | number | degrees, roughly −180..180 | Drone/camera yaw (heading) at this waypoint. `0`-ish generally points camera at the car; sign flips mirror left/right framing (see the mirror-plan feature in `flight-plan-viewer3.html` for the exact reflection formula). |
| `yaw-change-mode` | string | e.g. `gradient` | How yaw transitions across the leg into this waypoint — `gradient` = smoothly interpolated, not a sudden snap. |
| `pitch-mode` | string | e.g. `local-heading` | Same idea as `yaw-mode` but for camera/gimbal pitch. |
| `pitch-value` | number | degrees | Camera tilt. Negative values point the camera downward (toward the car), `0` = level with horizon. |
| `pitch-change-mode` | string | e.g. `gradient` | Smooth interpolation of pitch across the leg, like `yaw-change-mode`. |
| `agile-altitude-change-mode` | string | e.g. `gradient` | Smooth interpolation of altitude across the leg (vs. an abrupt/stepped change). |
| `trajectory-mode` | string | e.g. `smooth` | Path shape between waypoints. `smooth` = curved spline through waypoints (this viewer always renders a Catmull-Rom spline regardless of this field — it doesn't currently branch on other trajectory modes, so treat other values, if any exist in the wild, as unsupported here). |
| `smooth-trajectory-coefficient` | number | practically always `0.0001` | Tension/shape parameter for the smooth trajectory curve. Use `0.0001` unless you have a specific reason to deviate. |
| `waypoint_name` | number | — | A human-assigned display index for the waypoint. **Not necessarily contiguous** — e.g. a sample plan jumps from waypoint 4 to waypoint 6 with no 5, presumably left over from editing/deleting a point in an authoring tool. Don't rely on this for ordering or count; use the array position instead. |
| `photo-name` | string | e.g. `"photo_1"` | Output filename for the photo captured at this waypoint. |
| `video-name` | string \| null | e.g. `"vid1"`, `null` | The name of the video clip that **ends at** this waypoint (not one that starts here). `null` means no video clip concludes at this waypoint. A single named clip can therefore span several preceding waypoints/legs of recording, with the name only appearing once — on the waypoint where that recording stops. |

None of `radius`, `waypoint_name`, `photo-name`, or `video-name` are read by
the viewer's animation — they're purely metadata for the real flight
controller / media pipeline. Preserve them unchanged (or update sensibly,
e.g. renumber `photo-name`) when programmatically transforming a plan, but
don't expect the viewer to reflect changes to them.

## 5. Motion & timing model

This is how `flight-plan-viewer3.html` animates a plan, and is a reasonable
model of the physical flight too:

1. For each consecutive waypoint pair, compute the straight-line distance
   between their anchor-resolved positions (converted to metres per §3).
2. Travel time for that leg = `distance_m / speed_of_departing_waypoint`.
3. After arriving at a waypoint, the drone hovers for that waypoint's
   `hover-time` (milliseconds) before the next leg begins. This is when
   `photo-name`/`video-name` capture happens.
4. Total flight duration = sum of all leg travel times + all hover times.

Position along each leg is **not** a straight line in practice — see §7.

## 6. Camera framing

Camera field of view is modeled (in the viewer) as a **24mm-equivalent lens
on a 35mm full-frame sensor**, giving a horizontal FOV of
`2 · atan(36/48) ≈ 73.74°`. This is used only for drawing the visual FOV cone
in the viewer/preview and isn't stored in the flight plan file itself — but
it's useful context for reasoning about what a given `yaw-value`/
`pitch-value` at a given position/altitude will actually capture in frame.

Yaw `0°` and pitch `0°` roughly correspond to "camera level, facing the
direction implied by the local-heading frame" — in observed plans, yaw
values are chosen per-waypoint so the camera keeps the car in frame from
whatever position that waypoint is at (i.e. yaw is aimed at the car, not
fixed to a compass direction). Pitch is typically negative (looking down at
the car) whenever altitude is high relative to the car's height, and closer
to `0` when the drone is near car height doing a level pass.

## 7. Path interpolation

- **Position**: the viewer fits a Catmull-Rom spline through all waypoints
  (in anchor-resolved, metres-equivalent space), so the drone's actual flown
  path curves smoothly through each waypoint rather than moving in straight
  segments. This matches `trajectory-mode: "smooth"`.
- **Yaw**: interpolated with shortest-path angular interpolation (i.e. it
  turns whichever way is fewer degrees, not always increasing/decreasing),
  consistent with `yaw-change-mode: "gradient"`.
- **Altitude/pitch**: implied to interpolate smoothly across each leg as
  well, consistent with `agile-altitude-change-mode: "gradient"` and
  `pitch-change-mode: "gradient"`.
- **Hover**: during a waypoint's `hover-time`, the drone's position/yaw/pitch
  are frozen at that waypoint's exact values — there is no motion during a
  hover.

## 8. Capture output naming

`version: 6` plans express capture intent as explicit output filenames
rather than booleans:

- `photo-name` is set on every waypoint that should capture a still photo
  (every sample waypoint has one — photo capture appears to happen at every
  stop).
- `video-name` marks where a video clip **ends**, not where it starts — it's
  the name of the clip that concludes at this waypoint. It's commonly `null`
  on most waypoints, with a real name appearing once at the waypoint where
  that recording stops. A `video-name` on an early waypoint (e.g. waypoint 1)
  can mean a clip that started before/at the beginning of the flight ends
  there, with the rest of the flight left unrecorded or covered by later
  named clips elsewhere in the plan.

## 9. Practical guidance for generating new plans

- Always provide at least 2 waypoints; the viewer (and presumably the real
  controller) rejects anything shorter.
- Pick the anchor (`a`/`b`/`c`/`d`) closest to where the waypoint actually
  sits, then use small offsets from it — this keeps offset numbers modest
  and legible, and mirrors how the sample plans were authored (offsets
  mostly in the tens-to-low-hundreds range, rarely more than ~600).
  Waypoints that are logically clustered around one side of the car
  typically all share the same anchor.
  - When mirroring a plan left-to-right, swap anchors `a↔d` and `b↔c`,
    negate `x-offset`, keep `y-offset`/`altitude` unchanged, and negate
    `yaw-value` (normalized back into range). See `mirrorWaypoint()` in
    `flight-plan-viewer3.html` for a working reference implementation.
- Keep `speed` and `hover-time` sensible for real drone flight — sample
  plans use `speed: 0.7` (m/s) and `hover-time: 1000` (ms) almost
  universally; large speed jumps between adjacent legs will look/feel
  abrupt.
- Keep `yaw-value` aimed so the car stays in frame from wherever that
  waypoint sits — a waypoint on the car's left side generally wants a yaw
  that points rightward, back toward the car, and vice versa.
- Reuse the fixed-default fields (`yaw-mode: "local-heading"`,
  `yaw-change-mode: "gradient"`, `pitch-mode: "local-heading"`,
  `pitch-change-mode: "gradient"`, `agile-altitude-change-mode: "gradient"`,
  `trajectory-mode: "smooth"`, `smooth-trajectory-coefficient: 0.0001`)
  unless you have a specific reason to deviate — every sample file in this
  repo uses these exact values with no variation.
- Set `photo-name` per waypoint following the plan's existing numbering
  (e.g. `photo_1`, `photo_2`, …), and leave `video-name` as `null` except on
  the waypoint where a recording actually ends — see §8.
- Don't renumber/rely on `waypoint_name` for logic — it can have gaps.
  Regenerate it sequentially if you want it tidy, but the array order is
  what actually matters.

## 10. Annotated example waypoint

```json
{
  "waypoint_name": 1,
  "anchor-point": "d",              // offset from the front-right corner
  "x-offset": -200,                 // shifted toward car's left side
  "y-offset": -180,                 // shifted toward the front of the car
  "altitude": 100,                  // moderate altitude
  "speed": 0.7,                     // 0.7 m/s to fly to the *next* waypoint
  "hover-time": 1000,               // pause 1s here to capture
  "photo-name": "photo_1",          // still photo captured at this stop
  "video-name": "vid1",             // a video clip named "vid1" ends at this waypoint (later waypoints use null)
  "yaw-mode": "local-heading",
  "yaw-value": 40,                  // camera yawed to keep the car in frame from this position
  "yaw-change-mode": "gradient",
  "pitch-mode": "local-heading",
  "pitch-value": -5,                // slight downward tilt
  "pitch-change-mode": "gradient",
  "agile-altitude-change-mode": "gradient",
  "trajectory-mode": "smooth",
  "smooth-trajectory-coefficient": 0.0001,
  "radius": 20                      // 20cm capture/arrival tolerance around this waypoint (unused by this viewer)
}
```
