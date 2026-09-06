# OptiScaler DLSS-NR — multi-pass with halo controls (THERMOTRON / Blue Rattler fork)

A fork of [OptiScaler_DLSSNR](https://github.com/Dagherbou/OptiScaler_DLSSNR) that runs NVIDIA's DLSS 5
Neural Rendering model **more than once per frame** — up to 4 real passes, each on its own model
feature — and adds the controls we needed to keep the picture clean when doing so:

- **Passes 1–4.** Each pass is a separate NGX feature fed the previous pass's answer (no shared
  history, so no smear), built one frame ahead so a pass-count change never stalls the frame.
- **Per-pass strength** (`PassDecay2/3/4`): the later passes run with Intensity and Local structure
  scaled down. The silhouette glow ("halo") is border contrast the model pulls, and N passes pull it
  N times; weaker later passes pull less of it while the model still sees the whole frame.
- **Per-pass resolution** (`PassScale2/3/4`): each later pass at its own fraction of the working
  size. It is shown the current picture shrunk, and what it adds comes back as a residual over the
  full-size picture — pass 1's fine texture is kept whole; the later passes run smaller, cheaper and
  softer. Go **down** the ladder (100 / 85 / 70), never up.
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

## The knobs (menu names → ini keys, `[DlssNr]`)

| Menu | Ini | What it does | Our value |
|---|---|---|---|
| Passes | `Passes` | model runs per frame, 1–4. Cost is linear (~7 ms per pass at 4K on a 5090) | 3 |
| Pass 2/3/4 strength | `PassDecay2/3/4` | multiplier on Intensity + Local structure for that pass | RE Engine 0.93 / 0.50 · REDkit 1.00 / 0.89 |
| Pass 2/3/4 resolution | `PassScale2/3/4` | that pass's raster as a fraction of the working size | RE Engine 0.93 / 0.80 · REDkit 1.00 / 1.00 (or 0.95 / 0.83 to save 2.5 ms) |
| Model resolution | `WorkingScale` | pass 1's raster (all passes follow it) | 1.0 (0.70 is a good cheap preset) |
| Edge guard (halo) | `EdgeGuardMode` | 0 off · 1 soften · 2 no brightening · 3 luma lock · 4 detail only · 5 detail only + band | 0 — with per-pass strength and resolution the guard is no longer needed; 4 costs lighting |
| Edge guard strength / threshold / radius | `EdgeGuard` / `EdgeThreshold` / `EdgeRadius` | how much / what counts as a silhouette (1/z jump) / band width in px | 1.0 / 0.10 / 10–12 |
| Lock between passes | `EdgeBetweenPasses` | experimental; costs detail, leave off | false |
| Reversible proxy | `ReversibleMode` | 3 = hybrid (composed), 4 = hybrid + replace (the model's answer is the picture) | 4 on all three games |

Model tuning we ship: Intensity 1.5 (1.4 on REDkit), Local structure 1.1, Local tone 0.8, Skin 1.2
(0.89 on REDkit), Preset 3, auto skin mask, Detail strength 1.1, Colour 1.0. Style is taste per
scene: Natural for realism, Cinematic for punch, Default for "no filter".

## Presets by GPU tier (starting points — report your ms)

The first pass is where the detail comes from; the later passes mostly add polish and glow.
So the cheap way down the ladder is: keep pass 1 full, shrink the later ones, then lower Model
resolution. Only the 5090 row is measured (4K); the others are proportional estimates — please
open an issue with your GPU, resolution and the ms readout from the menu.

| GPU | Passes | Pass 2 / 3 strength | Pass 2 / 3 resolution | Model resolution | ~ms |
|---|---|---|---|---|---|
| RTX 5090 / 5080, 4K | 3 | 0.93 / 0.50 (RE Engine) · 1.00 / 0.89 (REDkit) | 93 / 80 · 100 / 100 | 100% | 18–21 |
| RTX 4090 / 4080, 4K | 3 | 0.90 / 0.50 | 90 / 70 | 100% | ~22–26 |
| RTX 4070 / 3080, 1440p | 2 | 0.80 | 70 | 85% | ~10 |
| RTX 3070 / 4060, 1440p | 1 | — | — | 75% | ~7 |

`presets/OptiScaler.lowend-2x.ini` is the 2-pass starting point. Two passes with pass 2 at 70%
cost about 1.5x a single pass, and it is still real multi-pass: each pass on its own model
feature, fed the previous answer.

## What this fork is, next to the other multi-pass work

Multi-pass DLSS-NR is being worked on in parallel: PRs [#23](https://github.com/Dagherbou/OptiScaler_DLSSNR/pull/23)
and [#26](https://github.com/Dagherbou/OptiScaler_DLSSNR/pull/26) on upstream, and the forks by
wilsjo2 and y4my4my4m. What this branch adds on top of "run the model again" is the part that
made more than one pass usable to our eyes: per-pass strength, per-pass resolution with the
residual compose, the depth-aware edge guard, and presets measured on three engines. We would be
glad to see any of it merged upstream.

## What we learned about the halo

The bright rim around a dark subject on a light background is not one pass's mistake: it is border
contrast the model pulls, summed over the passes. Things that attack the sum work (weaker and
smaller later passes, edge guard); things that try to erase it afterwards lose detail (we tried a
between-pass luminance lock — it is still in the menu, off). NVIDIA's own model at 1 pass still
pulls its share; that part only a model update can fix.

## Credits

Built on Dagherbou's OptiScaler_DLSSNR and the OptiScaler project. Matched-residual resolve from
hhkbble's PR. Everything else in `feat/dlssnr-multipass-frame-ahead` by THERMOTRON, with Claude
as lab partner.
