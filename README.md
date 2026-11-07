# Crystal Moon V2

The geometry starts acting on itself.

[CrystalMoon](https://github.com/anttiluode/CrystalMoon) was one Mexican-hat crystal per world, a lens that bends light toward dense crystal, a clock that slows it, and a moon that is a sphere inversion with another world inside. V2 keeps all of that and adds four ways for the world to work on itself:

1. **Two scales.** A coarse crystal grows the bodies. A fine lattice can only live inside them, so every body is made of smaller crystal, and the fine crystal feeds back on where the body may grow.
2. **The moon is the light, and the light writes.** 16 384 photons leave the moon in a steady stream of traces and travel through the same lens the view rays use, so crystal bends them toward itself. Inside the islands, lit crystal grows and dark crystal melts. Crystal redirects the light, the redirected light writes crystal, and that crystal redirects the light again: the loop of a self-written waveguide. The same light shades the crystal and hangs in the air as shafts.
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

**Moonlight: photons through the lens.** One trace at a time, spread over 3 frames (16 steps a frame), alternating outer, outer, inner:

```
emit     K = 16 384 photons from the moon's surface, directions on a Fibonacci sphere under a random rotation
step     d ← normalise(d + h (∇n − (∇n·d) d) / n),   n = 1 + κρ      (the view rays' own ray equation)
         x ← x + h d,      w ← w · exp(−0.12 · occ(x) · h),   occ = smoothstep(0.6·iso, 1.2·iso, Ψ)
deposit  w·h into the voxel at x                                      (additive blending, 48 steps, h = 0.8)
light    fluence · 4π(R+2)²/K, blurred once, capped at 3, averaged with the old light (weight 0.3)
```

With no crystal this is the free 1/r² falloff, normalised to 1 at two voxels above the moon. Crystal is glass here: it absorbs weakly (σ = 0.12, so it still casts shadows) and bends strongly, so a body that faces the moon can gather light and carry it outward. Light enters growth only inside the islands, so the open sea is never lit into crystal; it sculpts the islands.

The V1-style straight light is still there (uncheck *moonlight bends through the crystal*): the transmittance exp(−0.7 ∫ occ ds) along the straight line to the moon, times 1/(1 + ((|x − c| − R)/18)²). Switching between the two is the A/B for what the bending does.

The inner world is lit by its own moon at its centre. Through the portal its centre is infinity, which is out past the membrane on our side, so from inside the moon the inner crystals are lit from the direction of the world you came from.

**Unchanged from V1:** the lens n = 1 + κρ (ρ now from Ψ²), the portal Φ(x) = c − R²(x − c)/|x − c|², the half-turn on crossing, one law per level from a hash of the level number, the wake you leave, and everything the camera does (it now gates both fields, so the picture's crystal is made of the fine lattice).

## Ledger

`node tests/core.test.cjs` checks the CPU reference in `core.js`:

(12/12 pass, about 6 minutes)

| test | result |
|---|---|
| T1 one law per level, same every visit; explicit step inside the stability bound for both fields | worst dt·\|op\| over 60 levels = 1.50 < 2 |
| T2 strongest growth, feedback and light: bounded, no NaN | max \|ψ\|, \|φ\| = 1.81 |
| T3 the fine lattice grows inside the coarse body and nowhere else | in 55% of voxels where ψ > 0.4, 1% where ψ < 0, 0% in the far sea |
| T4 each Mexican hat picks its own wavelength | ψ peaks at \|m\| = n/λc = 4, φ at n/λf = 8, exactly |
| T5 the lattice feeds back on the body | rms body amplitude 0.633 / 0.735 / 0.839 at k_fb = −0.4 / 0 / +0.4 |
| T6 straight moonlight: a crystal ball casts a shadow | light behind 0.004, beside 0.53, between ball and moon 0.99 |
| T7 light is a growth term: same ε (−0.2), lit half crystallises, shadowed half stays empty | 42% vs 0% |
| T8 portal Φ: involution, \|Φx−c\|·\|x−c\| = R², conformal, half-turn on the moon; Φ∘I = 2c − x, exact on voxels | all errors < 2e-13; voxel mirror exact |
| T9 the membrane: an island touching the moon grows its continuation in the inner world at 2c − x, and only there | inner \|ψ\| > 0.1 at the mirror site: 37% glued, 0% unglued; mean \|Ψ_in − Ψ_out(2c−x)\| at the membrane 0.033 |
| T10 camera brightness gates both fields | crystal on the bright half 97%, dark half 0%; fine lattice in the bright half 70% |
| T11 motion drives | 100% crystal in the moving patch, 0% elsewhere |
| T12 photons: a crystal rod that casts a shadow in straight light becomes a light guide when light bends (κ = 0.8) | light inside the rod 24 voxels out: straight 0.058, bent 3.0 (the cap); 5 voxels beside it: 0.25 → 0.10 |

T12 is the waveguide in its simplest form: the same rod, the same light, and the only change is whether the light bends. Bending turns its shadow into the brightest place in the world, and takes the light from beside it.

**The GPU matches the CPU.** Open `index.html?selftest` and the page steps both fields on the GPU and in `core.js` from the same start, for 25 steps, with the light term, the membrane, the nucleation noise and the camera all on. The noise is an integer hash written identically in JS and GLSL, so it is part of the comparison, not switched off for it.

| run (25 steps, light = k/16 pattern) | largest \|GPU − CPU\| | largest \|field\| |
|---|---|---|
| both fields, no membrane | 4.8e-8 | 0.12 |
| membrane on (other world uploaded too) | 5.0e-8 | 0.12 |
| camera picture on the sheet | 6.7e-8 | 0.12 |
| camera picture on the moon's shell | 3.4e-8 | 0.12 |

Found while checking: with the nucleation noise switched on only where ε > 0, the comparison failed at 1e-3 in rare voxels where ε was zero to within rounding and the two sides disagreed about its sign. The noise is now gated by smoothstep(0, 0.02, ε), which is continuous, and the difference is back at rounding level.

(headless Chrome, SwiftShader.)

## What this is not (yet)

- **Speed.** The first version ran at 20 fps on Antti's RTX 3060 — and the counter could not show less than 20, because it divided by the physics time step, which is clamped at 0.05 s. It now uses the real frame time. Since then: the view fields are 64³ 3D textures with hardware trilinear filtering and REPEAT wrap (one texture fetch per sample instead of two, and no gutter), the march takes longer strides with distance and stops at 125 (fog hides both), the islands' ε is written once per level, the photon budget is 16 384 spread over three frames, the glass ray is shorter and skipped where it would barely show, and *hold the frame rate* lowers the render scale under 40 fps. In headless SwiftShader one 320×180 frame went from 4.9 s to 2.2 s. The fps on the 3060 is not measured yet.
- **The photons are not waves.** They bend and concentrate, so light can be channelled by crystal, but there is no diffraction or interference. This is ray optics; it cannot show a spatial soliton. The photon tracer is checked on the CPU (T12); the GPU version mirrors it but reads ρ from half-float textures and is not compared number for number.
- **No light crosses the membrane.** Each world is lit by its own moon.
- **The moon lights only its own cell.** The world repeats every 64 units; photons stop 46 units from the moon, and the renderer fades the light out beyond ~40 units so the periodic copies are not lit by a moon that is not there.
- **The bodies can still fuse.** With the lattice feeding the body (k_fb > 0) and strong light, ε inside lit bodies roughly doubles and the bodies swell into smooth blobs. Level 0 now has k_fb = −0.1 (the lattice eats the body, which leaves it porous); deeper levels draw k_fb from −0.2 to +0.06.
- **The two clocks are separate.** One shared clock 1/(1 + T(ψ² + φ²)) was the first version; inside dense bodies it slowed the fine lattice about 3.5×, and it took ~3× longer to appear. Each field now runs on its own density.
- **The lattice appears after the bodies.** Bodies condense in ~1500 steps (about 2 s at default speed), the lattice inside them in ~3500.
- **Two scales on a 64³ grid are not far apart.** λc/λf is about 2. A body is a few lattice cells thick. More separation needs a finer grid.
- **The membrane is a coupling I chose,** not something derived. The identity Φ∘I = 2c − x is exact; the rule that the two fields relax toward each other across it is a modelling decision, and so is sharing ε.
- **The descent is still one way,** and the moon's inner world is still a separate simulation viewed through the inversion.

## Files

- `index.html`: the world (both fields on the GPU, photon and straight moonlight, membrane, density lens, curved-ray raymarcher with the glass ray, moon portal, flight, camera)
- `core.js`: the laws, the CPU reference for both fields, both kinds of light, the portal and membrane maths, the camera placement. It is the spec the shaders mirror
- `tests/core.test.cjs`: the ledger

Built with Claude (Opus 5.5), Saturday 7 November 2026, from CrystalMoon and a talk with Astra about what it was begging for: two growth scales, the moon folding the growth and not only the view, light that leaves a structural memory, and glass you can see through.
