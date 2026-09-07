# Rattler — multi-pass DLSS Neural Rendering

> A fork of [OptiScaler_DLSSNR](https://github.com/Dagherbou/OptiScaler_DLSSNR), which is itself a fork of [OptiScaler](https://github.com/optiscaler/OptiScaler). Not affiliated with, or endorsed by, either project or NVIDIA. Upstream's own README is kept here as [README_OptiScaler.md](README_OptiScaler.md).

The name is the only thing here that is a joke: a rattlesnake's rattle is built of stacked
segments, one added at a time, which is what this does to the model's passes.

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
- **Model runs before or after the upscaler** (`Placement`): the model's cost is very nearly the
  area it works on, and after the upscaler that area is the display resolution with nothing to be
  done about it. Run it before instead and the area becomes the game's own render resolution.
  Cheaper, and worse — both measured, both below. A switch, not an upgrade.
- **Colour guard** (`ChromaGuard`): bounds how far a pixel's colour may travel from the frame's,
  the way the highlight guard already bounds how much brighter it may get. Nothing bounded colour,
  and with several passes each recolours what the last one recoloured; on skin that ends at
  magenta. Costs no detail, because detail is luminance. Off by default.
- **UI correction** (`UiCorrection`): whether the model corrects for a drawn interface. It shipped
  hardcoded **on** from v0.2.0 to v0.5.0, with no way to turn it off, in a pass that hands the model
  no UI layer, no alpha and no composited back buffer. Off by default now, and a setting — because
  tried both ways it made no visible difference, and a belief nobody can test is not worth shipping
  as a constant.
- **Jitter** (`Jitter`): the model has `JitterOffsetX/Y` inputs and this fork had never written
  them, so a temporal model was being shown sub-pixel-offset frames without being told they were
  offset. Now sent, and a switch. Also no visible difference, which is worth saying plainly.
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
| **Performance** | 1 | 100 | 1.00 | **~7.2** |
| **Balanced** | 2 | 100 / 100 | 1.00 | **~14.0** |
| **Quality** | 3 | 100 / 100 / 100 | 1.00 | **~20.7** |
| **Photo** | 4 | all 100 | 1.00 | **~27.5** |

Every rung is the same thing N times: N passes, full size, every pass given the same instructions.
Nothing else differs between them, so the cost of a rung is the cost of a pass — about seven
milliseconds at 4K on a 5090 — and the menu cannot mislead about it.

They laddered until v0.5.2: later passes weaker and smaller. That ladder was compensating for later
passes being told something different from the first, which is the subject of the section after
next. With that fixed it stopped paying — three identical passes measured **20.69 ms** against the
old four-pass Quality at **23.17**, and were the ones preferred. Per-pass strength and resolution
are still there, below the presets, for anyone who wants them.

**If a rung costs more than you have, raise Intensity before you add a pass.** Across 1.00 to 1.50
the measurement does not move, and from 1.20 up it carried more grain than an extra pass did. The
pass is seven milliseconds; the slider is free.

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

A note on that table now that the presets no longer ladder: those rows are still what a laddered
chain costs, and the formula still predicts them. They are the reason the ladder was built and the
reason it is no longer the default — cheap on paper, and the thing it bought turned out to be a
compensation for a bug in how later passes were driven.

## The model will not scale

Cost is area, so the question worth asking is whether the model can be handed a small picture and
asked for a big one. It cannot.

It carries an input size, an output size, an upscaling flag and a scale, and its binary computes a
scaling ratio. All of it is inert. Asked at feature creation — the only place the model reads its
parameters, since everything set at evaluate is ignored — for six ratios in one session:

| model is told its input is | and its output is | create |
|---|---|---|
| 3456x1944 | 3840x2160 | succeeded |
| 2880x1620 | 3840x2160 | succeeded |
| 1920x1080 | 3840x2160 | succeeded |
| 1267x712 | 3840x2160 | succeeded |
| 960x540 | 3840x2160 | succeeded |
| **5760x3240** | **3840x2160** | **succeeded** |

The last row is an input larger than the output, which no upscaler can honour, and it was accepted
like the rest in the same 100 ms. Nothing in that family is read. The model works at `DLSSNR.Width`
by `DLSSNR.Height` and nowhere else, and its own scaling-ratio callback says the same: 1.0000 for
every quality mode, with UltraPerformance refused.

That settles three things. The reduce, run, enlarge behind `Model resolution` and the per-pass ladder
is not a shortcut taken here — it is the only way to work below the output size. `Model runs = before`
loses for the same reason and no parameter fixes it. And the one route left to a cheaper pass is
fewer passes, which is a question about how much a pass gives back, not about how big it is.

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

## Where the model runs, and what it costs

Everything above assumes the model runs after the upscaler, on the finished frame. It does not have
to. `Model runs` in the *Cost* section moves it to before the upscaler, where it works on what the
game actually rendered.

Same scene, same settings, one switch:

| | model works on | ms |
|---|---|---|
| after the upscaler | 5760x3240 | **39.49** |
| before the upscaler | 1920x1080 | **14.45** |

Read the setup before you read the ratio: that was DSR 2.25 with DLSS Ultra Performance, so the
output was **nine times the area of the render**, which is why the gap is that wide. At plain 4K
with DLSS Quality the same switch is nearer 27.5 against 14.5. The further apart the render and the
display are, the more running early saves.

**It is not free, and the loss is visible.** Detail synthesised at render resolution is
render-resolution detail, and the upscaler then enlarges it.

Tested in a bright scene, a dark scene, at matched cost, and at two setups far apart — DSR 2.25 with
DLSS Ultra Performance, and plain 4K with DLSS Quality, where the render is nearly the display and
the theory said the loss should vanish. It did not. Before degrades noticeably in both.

The practical form of that: **before came out worse than every `after` configuration tried, including
the cheap ones.** So if the model at display resolution costs more than you have, the better move is
to stay on `after` and spend less there — fewer passes, or a lower Model resolution — rather than to
move it early. This switch is for a card that cannot afford `after` at all, and for finding out
whether that is true of yours.

The idea is not ours. It is what the [Neural Upstream](https://github.com/matiasLombo) ReShade
addon does; what this fork adds is the same choice inside OptiScaler, as a switch, so the two can
be compared side by side without swapping mods. Direct3D 12 only — the D3D11 and Vulkan bridges
have no seam before the upscaler and keep running after it.

## Three numbers the menu now shows

Under the ms readout: what the model works on, what the frame is, and what the game actually drew.

```
model 5760x3240   frame 5760x3240   game drew 1920x1080
   the model works on 9.0x the pixels the game drew
```

That line is there because its absence cost us an afternoon. Nothing above "game drew" is real —
past it the model is elaborating on its own guesses — and if a downsampler (DSR, DLDSR, output
scaling) sits after all of this, whatever was synthesised above the display's own resolution is
averaged away before anyone sees it. Both were true at once, and only the log knew.

## Things we got wrong, so you do not have to

**The ms readout is not cost per frame.** It is GPU time between the pass's own start and end
markers, and anything the GPU waits on in between is inside it. Lightly loaded the two agree, which
is why the cost model below fits. Under load they diverge: we have seen it report 53 ms inside an
18 ms frame, which is impossible as a per-frame cost. When that happens, trust the frame rate.

**The cost formula assumes a fixed output.** It was fitted at 4K and it holds there. Move the
output and it misses — 18%, 38%, 45% over as the output grew, with the model's own size unchanged.
The area term is real; there is a second term for the frame around it that we have not measured.

**Model resolution below ~80% swims.** Reducing it is the largest saving available, and on a still
frame you can go to 67% and barely see it. In motion, faces and small objects start to shimmer: the
game jitters its camera every frame, and a reduced raster lands on a different phase each time.
79–80% was where it stopped, measured by eye on a 5090 in one game. Test yours.

**A detail metric is not a quality metric.** We measured per-block luminance variation and it told
us the before-upscaler mode had 22% *more* detail at matched cost. It does not: noise, ringing and
enlarged synthetic detail all raise that number the same way real texture does. The eye said
otherwise across many configurations and the eye was right. A reference-based comparison would have
caught it; ours had no reference.

- **Reading a ratio is not finding a cause.** Three model parameters came back at exactly 0.6 of
  what the menu said, which looked like the model clamping its inputs and got written up as such.
  It was this fork's own per-pass decay, multiplying them ten lines above the call. The model had
  received exactly what it was sent. Check your own code path before you accuse someone else's.
- **A blink when you toggle a setting is the setting changing, not the setting working.** Two of the
  changes in v0.5.1 rebuild the model feature, which costs a visible hitch. That hitch was briefly
  mistaken for the effect.
- **Tone is a model parameter, not a colour filter.** Local tone was zeroed on every pass but the
  first, so that a look would not be applied twice. What that actually did was run pass two with a
  different model configuration from the one that had produced the picture pass two was handed.
  Sending it on every pass is the single change that closed the gap reported against RenoDX's
  add-on. Watch the dose: arriving twice at 1.0 read as wax on skin, and 0.5 across two passes sits
  about where 0.8 on one did.
- **`PassDecay` and the resolution ladder were compensating for that.** Later passes were weakened
  and shrunk because multi-pass compounded glow and shimmer. Given the same instructions, two passes
  at full strength beat three laddered ones on grain and on stability, for six milliseconds less.
  Both controls remain and both still do what they say — they are no longer the first thing to reach
  for.
- **Reading a parameter block says who wrote, not what is supported.** The size family above was
  first declared missing because `DLSSNR.InputWidth` and the rest read back
  `FAIL_UnsupportedParameter`. They read back that way because nobody had written them:
  `DLSSNR.Width` answers only because the forwarder's create writes it, and an unwritten name is
  indistinguishable from a refused one. Written, they all read back fine — which was then taken for
  the model using them, and is not that either. A block that keeps a name and a model that reads one
  are different claims, and only a create can tell them apart.

## Known issue: `Model runs = before` crashes in RE Engine

Switching **Model runs** to *before the upscaler* crashes Onimusha: Way of the Sword, on the first
frame the model runs, every time. It works in The Blood of Dawnwalker, Starfield and Mortal Shell II.

We spent an evening on it and **do not know why**. What it is not, each ruled out by measurement:

- not the render subrect, and not a resolution mismatch — every size in the log agrees
- not toggling the switch at runtime — it crashes just as reliably from a clean start
- not Streamline — Starfield's upscaler runs behind it too, and survives
- not ReShade — all four games have it
- not the multi-pass chain — one pass crashes the same
- not the colour buffer's arrival state — declaring it changed nothing

The fault lands inside NVIDIA's D3D12 user-mode driver, on the frame the model's feature is created.
Two things that were wrong in that path *were* fixed in this release and neither was the crash.

**The default is `after`, and nothing here affects anyone who leaves it there.** If your game dies
when you flip that switch, flip it back and tell us the engine — that list above is only four games
wide and the pattern in it is what we are missing.

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
| Model runs | `Placement` | 0 = after the upscaler (display resolution) · 1 = before it (render resolution) | 0 — 1 is for a card that cannot afford 0 |
| Colour guard | `ChromaGuard` | how far a pixel's colour may travel from the frame's, as a ratio; below 1.0 it does not run | 1.00 — holds the hue to the game's, which is what stopped the magenta |
| UI correction | `UiCorrection` | whether the model corrects for a drawn interface it is not given | false — on until v0.5.0 and invisible either way |
| Jitter | `Jitter` | hand the model the game's sub-pixel camera offset | true — confirmed arriving, no visible difference |
| One history | `SharedHistory` | every full-size pass on one model feature, sharing one history, instead of one each | true from v0.5.2 — a reduced pass keeps its own, so the ladder still works |
| Tone on every pass | `ToneEveryPass` | send Local tone to every pass, not only the first | true from v0.5.2 — halve the value against what you used on one pass |
| Scale test | `ScaleTest` | diagnostic: asks at feature creation whether the model takes an input below its output, on a throwaway feature, and logs it | 0 — the answer is in the section above and it is no |

Model tuning we ship: Intensity 1.5 (1.4 on REDkit), Local structure 1.1, Local tone 0.8, Skin 1.2
(0.89 on REDkit), Preset 3, auto skin mask, Detail strength 1.1, Colour 1.0. With `ToneEveryPass` on
and more than one pass, Local tone arrives once per pass — 0.5 across two is about what 0.8 across
one was. Style is taste per
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

If you cannot afford one pass after the upscaler, try `Model runs` = before. That is the one lever
here that changes the floor rather than dividing what sits above it, and it is the reason the switch
exists — with the loss in the section above understood and accepted, not discovered later.

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
hhkbble's PR. Running the model before the upscaler is the idea behind matiasLombo's Neural
Upstream ReShade addon; the implementation here is our own, and the switch exists so the two
placements can be compared without swapping mods.

The gamut compression is clshortfuse's, from RenoDX (see `Licenses/RenoDX_ATTRIBUTION.txt`), and
RenoDX earned a second mention in v0.5.1: comparing this fork's parameter list against its DLSS 5
add-on's is what found the jitter inputs we had never written and the UI correction we had been
sending unconditionally into a pass with no interface in it. Neither turned out to change the
picture, but neither was defensible before and both are settings now.

Everything else by THERMOTRON.

Licensed GPL-3.0, same as upstream. This is a modified version: the changes are the ones listed
at the top of this file, and upstream's own README is kept intact as README_OptiScaler.md.
"Rattler" names this fork only — it is not an OptiScaler release and carries no endorsement from
the OptiScaler project or from NVIDIA.
