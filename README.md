# OptiScaler DLSS-NR — multi-pass with halo controls (THERMOTRON / Blue Rattler fork)

> A fork of [OptiScaler_DLSSNR](https://github.com/Dagherbou/OptiScaler_DLSSNR), which is itself a fork of [OptiScaler](https://github.com/optiscaler/OptiScaler). The upstream project's own README is kept here as [README_OptiScaler.md](README_OptiScaler.md).

It runs NVIDIA's DLSS 5
Neural Rendering model **more than once per frame** — up to 4 real passes, each on its own model
feature — and adds the controls we needed to keep the picture clean when doing so:

- **Passes 1–4.** Each pass is a separate NGX feature fed the previous pass's answer (no shared
  history, so no smear), built one frame ahead so a pass-count change never stalls the frame.
- **Per-pass strength** (`PassDecay2/3/4`): the later passes run with Intensity and Local structure
  scaled down. The silhouette glow ("halo") is border contrast the model pulls, and N passes pull it
  N times; weaker later passes pull less of it while the model still sees the whole frame.
- **Per-pass resolution** (`PassScale2/3/4`): each later pass at its own fraction of the working
  size. It is shown the current picture shrunk, and what it adds comes back as a residual over the
  full-size picture — pass 1's fine texture is kept whole. Go **down** the ladder (100 / 85 / 70),
  never up. What this costs you is measured below; it is not free.
- **Cost presets** in the menu: Performance / Balanced / Quality / Photo, set from the measurements
  in this README. They touch passes, per-pass resolution and per-pass strength only — style,
  intensity, skin and tone are left exactly as you set them.
- **Edge guard** (`EdgeGuardMode`): depth-aware silhouette band (soften / no brightening / luma
  lock) and the "detail only" modes, which divide the model's large-scale brightness change back
  out so only texture stays. Debug view shows the band.
- Everything above works with the fork's existing reversible / replace modes, model resolution,
  supersampling and exposure tools.

