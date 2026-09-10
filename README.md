# Euro-LFO-2

Claude Conversation about the LFO Math

Here's the whole thing in one place, cleaned up.

## The circuit

```
        R6 10K        SHAPE 500K        D1/D2         RATE 100K
U1B(-) ──/\/\──── wiper ──/\/\──┬──── steering ────── wiper
  │                             │      diodes           │  pin1 = square (U1A out)
 C1 47n (to U1B out)                                     │  pin3 = GND via R4 4K7
                                                    
U1B: integrator   (+in = GND, C1 from out→in, out = triangle)
U1A: Schmitt trig  (-in = GND, +in from R1 150K [own out] and R5 100K [triangle])
     out = square, ±V_sat
```

## Step 1 — Schmitt thresholds set the triangle amplitude

U1A's + input, no current into the pin:

```
V+ = (V_tri·R1 + V_sq·R5) / (R1 + R5)
```

Trips when `V+ = 0`:

```
V_tri,trip = −V_sq · (R5/R1)
```

So the triangle bounces between `±V_sat·(R5/R1)`, giving a peak-to-peak swing:

```
ΔV_pp = 2·V_sat·(R5/R1) = 2·V_sat·(100K/150K) = 1.33·V_sat  ≈ 14 V pp  (±7 V)
```

## Step 2 — Integrator ramp rate

Summing node is a virtual ground, so the timing current is set entirely by the drive voltage and the series resistance:

```
slope = |dV_tri/dt| = V_drive / (R_t · C1)
```

`V_drive` = RATE wiper voltage minus one diode drop; `R_t` = series resistance for that half-cycle.

## Step 3 — Period

Each ramp traverses the **full** pp swing, and the two halves use the two halves of SHAPE (`R_a + R_b = 500K`) with R6 and the RATE source resistance `R_th` in both:

```
T = t_up + t_down
  = 2·(R5/R1)·(V_sat/V_drive)·C1·[ (R6 + R_a + R_th) + (R6 + R_b + R_th) ]
```

```
┌────────────────────────────────────────────────────────────────┐
│  T = 2·(R5/R1)·C1·( 2·R6 + R_SHAPE + 2·R_th ) · V_sat/(V_th−V_f) │
└────────────────────────────────────────────────────────────────┘
```

Key point: **R6 and R_th count twice** (in the path on both ramps); **SHAPE counts once** (the wiper splits it, halves sum to 500 K). That's why SHAPE changes symmetry/PWM but not frequency.

## Idealized form (RATE max, ignore diode)

`V_th → V_sat`, `R_th → 0`, and **V_sat cancels**:

```
        R1                    1
f  ≈  ────── · ──────────────────────────────
        R5       2 · C1 · (2·R6 + R_SHAPE)
```

## Numbers

```
R1/R5 = 1.5
2·C1·(2·R6 + R_SHAPE) = 2·(47 nF)·(20K + 500K) = 0.0489

f_max ≈ 1.5 / 0.0489 ≈ 31 Hz     (T ≈ 33 ms)
```

As RATE turns down, `V_th` falls and stretches the period:

| RATE wiper | V_th | f (approx) |
|---|---|---|
| full up | V_sat | ~30 Hz |
| middle | ~0.5·V_sat | ~14 Hz |
| near bottom | ~0.045·V_sat ≈ 0.47 V | crawls / stalls (below the ~0.5 V diode drop) |

**Practical range ≈ 0.3 Hz to ~30 Hz.**

## Notes

- **V_sat** ≈ ±10.5 V (TL074 stops ~1.5 V short of the rails). It drops out of the ideal formula — bigger swing = taller triangle *and* faster ramp, cancels.
- **V_f** ≈ 0.4–0.5 V steering-diode drop. It does *not* scale with V_sat, so it's what makes the low end nonlinear and adds mild temp drift (−2 mV/°C).
- **To lower the whole range**, scale **C1** up (470 n → ~0.1–3 Hz; 1 µF → ~1.4 Hz max). Don't retune with R5/R1 — that moves amplitude and frequency together.
- **C1 dielectric:** X7R drifts a few % — fine for an LFO. C0G only if you want the rate rock-stable.
- Comparator propagation delay (~1–2 µs) vs a 33 ms period → negligible.




Close, but the RATE knob isn't a series resistance you can add — it's a **voltage divider** on the drive (square wave → wiper → GND through R4). So it *multiplies* the rate rather than adding to the timing resistance.

## Ballpark model

**Fast end (RATE maxed)** — the idealized formula as-is:

```
        R1                1                1.5
f_max ≈ ── · ───────────────────────── ≈ ───────────── ≈ 30 Hz
        R5    2·C1·(2·R6 + R_SHAPE)      2·47n·520K
```

**Anywhere else** — scale by the knob:

```
f ≈ f_max · (V_th / V_sat) ≈ f_max · (fraction of knob rotation)
```

roughly linear for a linear-taper pot.

**Slow end (RATE at minimum)** — the drive collapses to the R4 divider:

```
                R4              4.7K
f_min ≈ f_max · ────────── ≈ 30·────── ≈ 1.4 Hz
                R_RATE+R4        104.7K
```

and the ~0.45 V diode drop eats the last sliver, so it actually **stalls at the end stop** — call the usable floor ~1 Hz.

## So

**~1 Hz to ~30 Hz, ~15 Hz at noon.**

## Why "+100K after R_SHAPE" doesn't give the min

The RATE pot's *resistance* contribution (its Thévenin source R looking back into the wiper) only swings from 0 to about **25K** — parallel combination of its two halves — and even at max that changes `f` by ~5%. The 20:1 rate span is entirely the *voltage* division, which is a multiplier, not a term you add to the ohms.

If you want the classic slow-LFO window, scale **C1**: 470 n → ~0.1–3 Hz, 1 µF → ~0.05–1.4 Hz.