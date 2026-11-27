# Simulation: tanks, ground and collisions

How a design becomes a moving tank, how it follows the hills, and what happens when it hits
something. The aim throughout is "believable and predictable" rather than a full physics
simulation: a tank should behave the way you expect every time.

## From a design to driving numbers

Every tank, ready-made or designed by the player, is the same set of choices (size, hull shape,
turret, gun, barrel, engine, six armour plates, skirts), turned into numbers by one function.

**Weight is built up from the parts:**

```
mass = hull structure + armour steel + gun + engine + skirts
armour steel = Σ plate area (m²) × thickness (m) × 7850 kg/m³
engine = 1.5 kg per hp
```

A sloped plate is longer than a vertical one, so it weighs more: slopes are not free.

**Mobility comes from power-to-weight** (hp per tonne):

| | Formula |
|---|---|
| Top speed | `1 + 0.55 × hp/t` m/s, clamped to 4–20 m/s (14–72 km/h) |
| Pull from standstill | `0.1 × hp/t` m/s², clamped to 0.8–4 |
| Hull turning | `40 × √(hp/t ÷ 20)` °/s, scaled by hull size, clamped to 12–60 |

The class the AI uses (light, medium, heavy) comes from the final mass. The twelve historical
tanks were given the engine power that reproduces their real top speed with their modelled
weight, so a Tiger I does 45 km/h because of its 53 modelled tonnes and its engine, not because
it was told to.

## Driving model

The hull is a rigidbody, but it is **driven by setting its velocity**, not by pushing it with
forces. That is less realistic than a wheel-and-suspension model, but it is stable, it gives
the same result every time, and the tank still collides with everything.

- **Acceleration fades with speed**: `dv/dt = a × (1 − 0.7 × v / v_max)`. At top speed only
  30% of the starting pull is left, so a tank needs 8–10 s to reach 90% of its top speed
  instead of two or three. The time is shown in the tank details window
  (`t = −ln(1 − 0.9 × 0.7) × v_max / (0.7 × a)`).
- **Braking** is three times the pull (about 0.6 g).
- **Reversing** swaps left and right, so the tank backs towards the side you steer to, like a
  car with its wheel turned.
- **A broken track** stops the tank for six seconds while it is repaired.

## Following the ground

The terrain is a height function, so a tank needs no colliders under it to stay on the ground.
Every physics step:

1. The ground height is sampled at four points: front, back, and under each track.
2. Those give a slope along the hull and across it. Their cross product is the ground normal,
   smoothed slightly so the hull does not jitter over small bumps.
3. The hull is turned to its heading **lying along that normal**. The heading is kept as its
   own number, separate from the tilt, so going over a hill never changes which way the tank
   points.
4. The vertical velocity is set so the tank arrives at the average ground height under its
   tracks at the end of the step, looking one step ahead so it does not lag on fast descents.

**Slopes change speed** in the direction of travel: uphill `1 − 1.6 × climb` (a 26° slope
leaves about a third of the speed), downhill up to 16% faster. Anything steeper than 26° is
marked unwalkable for the AI.

**Fallen logs** raise the height the tracks rest on with a smooth ramp, so a tank climbs over a
log instead of being stopped by its collider.

## The terrain

Heights are a few layers of smooth value noise (large hills with smaller bumps on them),
eased flat near the walls and kept gentle where the teams start. Capture bases and every
building get a **pad**: the ground under them is flattened and blended smoothly into the hills
around it, so houses stand level.

After the map is generated, the heights are sampled into a 2.5 m grid. The **same triangles**
are used for the visible mesh, its collider, and every height query, so a tank sits exactly on
the ground you see. Everything else asks the same function too: where props stand, the
camera's minimum height, the AI's walkable cells and the minimap's hill shading.

## Collisions

Each tank has two kinds of colliders on separate physics layers:

| Layer | What it is | Collides with |
|---|---|---|
| Chassis | one box around the hull and tracks | other tanks, buildings, rocks (not the ground: the tank follows the height function) |
| Armour | the exact hull and turret shapes | nothing; only shells and sight lines look for them |

This keeps the physics cheap and stable while shells still hit the real sloped shapes.

### Ramming

When two tanks crash, the crash energy damages both:

```
closing speed  from how each tank really moved over the last step (not what it was asked to do)
energy         ½ × m₁m₂ / (m₁ + m₂) × v²
damage         0.3 per kJ, shared by weight: the lighter tank takes more
               the rammer meets it with its front and takes half of its share
```

Below 9 km/h it is just a bump. The same two tanks cannot hurt each other again for two seconds,
and team-mates only push each other. Examples: a 57 t tank hitting a 34 t tank at 30 km/h
does 278 damage to the lighter one and 83 to itself; a 14 t tank hitting a 57 t one at 60 km/h
does 185 to the heavy one and 376 to itself.

### Scenery that falls over

Trees, fence sections, poles and street lamps have a **trigger** collider, which every raycast
in the game ignores, so they never stop shells or block sight. When a tank drives into one, it
tips over away from the tank. The harder the hit, the faster it goes: a fast tank snaps a tree
down in about half a second, a crawling one pushes it over slowly. Trees also slow the tank
more and crack loudly when they break.

## Simulation mode

Besides playing, a battle can be run with **AI on both sides** and watched:

- step through every tank with E / Q, with a line showing what its AI is doing
- fly a free camera anywhere (WASD level with the ground, Space / C up and down)
- run the game at ×0.25 to ×3 speed

It was the main tool for tuning the AI and the balance: twenty tanks fighting at triple speed
show a stalled battle, a tank that keeps getting stuck, or an overpriced design within minutes.

## What went wrong along the way

- **Ramming that never stopped.** Damage used to come from the velocity each tank was *trying*
  to reach, so a tank pushing against another kept doing damage every frame. It now uses the
  velocity measured from actual positions (pushing moves both tanks together, so the closing
  speed is zero), the crash energy instead of a flat number, and a cooldown per pair.
- **Tanks trapped between rocks.** Long canyon ridges were spaced by checking only their ends
  and middle, so two ridges could almost touch halfway along and trap a tank in the wedge
  between them. Placement now checks the whole footprint with a gap wider than any tank, and a
  stuck tank (player or AI) can be moved to the nearest open ground.