Tested on RE Engine (Onimusha: Way of the Sword), REDkit (The Blood of Dawnwalker) and Creation
Engine 2 (Starfield), RTX 5090 at 4K, driver 616.64. Videos: [Blue Rattler on YouTube](https://www.youtube.com/@BlueRattler).

> **You must supply `nvngx_dlssnr.dll` yourself** (NVIDIA's model, ~165 MB). It is not, and will
> never be, in this repository or its releases. Everything here is undocumented and unsupported by
> NVIDIA; use at your own risk.

## Install

1. Build (Visual Studio 2022, Release x64) or take `OptiScaler.dll` from a release, rename it
   `dxgi.dll` and put it next to the game's executable, together with `nvngx.dll_dlssnr.dll`
   (the forwarder, in the release) and your own `nvngx_dlssnr.dll`.
2. Copy the matching `OptiScaler.ini` from `presets/` (one per game) next to it.
3. In game, open the OptiScaler menu (Insert) → *DLSS Neural Rendering*. Everything below is live.

## Start here: the cost presets

Four buttons at the top of the *Cost* section. They apply immediately and the active one is
highlighted; move any slider and the highlight clears.

| Preset | Passes | Resolution | Strength | ms (5090, 4K) |
|---|---|---|---|---|
| **Performance** | 2 | 100 / 100 | 1.00 | **~14.0** |
| **Balanced** | 3 | 100 / 100 / 50 | 1.00 / 0.70 | **~16.5** |
| **Quality** | 4 | 100 / 100 / 100 / 50 | 1.00 / 0.80 / 0.60 | **~23.4** |
| **Photo** | 4 | all 100 | 1.00 / 0.85 / 0.70 | **~27.5** |

The shape of them follows from the two measurements below: keep resolution full wherever the
budget allows, and spend what is left on one reduced pass, which is the cheap way to add volume.
Photo holds the later passes back less than the others — that restraint exists to protect motion,
and on a still it only costs you shaping.

## What a pass costs

Fitted on an RTX 5090 at 4K in The Blood of Dawnwalker, using the menu's own ms readout:

```
ms = 0.43 + 0.92 × passes + 5.85 × Σ(pass scale²) + 0.18 × (reduced passes)
```

| Passes | Per-pass resolution | Measured | Model |
|---|---|---|---|
| 3 | 100 / 100 / 100 | 20.72 | 20.74 |
| 3 | 100 / 88 / 69 | 16.70 | 16.71 |
| 3 | 100 / 75 / 50 | 14.14 | 14.15 |
| 4 | 100 / 80 / 62 / 45 | 17.66 | 17.67 |
| 4 | 100 / 100 / 100 / 50 | **23.43** | **23.30** ← predicted before it was run |

The useful term is the **0.92 ms every pass carries whatever raster it runs at**. Together with our
compose it puts a **floor of about 1.1 ms under any pass**: shrinking one cannot make it free, and a
pass only pays for its ladder step once it drops below roughly 98%.

Three laddered passes cost less than two full ones, and four laddered cost less than three full —
but read the next section before treating that as a win.

Your absolute numbers will differ. The shape should not: measure your own three points and refit.

## What the ladder actually buys

At the **same price** (~14 ms), two full passes against three with a 75 / 50 ladder, same scene,
70 seconds apart, HUD excluded, 20 stable scene blocks. Detail measured as the mean of per-8×8-block
luma standard deviation:

| Surface | 2 full | 3 laddered | |
|---|---|---|---|
| smooth / dark | 1.641 | 1.382 | **−15.8%** |
| medium | 3.114 | 2.874 | **−7.7%** |
| all 20 blocks | 2.808 | 2.571 | **−8.4%**, losing in 17 of 20 |

And against three full passes (20.72 ms) as the reference:

| Config | ms | detail | % of full | **detail per ms** |
|---|---|---|---|---|
| 2 full | 13.97 | 3.426 | 85.9% | **0.245** |
| 3 laddered 75 / 50 | 14.14 | 3.285 | 82.3% | 0.232 |
| 3 full | 20.72 | 3.989 | 100% | 0.193 |

So, plainly:

> **Resolution buys texture. Passes buy shaping.**

A reduced pass adds volume, depth and skin — none of which need a full raster, which is why it is
cheap. It does **not** add fine texture, because its residual is smooth by construction. Two full
passes beat a three-pass ladder of the same price on surface detail, on both counts at once. And
the third full pass costs 48% more for 16% more detail: real, and steeply diminishing.

Two things this does not measure, and they matter: **shaping**, which is the reduced pass's whole
product, and **halo**. A config that loses on this table can still be the one you prefer. It is one
game, one GPU, one dark scene, and a proxy for texture rather than for beauty.

Where the ladder is unambiguously right: when you want *more passes than you can afford at full
resolution*. That is what Balanced and Quality do.

## The knobs (menu names → ini keys, `[DlssNr]`)

| Menu | Ini | What it does | Our value |
|---|---|---|---|
| Passes | `Passes` | model runs per frame, 1–4 | 2–4, see presets |
| Pass 2/3/4 strength | `PassDecay2/3/4` | multiplier on Intensity + Local structure for that pass | 1.00 / 0.80 / 0.60 |
| Pass 2/3/4 resolution | `PassScale2/3/4` | that pass's raster as a fraction of the working size | full where affordable, one reduced pass on top |
| Model resolution | `WorkingScale` | pass 1's raster (all passes follow it). **This is the one that softens everything** — pass 1 carries the texture | 1.0 |
| Edge guard (halo) | `EdgeGuardMode` | 0 off · 1 soften · 2 no brightening · 3 luma lock · 4 detail only · 5 detail only + band | 0 — with per-pass strength and resolution the guard is no longer needed; 4 costs lighting |
| Edge guard strength / threshold / radius | `EdgeGuard` / `EdgeThreshold` / `EdgeRadius` | how much / what counts as a silhouette (1/z jump) / band width in px | 1.0 / 0.10 / 10–12 |
| Lock between passes | `EdgeBetweenPasses` | experimental; costs detail, leave off | false |
| Reversible proxy | `ReversibleMode` | 3 = hybrid (composed), 4 = hybrid + replace (the model's answer is the picture) | 4 on all three games |

Model tuning we ship: Intensity 1.5 (1.4 on REDkit), Local structure 1.1, Local tone 0.8, Skin 1.2
(0.89 on REDkit), Preset 3, auto skin mask, Detail strength 1.1, Colour 1.0. Style is taste per
scene: Natural for realism, Cinematic for punch, Default for "no filter".

Changing a pass's resolution rebuilds that pass's model feature, so the picture pops for a couple of
frames. That is the rebuild, not the setting — judge the image a second after you let go.

## Sizing it for your GPU

Use the cost formula rather than a tier table: measure **one** config on your machine, refit, and
pick a budget. Only the 5090 row below is measured; the rest are the formula scaled by raw
throughput, which is a guess. Please open an issue with your GPU, resolution and the menu's ms.

| GPU | Suggested start | ~ms |
|---|---|---|
| RTX 5090 / 5080, 4K | Quality, or Balanced while playing | 23 / 16.5 |
| RTX 4090 / 4080, 4K | Balanced | ~20 |
| RTX 4070 / 3080, 1440p | Performance | ~10 |
| RTX 3070 / 4060, 1440p | 1 pass, Model resolution 75% | ~7 |

If you cannot afford one pass, nothing here rescues that: pass 1 is the floor and it is upstream's,
not ours. What this fork gives you is the choice of where to spend once you can afford one.

## What this fork is, next to the other multi-pass work

Multi-pass DLSS-NR is being worked on in parallel: PRs [#23](https://github.com/Dagherbou/OptiScaler_DLSSNR/pull/23)
and [#26](https://github.com/Dagherbou/OptiScaler_DLSSNR/pull/26) on upstream, and the forks by
wilsjo2 and y4my4my4m. What this branch adds on top of "run the model again" is the part that
made more than one pass usable to our eyes: per-pass strength, per-pass resolution with the
residual compose, the depth-aware edge guard, the measured cost model above, and presets tested on
three engines. We would be glad to see any of it merged upstream.

## What we learned about the halo

The bright rim around a dark subject on a light background is not one pass's mistake: it is border
contrast the model pulls, summed over the passes. Things that attack the sum work (weaker and
smaller later passes, edge guard); things that try to erase it afterwards lose detail (we tried a
between-pass luminance lock — it is still in the menu, off). NVIDIA's own model at 1 pass still
pulls its share; that part only a model update can fix.

The per-pass resolution ladder came out of the same diagnosis and it does reduce the glow, because a
smooth residual cannot carry a hard rim. The measurements above are the other half of that story:
it cannot carry fine texture either. In stills the ladder held a character's mask, chainmail and
skin intact down to a pass at 50% — pass 1 was carrying them the whole time.

## Credits

Built on Dagherbou's OptiScaler_DLSSNR and the OptiScaler project. Matched-residual resolve from
hhkbble's PR. Everything else by THERMOTRON.
