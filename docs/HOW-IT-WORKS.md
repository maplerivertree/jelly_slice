# How Jelly Slice is built

This is a guided tour of [`index.html`](../index.html): what each part does, *why* it is done that way, and the numbers that make it feel right. You should be able to rebuild it — or bend it into your own jelly — from this document.

**Contents**

1. [The big picture](#1-the-big-picture)
2. [Making it look like jelly](#2-making-it-look-like-jelly)
3. [Building the shape](#3-building-the-shape)
4. [Making it wobble: shape-matching physics](#4-making-it-wobble-shape-matching-physics)
5. [Touching it: grab, toss, poke](#5-touching-it-grab-toss-poke)
6. [Collisions](#6-collisions)
7. [The knife and cutting](#7-the-knife-and-cutting)
8. [The kitchen](#8-the-kitchen)
9. [Performance](#9-performance)
10. [Tuning cheat-sheet](#10-tuning-cheat-sheet)
11. [Ideas for extending it](#11-ideas-for-extending-it)
12. [References](#references)

---

## 1. The big picture

Everything lives in one file. The `<script type="module">` is split into sections marked `// ---------- name ----------`, in this order:

| Section | Responsibility |
| --- | --- |
| fruit palettes | colours for each citrus variety |
| renderer / scene | Three.js renderer, camera, orbit controls, lights |
| kitchen | procedural textures + props (counter, board, bowl, jar, basil, towel) |
| citrus slice textures | canvas-painted colour / transmission / bump maps |
| 2D convex polygon helpers | clip, inset, area, centroid, closest point |
| slab geometry | turns a 2D outline into a rounded 3D mesh |
| mat3 / quat helpers | tiny maths kit, incl. rotation extraction |
| jelly material | the single shared `MeshPhysicalMaterial` |
| `class Piece` | one soft body: its mesh, particles, clusters and solver |
| world | the list of pieces, piece-vs-piece contact, the blade collider, `worldStep` |
| cutting | `splitPiece` / `performCut` + nearest-neighbour state transfer |
| the knife | knife model, cut animation timeline |
| UI / pointer | tools, fruit chips, firmness, grab/toss/poke/stroke |
| loop | fixed-timestep simulation + render |

Each frame:

```
real time ──► accumulator ──► for each 1/60 s tick (max 3 per frame):
                                  update knife animation  → blade collider (or null)
                                  6 × worldStep(h = 1/360 s)
                              ──► copy particles into vertex buffers, recompute normals
                              ──► render
```

A key design choice: **the mesh vertices *are* the physics particles.** There is no separate simulation mesh and no skinning — `piece.x` is literally the `position` attribute's `Float32Array`. That keeps the code short and means anything the physics does is exactly what you see.

---

## 2. Making it look like jelly

### The material

All pieces share one `THREE.MeshPhysicalMaterial`:

| Property | Value | Why |
| --- | --- | --- |
| `transmission` | `1` | light passes *through*: the board's grain is visible, bent, behind the jelly |
| `ior` | `1.42` | index of refraction close to gelatin/sugar syrup — controls how much the background bends |
| `thickness` | `0.55` | how "deep" the refraction is assumed to be |
| `attenuationColor` / `attenuationDistance` | fruit colour / `0.7` | Beer–Lambert absorption: light that travels further through the jelly gets more saturated. This is what makes it read as a coloured solid, not tinted glass |
| `roughness` | `0.1` | slightly soft reflections |
| `clearcoat` / `clearcoatRoughness` | `1` / `0.06` | a second, sharper specular layer — the wet sugary gloss |
| `map`, `transmissionMap`, `bumpMap` | canvas textures | the citrus pattern (below) |

Three.js implements transmission by first rendering the opaque scene to a texture, then sampling it with an offset from the refracted view ray. That's why the knife blade, once it goes *into* the jelly, appears refracted for free — it's part of the opaque pass.

The scene has an environment map generated from Three's `RoomEnvironment` via `PMREMGenerator`, so the clearcoat has something to reflect even though there's no HDRI file.

### Painting the slice (`paintCitrus`)

Three 1024² canvases are painted with the 2D canvas API every time you pick a fruit:

1. **Colour map** — peel colour everywhere → pith disc (93% radius) → flesh disc (86%) with a radial gradient → ~2600 tiny semi-transparent ellipses oriented radially (the juice vesicles) → 10 slightly curved membrane strokes → a pith dot in the centre.
2. **Transmission map** — same layout but in greys: flesh is white (fully clear), membranes light grey, pith and peel darker (more opaque). The material multiplies `transmission` by this, so the rind looks solid while the flesh glows.
3. **Bump map** — vesicles are lighter blobs and membranes are white ridges, so the surface catches light like real fruit flesh.

A seeded random generator (`seed * 16807 % 2³¹−1`) keeps the pattern identical between reloads.

### Slice-space UVs: why cuts line up

UVs are **planar**, computed from the vertex's *rest* position in the original slice's frame:

```
u = 0.5 + 0.49 · x / R        v = 0.5 + 0.49 · z / R
```

Every vertex — top, bottom, rounded edge, and later every piece produced by a cut — uses this same mapping. So a piece cut from the left half shows exactly the left half of the pattern, and a fresh cut face (which lies *inside* the original disc) automatically shows flesh and membranes rather than peel. No texture bookkeeping is needed when cutting.

---

## 3. Building the shape

### A convex outline, inset, then inflated

Every piece is described by **one convex 2D polygon** in the slice's x–z plane (the `outline`). The whole slice starts as a 72-gon of radius 1.

To get rounded gummy edges, the 3D shape is the outline *shrunk* by `C` (half the thickness) and then *swept by a sphere* of radius `C` — mathematically the Minkowski sum of a flat convex polygon with a ball:

```
 outline (72-gon)        inset by C              swept by a sphere of radius C
   ________                 ____                     .--------------.
  /        \               /    \                   (    rounded     )
 |          |      →      |      |        →          '--------------'
  \________/               \____/                  side view: flat top/bottom,
                                                    half-round edge everywhere
```

`insetPoly` does the shrinking by clipping the polygon against each of its own edges moved inward by `C` (Sutherland–Hodgman clipping, `clipPoly`). If the polygon becomes empty, the piece is too thin to exist — that's how slivers are rejected during cutting.

### Turning it into a mesh (`buildSlab`)

1. **Walk the inset boundary** counter-clockwise and sample `NS` points with their outward normals. At each corner the normal *sweeps* from the previous edge's normal to the next one's, so the rounded corners get samples too. Sampling is uniform in arc length of the *offset* curve (edge lengths + `C × corner angle`), and `NS = clamp(round(L × 15), 24, 96)` scales with the perimeter.
2. **Sweep a profile** at each sample. Going outward over the top, around the edge and back under the bottom:
   - `NT` top rings: lerp from the centroid to the boundary point, at height `C + dome(t)` where `dome(t) = 0.03·(1−t²)` gives a very slight pillow on top;
   - `NARC − 1 = 7` points on a half-circle: `boundary + normal·C·cos θ`, height `C·sin θ`, θ from +90° to −90°;
   - `NT` bottom rings back toward the centre at height `−C`.
3. **Two pole vertices** close the top and bottom (triangle fans).
4. **Index the grid** as quads `(a, d, c) + (a, b, d)`, wrapping around in `s`, so there's no seam and the mesh is a closed, watertight surface.

`computeVertexNormals()` is called every frame after the physics moves the vertices; since vertices are shared, the normals are smooth.

The full slice ends up with about 2,900 vertices — and therefore 2,900 particles.

---

## 4. Making it wobble: shape-matching physics

### Why shape matching?

A gummy needs to (a) spring back to its shape, (b) squash locally where you press, (c) keep its volume, and (d) never explode. Mass-spring networks tend to be either floppy or unstable; full finite-element / XPBD tetrahedral meshes are excellent but much more code. **Meshless shape matching** (Müller et al. 2005) gives (a)–(d) with a few dozen lines and is unconditionally stable.

### The time step (position-based dynamics)

`worldStep(h)` runs 6 times per 1/60 s frame (`h = 1/360 s`). For every particle:

```
1. predict    v.y += g·h              (g = −14, a bit stronger than real gravity: snappier)
              p   = x + v·h
2. constrain  move p toward the shape-matching goal     (below)
              apply grab, blade, piece-vs-piece pushes
              floor / walls with friction
3. update     v = (p − x) / h · damping       (damping = 1 − 0.5·h)
              x = p
```

Because velocity is *derived* from how far the constraints moved things, anything you do with positions — dragging, the knife wedging pieces apart — automatically turns into momentum. That's why tossing works with zero extra code.

### Shape matching for one cluster

Each cluster has a set of particles with rest positions. Precompute, once:

- rest centroid `c₀`, and rest offsets `qᵢ = restᵢ − c₀`
- `A_qq = Σ qᵢ qᵢᵀ`, and its inverse

Every substep, with current predicted positions `pᵢ`:

$$c = \frac{1}{m}\sum p_i, \qquad A_{pq} = \sum (p_i - c)\, q_i^{\mathsf T}$$

`A_pq` encodes how the cluster is currently rotated *and* stretched relative to rest. We want the best **rotation** `R` hidden inside it (its polar decomposition). We also compute the best **linear** fit:

$$A = A_{pq}\,A_{qq}^{-1}, \qquad A \leftarrow \frac{A}{\sqrt[3]{\det A}}$$

Dividing by the cube root of the determinant makes `det A = 1`: the cluster may shear and squash, but **its volume is preserved** — exactly how a gummy behaves when you press it (it bulges sideways).

Blend the two and form goals:

$$M = \beta A + (1-\beta) R, \qquad g_i = M q_i + c$$

`β` controls how much squash is allowed (0 = rigid-ish, 1 = very gooey). If `det A` is tiny or negative (an inverted cluster), we fall back to pure rotation.

### Overlapping clusters → local wobble

A single cluster over the whole slice can only deform *affinely* (the whole thing tilts/stretches at once) — it looks like a rubber coin, not jelly. So each piece gets **many overlapping clusters**:

- cluster centres on a 0.4-unit grid (in rest space) inside the outline, plus the centroid;
- each cluster takes every particle within 0.55 of its centre (top *and* bottom, so it's truly 3D);
- clusters with fewer than 16 particles are dropped (too degenerate), and any orphaned particle joins its nearest surviving cluster.

Each particle gets a goal from every cluster it belongs to; we **average** them and move the particle a fraction `α` of the way there:

$$p_i \mathrel{+}= \alpha \left( \frac{1}{k_i}\sum_{\text{clusters} \ni i} g_i - p_i \right)$$

Neighbouring clusters disagree slightly when one part is poked, and that disagreement *is* the local dent and ripple. The full slice has about 20 clusters.

### Finding the rotation robustly

Classic shape matching uses `R = A_pq (A_pqᵀ A_pq)^{-1/2}`, which needs an eigen-decomposition and misbehaves near degenerate or inverted states. Instead `extractRotation` uses Müller et al. 2016:

```
given A and a warm-start quaternion q:
repeat a few times:
    R = matrix(q)
    ω = Σ_columns (R_col × A_col)  /  |Σ_columns (R_col · A_col)|
    if |ω| ≈ 0: stop
    q = quaternion(axis = ω/|ω|, angle = |ω|) · q
```

Each cluster *keeps its quaternion* between substeps, so 3 iterations per substep is plenty — the rotation barely changes in 1/360 s. It never produces reflections and never NaNs.

### Firmness

The slider value `f ∈ [0, 1]` maps to

```
α = 0.05 + 0.45·f²      (stiffness: how hard particles pull to the goal each substep)
β = 0.45 − 0.25·f       (how much squash/shear is allowed)
```

Low `f` = slow, gooey, very squashy; high `f` = snappy gummy bear.

### Rigid best fit of the whole piece

After every substep, `Piece.fit()` runs one more shape match over *all* of a piece's particles (2 rotation iterations) to get its overall rotation `R_fit` and centre `c`. That's not used for dynamics — it's the piece's "where am I and which way is up" that collisions and cutting need to convert between world space and rest space:

```
world → rest:  r = R_fitᵀ (x − c) + c₀
rest → world:  x = R_fit (r − c₀) + c
```

---

## 5. Touching it: grab, toss, poke

**Grab (Hand tool).** On `pointerdown` we raycast all piece meshes. On a hit:

- pick every particle of that piece within 0.45 of the hit point, with weight `w = (1 − d/0.45)²` (smooth falloff), remembering each one's offset from the hit;
- create a **drag plane** through the hit point facing the camera;
- on `pointermove`, intersect the mouse ray with that plane → `target` (clamped above the board).

Each substep, grabbed particles are nudged toward `target + offset` by `0.3·w`. Shape matching drags the rest of the jelly along, so you see a stretched "neck" between your hand and the body. Let go while moving and the velocity is already there → toss.

Orbit controls and grabbing coexist by listening to `pointerdown` in the **capture** phase: if the ray hits jelly, we disable the orbit controls before they see the event.

**Poke (double-click).** Particles within 0.4 of the hit get an inward velocity impulse along the face normal (`7·w`). The clusters push back → the dent springs out with a little wobble.

---

## 6. Collisions

**Floor.** The cutting board's top is `y = 0`. Any particle below gets projected up, and its horizontal motion for that substep is scaled by `1 − μ` (`μ = 0.6`) — simple position-based friction, so pieces stick and roll instead of sliding like ice.

**Board edges.** Particles are clamped to the board's rectangle, acting as invisible walls so nothing slides off onto the counter.

**Piece vs piece.** After cutting, pieces must not pass through each other. For each pair whose bounding spheres overlap, every particle of piece A that lies inside B's bounding sphere is:

1. transformed into **B's rest frame** with B's rigid fit;
2. tested against B's *exact* shape. Since B is "flat inset polygon swept by a sphere of radius C", its signed distance is simply
   `distance(point, flat inset polygon at y = 0) − C`
   — `closestOnPoly` gives the nearest point on the polygon (or the point itself if inside), and the 3D distance from there is the answer;
3. if inside (distance < C), pushed out by half the penetration along the gradient, transformed back to world space.

Doing it in B's rest frame turns a deforming, rotating 3D shape into a 2D polygon query. It ignores B's *local* deformation, which is fine for gummies that are mostly holding their shape.

---

## 7. The knife and cutting

### The knife model

Built from primitives in `buildKnife()`:

- **blade**: a `THREE.Shape` (flat edge, curved belly rising to the tip, straight spine) extruded 0.014 units thick (the slice radius is 1) with a tiny bevel; brushed-steel `MeshPhysicalMaterial` (`metalness 1`, `roughness 0.2`, light clearcoat);
- **bolster**: a thin steel box;
- **handle**: a `CapsuleGeometry` scaled into an oval, walnut-brown; three steel rivets (cylinders).

The group's origin is the middle of the cutting edge, so placing the knife is just "edge goes here".

### Aiming

In knife mode a `pointerdown` starts a stroke on a horizontal plane at jelly-top height. While dragging, a dashed line shows the cut and the knife hovers over its midpoint, rotated along it (`rotation.y = atan2(−dz, dx)`). On release:

- stroke longer than 0.15 → cut through the stroke's midpoint along its direction;
- otherwise (a click) → chop at the cursor using the current blade angle (`Q`/`E` to turn).

### The cut animation (`updateCut`)

| time | phase |
| --- | --- |
| 0 – 0.2 s | glide from hover position to above the cut line, rotate to align |
| 0.2 – 0.8 s | descend to the board (smoothstep easing) |
| 0.8 – 0.95 s | hold at the bottom |
| 0.95 – 1.25 s | lift back up |

Each tick, `updateCut` returns a **blade collider** description (`mid`, along-blade direction `d`, normal `n`, `edgeY`, `pressing`) that `worldStep` applies to every piece.

### The blade as a collider (`applyBlade`)

For each particle, compute `along = (p − mid)·d` and `side = (p − mid)·n`. Only particles within the blade's length (`−1.45 … 1.0`) and between its edge and spine heights are affected.

- **Before breakthrough (`pressing`)** — particles within 0.07 of the blade plane are not allowed above the edge: `y = edgeY`. The jelly gets pushed *down* along a line, the clusters resist, and you see a groove form and the slice squash — the "it doesn't cut instantly" moment that sells the material.
- **When the edge reaches ~1.1·C above the board** (roughly half-way through the thickness), we perform the actual cut (next section) and switch to:
- **Wedge mode** — the blade is treated as a wedge, thin at the edge and thicker higher up: `half = 0.012 + 0.09·min(1, height/0.34)`. Particles inside are pushed sideways to `±half`. As the blade descends, this shoves the two halves apart — and because PBD derives velocity from movement, they keep drifting a little after the knife leaves.

### Splitting a piece (`splitPiece`)

1. **Is the blade over it?** At least 6 particles must lie within the blade's length and 0.12 of its plane.
2. **Plane into rest space.** The knife plane (point `mid`, horizontal normal `n`) is transformed into the piece's rest frame with its rigid fit. Intersecting it with the slab's mid-plane `y = 0` gives a 2D line `n'·(x, z) = d'`. If the piece is standing on its edge (the line's normal is mostly vertical in rest space, `|n'ₓ_z| < 0.35`), it's skipped.
3. **Clip.** `clipPoly(outline, n', d' + GAP)` and `clipPoly(outline, −n', −d' + GAP)` give the two convex halves, with a gap of 0.012 units (the slice radius is 1) so they don't start overlapping. Both must have area > 0.05 and a non-empty inset (area > 0.004) — otherwise the cut would create a sliver and is cancelled for that piece.
4. **Build two new `Piece`s** from the halves. They get their own mesh, particles and clusters — and because UVs are in slice space, the texture continues seamlessly.
5. **Transfer state.** The new pieces start at rest pose, but the old piece was squashed by the knife and possibly moving. For each new particle at rest position `r`:
   - find the nearest *old* particle `j` in rest space (both share the same slice frame!) — using a hash grid of 0.1-unit cells searched in growing rings, which made this ~2× faster than brute force;
   - set `x_new = x_old[j] + R_fit·(r − rest_old[j])` and `v_new = v_old[j] ± 0.7·n`.

   The new pieces therefore inherit the exact dent, rotation and momentum, and get a small push away from the blade. The cluster quaternions are seeded with the old piece's rotation so the solver doesn't have to "find" it.
6. The old mesh is removed and disposed; the new ones are added. The number of pieces is capped at 14.

Pieces can be cut again and again: a piece's `outline` is just a convex polygon, and convex ∩ half-plane is always convex.

---

## 8. The kitchen

No image files are used — every surface is drawn at startup:

| Object | How |
| --- | --- |
| **Cutting board** | rounded-rectangle `THREE.Shape` with an elliptical handle hole, extruded and bevelled; texture = 8 butcher-block strips of slightly different wood tones + 260 wavy grain lines + dark glue lines. Its top sits exactly at `y = 0`, the physics floor |
| **Marble counter** | big box; texture = pale base + 14 random-walk "veins" drawn with `ctx.filter = 'blur(1.5px)'`; clearcoated |
| **Backsplash** | plane with offset subway tiles (rounded rects with a soft vertical gradient each, sage tint) on grout; glossy clearcoat |
| **Fruit** | spheres squashed a little (`scale.y = 0.94`, lemons stretched along x), a dimple bump texture, a small stem nub |
| **Bowl** | `LatheGeometry` from an 11-point profile, teal glazed ceramic, double-sided |
| **Glass jar** | `LatheGeometry` with transmission (real refraction of what's behind), a cork cylinder lid, 26 small translucent gummies inside |
| **Basil** | tapered terracotta cylinder + torus rim + soil disc + 46 flattened-ellipsoid leaves with sheen |
| **Tea towel** | thin box with a red gingham canvas texture |

Lighting: a hemisphere light (warm sky / wood-coloured ground bounce), a directional "window" light from the upper left with soft PCF shadows covering the whole counter, ACES filmic tone mapping, and light fog to soften the far wall.

The rich background is not just decoration — the jelly is transparent, so **what's behind it is half of what you see**. A plain floor made the jelly look flat; wood grain and tiles give the refraction something to bend.

---

## 9. Performance

Measured on a laptop in Chromium:

| Situation | Physics cost per frame |
| --- | --- |
| one whole slice (~2,900 particles, ~20 clusters) | ≈ 4.5–7 ms |
| four pieces after two cuts | ≈ 5 ms |
| the frame where a cut happens | ≈ 30–50 ms one-off (building meshes, clusters, state transfer) |

Things that keep it fast:

- **One particle per vertex** — no separate sim mesh, no skinning.
- **Typed arrays + hand-unrolled 3×3 maths** in the hot loops; no `Vector3` allocations per particle.
- **Fixed timestep with an accumulator**, max 3 ticks per frame, so a slow frame can't spiral.
- **Resolution where it shows**: fine detail lives in the 1024² textures, not in the mesh. Going from 5,200 to 2,900 particles cut physics time by ~3× with no visible loss.
- **Warm-started rotations** — 3 iterations per cluster per substep are enough.
- **Bounding-sphere early-outs** for piece-vs-piece contact.

---

## 10. Tuning cheat-sheet

| Constant | Where | Effect |
| --- | --- | --- |
| `H = 0.34`, `C = H/2` | slab | thickness / edge roundness |
| `NS`, `NT`, `NARC` | `buildSlab` | mesh resolution (= particle count) |
| `dome` 0.03 | `buildSlab` | puffiness of the top |
| `SUB = 6` | loop | substeps per 1/60 s; more = stiffer & more stable, slower |
| gravity −14 | `predict` | heavier / snappier drops |
| damping `1 − 0.5h` | `finalize` | how long wobbles last |
| `μ = 0.6` | `finalize` | floor friction |
| cluster grid 0.4, radius 0.55 | `buildClusters` | smaller = more local wobble, more cost |
| `α`, `β` mapping | loop | firmness feel |
| grab radius 0.45, pull 0.3 | pointer / `worldStep` | how big a handful you grab |
| blade wedge `0.012 + 0.09·…` | `applyBlade` | how far halves are pushed apart |
| split height `1.1·C` | `updateCut` | how much it squashes before breaking |
| `GAP = 0.012` | cutting | spacing between new halves |
| separation speed 0.7 | `splitPiece` | how fast halves drift apart |
| `MAX_PIECES = 14` | cutting | cap on pieces |

The page exposes `window.citrusJelly` (pieces, `newSlice`, `setFruit`, `setTool`, `startCut`, …) for experimenting from the browser console.

---

## 11. Ideas for extending it

- **New fruit** — add an entry to `FRUITS` (peel, pith, flesh, highlight, attenuation colours). For a kiwi or watermelon, write a new painter like `paintCitrus`; the planar UVs work for any flat slice.
- **New shapes** — anything that can be described as a convex outline works immediately (wedges, squares, hexagons). Non-convex shapes (a jelly crab!) would need either a decomposition into convex parts or switching to a tetrahedral mesh.
- **Juice** — spawn small instanced droplets along the cut line at breakthrough, with simple ballistic motion and a fading wet decal on the board.
- **Subsurface glow** — a custom shader pass that measures real thickness (front-face depth − back-face depth) would let thin edges glow brighter than the centre.
- **Higher-end physics** — the original jelly videos that inspired this use XPBD on tetrahedral meshes with co-rotational shape matching per tetrahedron and volume constraints; that handles arbitrary shapes and cuts through any plane, at the cost of a tet-mesher and more code.
- **Sound** — a soft "squelch" whose pitch follows compression, and a "thunk" when the blade hits the board.

---

## References

- M. Müller, B. Heidelberger, M. Teschner, M. Gross. **Meshless Deformations Based on Shape Matching.** SIGGRAPH 2005. — the core physics.
- M. Müller, J. Bender, N. Chentanez, M. Macklin. **A Robust Method to Extract the Rotational Part of Deformations.** MIG 2016. — `extractRotation`.
- M. Müller, B. Heidelberger, M. Hennix, J. Ratcliff. **Position Based Dynamics.** 2007. — the predict / constrain / update loop.
- M. Macklin, M. Müller, N. Chentanez. **XPBD: Position-Based Simulation of Compliant Constrained Dynamics.** 2016. — the next step up.
- I. Sutherland, G. Hodgman. **Reentrant Polygon Clipping.** 1974. — `clipPoly`.
- [three.js `MeshPhysicalMaterial` docs](https://threejs.org/docs/#api/en/materials/MeshPhysicalMaterial) — transmission, attenuation, clearcoat.
