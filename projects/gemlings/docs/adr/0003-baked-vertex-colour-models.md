# ADR-0003: Creature models baked to vertex colours through a custom GLB writer

**Status:** Accepted

## Context

Fifty-five species need 3D models. Hand-modelling and rigging them is out of
reach for one developer. Hosted image-to-3D generation turns a painted
portrait into a model in under a minute — but the output is a ~1.4 MB
textured GLB per species, and texture loading on expo-gl is its own project
(bundled images must be copied out of the APK and handed to the GPU as a
file-backed texture).

## Decision

Bake every model to **vertex colours** and ship no textures for creatures:

1. Subdivide the generated mesh to ≥15k vertices (faces stop reading below
   that).
2. Sample the texture into per-vertex colour.
3. Decimate to 16k faces, then resample colour from the original by KD-tree
   lookup.
4. Normalise height to 1 with feet at y=0, and orient faces outward.
5. Write the GLB with a **custom exporter** emitting float32
   `POSITION / NORMAL / COLOR_0` (linear RGB) and uint32 indices.

## Consequences

**Positive**

- ~500 KB per species, 27 MB for all 55 — bundled in the base build with no
  asset pack.
- Zero texture plumbing on the device; a model is a parse and a material.
- The explorer uses the same bake, which is what later made a rig-free walk
  cycle possible: the single mesh is split by triangle into limbs at load
  time.

**Negative**

- The off-the-shelf exporter stored colours as normalised bytes with no
  normals and the models rendered **black** in three. Hence the custom
  writer — a couple of hundred lines that have never needed changing since.
- A bake can come out dark; a per-model gain/gamma pass exists for that.
- Faceted detail is capped by the 16k budget. Rarity is shown on the body
  through emissive intensity, metalness, and (for one tier) a slow hue shift,
  rather than through geometry.
