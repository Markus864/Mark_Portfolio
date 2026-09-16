# ADR-0002: An endless world streamed from a deterministic seed

**Status:** Accepted

## Context

The first version of the world was a fixed chamber per site with invisible
borders. Play-testing feedback was blunt: "each map is the same layout,
that's not exploration." The game's promise is an expedition that keeps
going and grows over time.

## Decision

The world is a pure function of `(siteId, tileX, tileZ)`. Each 24-unit tile
is generated from a seeded PRNG keyed by a hash of those three values:
terrain displacement, crystal gardens, crags, a landmark on roughly a third
of tiles, a wild creature on roughly a third, and which stones glow (the
chance rises with distance from camp). Tiles within a radius of the explorer
are kept built; the rest are unloaded. Tile (0,0) is camp — the arch, the
path, the daily geode — and is never unloaded.

Only what the player *did* is saved: landmark keys logged, tile keys walked
(the map's fog of war), stones opened (by slot, with a regrow timer).

## Consequences

**Positive**

- Endless, and the same place when you walk back, at zero storage cost.
- Landmarks and species are tables: adding a sighting kind or a creature is
  a row, not a level.
- The expedition map draws directly from the saved keys.

**Negative and how they were handled**

- Building 25 tiles in one tick blanked the first frame for ~10 s. Tiles now
  build from a nearest-first queue, a couple per frame, with the tile under
  the explorer built immediately.
- A tile that unloads must not take the stone the player is walking to. The
  streamer pins camp plus the tiles holding the engaged and the approached
  stone.
- Wild creatures belong to their tile and are removed with it; a creature the
  player is walking up to is handled by the same pin.
