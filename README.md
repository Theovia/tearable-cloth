# tearable-cloth

Vanilla canvas 2D Verlet cloth that **feels like plastic** — grab, pull, stretch, tear.

Single HTML file. Zero dependencies. ~280 lines.

**Live demo:** https://theovia.github.io/tearable-cloth/

Inspired by [pushmatrix/tearable](https://pushmatrix.github.io/tearable/) (which uses R3F + Three.js + WebGL). This is the lean version: same feel, no bundle.

---

## What makes it feel like plastic, not cardboard

Most Verlet cloth demos look stiff and synthetic. The fixes:

| Trap | Fix |
|---|---|
| Mouse cuts constraints directly | Mouse **grabs** nearby particles + pulls them with offset. Tearing is a *consequence* of strain, not the input. |
| Stiffness 0.5, 4 iters → rigid | Stiffness **0.28**, **10 iters** → rubbery |
| Tear by absolute distance | Tear by **ratio** of current/rest length (`> 5.5x` snaps). Local strain, scale-independent. |
| Only structural springs | + **Bend constraints** (skip-1 neighbor) → memory, page-like resistance |
| No visible tension | **Whiten triangles by strain ratio**. Compressed cells get a soft shadow. The eye reads the stress. |
| Hard pin pop on release | Soft follow (lerp 0.45) + verlet history fix → no snap-back |

## The physics in 60 seconds

**Verlet integration** stores `(pos, prevPos)` instead of `(pos, vel)`. Velocity is implicit:

```js
const vx = (this.x - this.px) * (1 - DAMPING);
this.px = this.x;
this.x += vx + GRAVITY * dt * dt;
```

**Constraints** are springs between neighbors. Each iteration nudges the two endpoints toward rest length:

```js
const dx = b.x - a.x, dy = b.y - a.y;
const d = Math.hypot(dx, dy);
const ratio = d / restLength;
if (ratio > TEAR_RATIO) { this.dead = true; return; }
const diff = (d - restLength) / d * STIFFNESS;
a.x += dx * diff; a.y += dy * diff;
b.x -= dx * diff; b.y -= dy * diff;
```

Run that 10× per frame across all constraints → cloth converges to a stable shape.

**Tear** = remove constraint from solve loop when `ratio > 5.5`. Hole opens automatically because the renderer skips quads with any dead edge.

## Texture warp on canvas 2D

The page art is rendered once to an offscreen canvas (720×960). Each grid cell becomes 2 triangles. To map a triangle of the source texture to the deformed triangle on screen, solve a 2×2 affine + translation:

```js
ctx.save();
ctx.beginPath();
ctx.moveTo(d0x, d0y); ctx.lineTo(d1x, d1y); ctx.lineTo(d2x, d2y);
ctx.closePath(); ctx.clip();

// solve dest = M * src + t
const det = sx1*sy2 - sx2*sy1;
const a = (dx1*sy2 - dx2*sy1) / det;
const b = (dy1*sy2 - dy2*sy1) / det;
const c = (dx2*sx1 - dx1*sx2) / det;
const d = (dy2*sx1 - dy1*sx2) / det;
const e = d0x - a*s0x - c*s0y;
const f = d0y - b*s0x - d*s0y;
ctx.transform(a, b, c, d, e, f);
ctx.drawImage(page, 0, 0);
ctx.restore();
```

This is the canvas-2D equivalent of UV-mapped triangles. WebGL would do this in a vertex shader — here we do it per-tri on CPU. Fast enough for 50×70 grid at 60fps on a laptop.

## Strain shading (the real trick)

After drawing the textured triangle, overlay a translucent fill whose alpha tracks the cell's average strain:

```js
if (strain > 1.05) {
  const t = Math.min(1, (strain - 1.0) / (TEAR_RATIO - 1.0));
  ctx.fillStyle = `rgba(255,255,255,${t * 0.7})`;
  // fill triangle
}
```

Stretched → whitening. Compressed → soft shadow. This single layer is what sells the plastic feel — without it, the same physics looks geometric.

## Params worth tuning

```js
const COLS = 50;
const ROWS = 70;
const GRAVITY = 380;
const DAMPING = 0.018;
const ITERATIONS = 10;
const STIFFNESS = 0.28;
const BEND_STIFFNESS = 0.06;
const TEAR_RATIO = 5.5;
const FADE_RATIO = 2.4;
const GRAB_RADIUS = 38;
const GRAB_SOFTNESS = 0.45;
```

- Tougher material (paper): stiffness 0.5, tear 2.5
- Softer (silk): stiffness 0.15, tear 8
- Latex: bend 0.0, stiffness 0.2, tear 6, gravity 200

## Future ideas

- **Multi-page stack** — array of Cloth instances, only top is active, lower pages reveal as top tears
- **Self-collision** — broadphase grid, keep particles from passing through each other
- **Audio** — Web Audio noise burst when dead-constraint count delta spikes
- **Cursor mass** — cursor itself becomes a heavy particle, drag has momentum
- **R3F port** — same physics, `<planeGeometry>` + `useFrame` + vertex buffer update; lets pages be real HTML→texture
- **Normal-map shading** — derive a fake normal from neighbor positions, light it → 3D feel without 3D

## Run locally

```bash
python3 -m http.server 4444
# open http://localhost:4444/
```

That's it. No build, no bundler, no deps.

## License

MIT.
