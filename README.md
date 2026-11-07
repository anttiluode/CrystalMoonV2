# Crystal Moon V2

The geometry starts acting on itself.

[CrystalMoon](https://github.com/anttiluode/CrystalMoon) was one Mexican-hat crystal per world, a lens that bends light toward dense crystal, a clock that slows it, and a moon that is a sphere inversion with another world inside. V2 keeps all of that and adds four ways for the world to work on itself:

1. **Two scales.** A coarse crystal grows the bodies. A fine lattice can only live inside them, so every body is made of smaller crystal, and the fine crystal feeds back on where the body may grow.
2. **The moon is the light, and the light writes.** Crystal casts shadows away from the moon. Inside the islands, lit crystal grows and shadowed crystal melts, so the crystals compete for the moon. The same light shades the crystal and hangs in the air as shafts.
3. **The moon is a membrane.** The world outside and the world inside are glued across it, so crystal that touches the moon grows through. Inside, the inversion bends what came through into arcs that shrink toward the core.
4. **Glass shows what is really behind it.** A refracted ray crosses each crystal, leaves it, and finds the next crystal (or the moon, or your face in the sky) behind it.

One HTML page, WebGL2, no libraries. **Live:** https://anttiluode.github.io/CrystalMoonV2/

**Controls:** W A S D fly · R / F up and down · Q / E roll · Shift fast · mouse look (click to capture) · click seeds a crystal · M faces the moon · V camera · H hides the panels. Fly into the moon to go a level deeper.

## The membrane, and the one identity it rests on

When a ray crosses the moon's surface at a point x outside, the renderer continues it in *moon space* at I(x), the ordinary inversion through the sphere (same direction from the centre, radius R²/|x − c|). At a moon-space point q it draws the inner world at Φ(q), the antipodal inversion from V1. So the outer world at x touches the inner world at Φ(I(x)), and

```
Φ(I(x)) = c − R² · (R²(x−c)/|x−c|²) / (R⁴/|x−c|²) = 2c − x.
```

The composition of the two inversions is just the point reflection through the moon's centre. With the centre at voxel (32, 32, 32) it is exact on the lattice: voxel k is glued to voxel (64 − k) mod 64, no interpolation.

So the gluing is cheap and exact. Near the moon (inside a band that fades out by R + 3.5):

```
ε_here   ← ε_here + b · (max(ε_here, ε_there(2c − x)) − ε_here)      the worlds share where crystal may live
∂ψ/∂t   += γ b · (ψ_there(2c − x) − ψ_here(x))                       and are pulled together (same for φ)
```

with γ = 2 · membrane. The two worlds have different laws (wavelengths, nonlinearity), so where they meet they disagree. The seam is where the defects are.

## The laws

```
∂ψ/∂t = Mψ · [ εc ψ − (qc² + ∇²)² ψ + gc ψ² − ψ³ ] + membrane          the bodies (λc ≈ 8–12)
∂φ/∂t = Mφ · [ εf φ − (qf² + ∇²)² φ + gf φ² − φ³ ] + membrane          the lattice (λf ≈ 4.2–5.4)

εc = ε_islands + trail + k_fb φ² + k_L (light − 0.25) · island
εf = ε_f,sea + (ε_f − ε_f,sea) · smoothstep(−0.2, 0.3, ψ)
Mψ = 1/(1 + Tψ²),   Mφ = 1/(1 + Tφ²)                                   (each lattice on its own clock)
what you see:  Ψ = ψ + βφ
```

- **The lattice lives in the body.** εf is negative everywhere except where ψ is positive, so φ only crystallises inside coarse crystal.
- **The lattice feeds back.** k_fb φ² adds to εc. With k_fb > 0 the fine crystal thickens the body; with k_fb < 0 it eats it. Each level draws its own k_fb.
- **The surface you see is the sum.** Far away a body looks smooth. Close up its surface is the fine lattice poking through.
- **q² is set for the lattice, not the continuum.** The 7-point Laplacian has symbol 2 − 2cos k along an axis, not k². At λ ≈ 9 the difference is negligible; at λ ≈ 4.5 it moves the wavelength by 10%. V2 sets q² = 2 − 2cos(2π/λ) so the spectrum peaks where the law says (T4).

**Moonlight.**
```
light(x) = exp(−0.7 ∫ occ ds) / (1 + ((|x − c| − R)/18)²),     occ = smoothstep(0.6·iso, 1.2·iso, Ψ)
```
The integral runs along the straight line from x to the moon (24 samples), through the crystal you can see. It is recomputed on the GPU every 3 frames (every 6 for the inner world). Light enters growth only inside the islands, so the open sea is never lit into crystal; it sculpts the islands instead.

The inner world is lit by its own moon at its centre. Through the portal its centre is infinity, which is out past the membrane on our side, so from inside the moon the inner crystals are lit from the direction of the world you came from.

**Unchanged from V1:** the lens n = 1 + κρ (ρ now from Ψ²), the portal Φ(x) = c − R²(x − c)/|x − c|², the half-turn on crossing, one law per level from a hash of the level number, the wake you leave, and everything the camera does (it now gates both fields, so the picture's crystal is made of the fine lattice).

## Ledger

`node tests/core.test.cjs` checks the CPU reference in `core.js`:

LEDGER_TABLE

**The GPU matches the CPU.** Open `index.html?selftest` and the page steps both fields on the GPU and in `core.js` from the same start, for 25 steps, with the light term, the membrane, the nucleation noise and the camera all on. The noise is an integer hash written identically in JS and GLSL, so it is part of the comparison, not switched off for it.

SELFTEST_TABLE

(headless Chrome, SwiftShader.)

## What this is not (yet)

- **Speed on a real GPU is not measured.** It runs in headless SwiftShader. The new costs are a light pass every 3 frames, one more texture read per sim step, and the glass ray, which can add up to ~75 samples on pixels that hit crystal. If it is slow, lower *glass* to 0 first (that falls back to V1's glass), then render scale.
- **The light travels straight.** The view rays bend through the lens; the light from the moon does not. It also casts no light through the membrane: each world is lit by its own moon.
- **The moon lights only its own cell.** The world repeats every 64 units; the light is faded out beyond ~40 units from the one real moon so the copies are not lit by a moon that is not there.
- **The two clocks are separate.** One shared clock 1/(1 + T(ψ² + φ²)) was the first version; inside dense bodies it slowed the fine lattice about 3.5×, and it took ~3× longer to appear. Each field now runs on its own density.
- **The lattice appears after the bodies.** Bodies condense in ~1500 steps (about 2 s at default speed), the lattice inside them in ~3500.
- **Two scales on a 64³ grid are not far apart.** λc/λf is about 2. A body is a few lattice cells thick. More separation needs a finer grid.
- **The membrane is a coupling I chose,** not something derived. The identity Φ∘I = 2c − x is exact; the rule that the two fields relax toward each other across it is a modelling decision, and so is sharing ε.
- **The descent is still one way,** and the moon's inner world is still a separate simulation viewed through the inversion.

## Files

- `index.html`: the world (both fields on the GPU, moonlight pass, membrane, density lens, curved-ray raymarcher with the glass ray, moon portal, flight, camera)
- `core.js`: the laws, the CPU reference for both fields, the light, the portal and membrane maths, the camera placement. It is the spec the shaders mirror
- `tests/core.test.cjs`: the ledger

Built with Claude (Opus 5.5), Saturday 7 November 2026, from CrystalMoon and a talk with Astra about what it was begging for: two growth scales, the moon folding the growth and not only the view, light that leaves a structural memory, and glass you can see through.
