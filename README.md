# Rattler — multi-pass DLSS Neural Rendering

> A fork of [OptiScaler_DLSSNR](https://github.com/Dagherbou/OptiScaler_DLSSNR), which is itself a fork of [OptiScaler](https://github.com/optiscaler/OptiScaler). Not affiliated with, or endorsed by, either project or NVIDIA. Upstream's own README is kept here as [README_OptiScaler.md](README_OptiScaler.md).

The name is the only thing here that is a joke: a rattlesnake's rattle is built of stacked
segments, one added at a time, which is what this does to the model's passes.

It runs NVIDIA's DLSS 5
Neural Rendering model **more than once per frame** — up to 4 real passes, each on its own model
feature — and adds the controls we needed to keep the picture clean when doing so:

- **Passes 1–4.** The passes run on **one** model feature, sharing one history, and each is fed the
  previous pass's answer. Every release up to v0.5.1 gave each pass its own feature deliberately;
  that was the thing costing us, and the measurement is below. Features are built one frame ahead so
  a pass-count change never stalls the frame. A pass at a *reduced* resolution keeps its own feature,
  because a feature built for one raster cannot be evaluated at another.
- **Same instructions on every pass** (`SharedHistory`, `ToneEveryPass`, `StillMv`): one history,
  tone sent to every pass rather than only the first, and — the one thing the passes must *not*
  share — the frame's motion. Passes two and up re-evaluate a picture that has not moved since pass
  one wrote it, so their honest displacement is zero. All three on by default from v0.5.2.
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
- **Jitter** (`Jitter`): sends `DLSSNR.JitterOffsetX/Y`. **This model build has no such parameter**
  — see below — so the switch does nothing here. It shipped in v0.5.1 described as a fix; it was not
  one, and the correction is kept rather than the claim.
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

## What the model does not have

Cost is area and the model will not scale, so the next question is whether anything *else* on its
interface can be reached for. We pulled every `DLSSNR.*` name out of NVIDIA's 165 MB
`nvngx_dlssnr.dll` and got **61 parameters**. What is there:

colour, depth (and a depth-inverted flag), motion vectors and their scales, output, width and
height, a scaling ratio, a back buffer, a **control mask**, a bidirectional distortion field, a UI
layer with alpha and a correction flag, enabled, reset, style, a render preset hint, intensity,
local structure, local tone, skin structure, an auto-mask flag — each with a subrect where it is a
resource.

What is **not** there, and matters:

- **No jitter.** The string "jitter" does not occur anywhere in the binary. `DLSSNR.JitterOffsetX/Y`
  was copied from RenoDX's add-on into our v0.5.1 and described there as a fix; this model build
  does not read it. Whatever the switch sends goes nowhere.
- **No input/output size family, no upscaling flag** that is honoured — see above.
- **No `GlobalToneStrength`**, though RenoDX's own settings file carries one.
- **No albedo, normal, roughness, specular, metallic, diffuse, G-buffer or irradiance input.**
  NVIDIA's official integration feeds the model colour, surface albedo, detailed lighting and
  surface normals from the engine's G-buffer. **We cannot.** We hook after the upscaler, where
  albedo and normals no longer exist, and this model build has no parameter to accept them if we
  had them. That is a ceiling on what any injector can do, ours included.

### The control mask does nothing here

`DLSSNR.ControlMask` sits beside `UseAutoMask` in the parameter list — the engine-level masking
NVIDIA describes. We painted one and handed it over. It is inert.

Five variants, all measured on the same wall:

| what the mask said | how it was bound | what changed |
|---|---|---|
| left half zero, right half one, auto mask on | at evaluate | boundary step 0.226 against a frame median of 0.173 — the largest steps in the frame were scene edges elsewhere |
| the same, auto mask off | at evaluate | 0.058 against a median of 0.177 — less than the median |
| all zero ("do nothing anywhere") | at evaluate | high-frequency energy 21.1 with the model on, 11.5 with it off |
| all zero | **at create** as well | cloth detail 32.59 against 10.99 with the model off, on a *closer* crop |
| all one | **at create** as well | 29.47 against 9.54 |

The fourth and fifth rows are the ones that matter: this tree's own forwarder documents that the
model reads its parameters once, when it builds the feature, so binding only at evaluate proves
nothing. Bound at create, with the mask telling it to do nothing anywhere, the model ran at full
strength. Both polarities tested, so it is not inverted — it is unread.

The `MaskTest` key that painted those masks is a diagnostic and is not in the release build.

**Reopened, and closed again with better evidence.** Pulling every ASCII string out of NVIDIA's
`nvngx_dlssnr.dll` (310.8.0.0, 165,840,496 bytes) turned up compiled CUDA kernels named for the
thing, sitting beside the ones the model normally runs:

```
cc_tinlayout_fused_post_block_swin_1h_32_simple_blend
cc_tinlayout_fused_post_block_swin_1h_32_control_mask
```

— plus the fp8 and full_rect variants of each, and a weight tensor named
`CC_Control_History_Blend_Quantize_With_Teacher_honest_tench_2026_07_04_22_30_weights`. So the path
is real, was trained deliberately, and is compiled in. That is a good reason to doubt a verdict
reached with one texture format at one model preset.

Twelve more prints said the same thing, on a much better bench: the frame frozen, the base taken by
un-checking *Apply the model* so it is the same pixels without the edit, and twenty seconds of
settling after every change. Eight prints across four model presets × mask off / all zero in
R8_UNORM, then R16_FLOAT and R32_FLOAT. **Every one of them landed between 1029 and 1038** — a spread
of 0.9%, with a pairwise difference of 0.68–0.88, which is this measurement's noise floor.

Neither the texture format nor the model preset selects it. The kernel exists and nothing an injector
can write reaches it.

### The four model presets are the same model

While the bench was up: `DLSSNR.Hint.Render.Preset` at Default, 1, 2 and 3 produced the same picture
within noise, at the same cost (6.99–7.16 ms). NVIDIA has said the models it ships have no
performance difference between them; here they have no visible difference either.

A caution worth publishing with it, because it cost us an hour: taken five seconds apart, the same
four presets appeared to differ, and preset 3 looked like a clearly better model. It was not. Both
bursts climbed from ~1031 to ~1110 in the same order regardless of which preset was selected —
changing the preset rebuilds the feature, changing the mask resets the temporal history (the DLL
carries `CG2R_ResetTemporalHistoryOnControlChange`), and a frame caught mid-settle measures the
transient. **Wait for the picture to stop moving before the shutter.**

## What the passes must not share

One history was the finding that closed the gap. It also introduced one of our own.

Passes two and up re-evaluate the *same frame*. The picture has not moved between them, so the
honest displacement is zero. We were handing every pass the frame's own motion vectors, which asked
one shared history to be warped once per pass — four times, at four passes, for one frame of real
movement. Where the model has pixels to correct from it hides the error; in shadow it has almost
none and leans on the history.

`StillMv`, on by default, gives the sharing passes zero motion. Measured with a five-shot ladder at
a fixed camera, one switch changed at a time, reading the darkest 15% of the frame:

| | brightness | standard deviation |
|---|---|---|
| model off | 9.66 | 5.02 |
| shared history, motion sent to every pass | 27.05 | 38.89 |
| shared history, `StillMv` | **24.99** | **36.94** |

Both numbers move back toward the base, and that switch is the only difference between those two
frames. It costs about 1.2 of the 2.8 of high-frequency energy the shared history adds. Walking and
turning through shadow with it on: no trailing.

**What this does not prove.** It was motivated by shadow flicker, measured from a still 13-second
capture where temporal standard deviation peaked at 52 against a scene mean of 1.4, every hot block
dark. On the retest the flicker did not reproduce with the switch either way. The link to that
flicker stays a hypothesis. What the change demonstrably does is put less invented energy into
shadow.

It may also be why turning the shared history off was steady: separate histories are each warped
once a frame, with the right vectors. Sharing is not the fault. Lying about motion is.

## Measured against RenoDX's add-on, matched

The add-on that drives the same model, at the same seam, is the only reference we have. Once its
defaults were matched on our side (intensity 1.0, local tone 1.0, local structure 1.0, skin 1.0,
auto mask on), from the same save, at the same camera — alignment `dy=2 dx=0`, normalised
cross-correlation up to 0.944 — and the same pass counts:

| gain over each one's own base | this fork | RenoDX |
|---|---|---|
| texture, 1 pass | +9.11 | +7.22 |
| texture, 2 passes | +21.41 | +17.68 |
| texture, 3 passes | +31.43 | +28.62 |
| grain on flat wood, 1 pass | +8.95 | +7.84 |
| grain on flat wood, 2 passes | +30.87 | +22.43 |
| grain on flat wood, 3 passes | +47.73 | +42.61 |

**Read that honestly: it is the same design at a different volume.** We add more of everything —
more texture where there is detail, and as much more grain where the wood is flat. The one real
difference is at two passes and it favours RenoDX: 17.68 of texture for 22.43 of grain, against our
21.41 for 30.87. At one and three passes the ratios tie.

An observer had reported our multi-pass as much inferior at the same cost. That report was correct
and it is what started all of this — but by the time the seams and the settings were actually
matched, what was left of the difference was **our intensity setting**, which was at 1.5 against
their default of 1.0. The eye that reported it settled at 0.67.

### What the model does to shadow is a local tone remap, not a wash

Taking the darkest 15% of the frame with the model off and reading the same pixels with it on:

| shadow lift over own base | this fork | RenoDX |
|---|---|---|
| 1 pass | +20.23 | +19.00 |
| 2 passes | +23.53 | +21.01 |
| 3 passes | +23.14 | +24.52 |

Both bases were the same darkness (10.04 against 11.46), so this is comparable, and for a while we
described it as both mods washing shadow. **That description was wrong**, and the aggregate is what
hid it. Split the dark pixels by how far they sit from the nearest lit pixel and the lift is not
spread at all — measured at four passes, model off as the base, with the game's post chain disabled
so nothing downstream could smear the result:

| dark pixels | base | lift at 4 passes |
|---|---|---|
| within 8 px of light — the contact shadow | 8.81 | **+13.21** |
| 8 to 25 px | 5.58 | +6.80 |
| beyond 25 px — real occlusion | 5.81 | **-0.15** |

**The black stays black.** What moves is the transition, which is where ambient occlusion lives. The
same shape appears when the pixels are binned by brightness instead of distance: the lift peaks at a
base value of about 14 (+16.98), falls off toward both ends, and turns slightly negative in the
mid-tones (-3.94 at a base of 94). The whole-frame mean moves by +3.2, so this is not exposure.

That is a dodge-and-burn curve — which is what the model's own `LocalTone` parameter is named for.
Whether a real contact shadow should be that soft is an aesthetic call and not ours to make; what is
measured is that the edit is structured and deliberate, not a black-level lift. The credit for
seeing it belongs to the eye, not the metric: it was reported as *the model is softening a contact
shadow that was too hard to begin with, and that is ambient occlusion's job* before it was measured.

**One asymmetry worth knowing.** Run the same table on `before` at four passes and the shadow half
matches, but the top half does not: mid-tones and highlights come down by about 20 levels across the
board (-20.42, -20.30, -18.47 against `after`'s -3.94, -0.41, +0.76), and the frame mean falls 9.5.
`after` does not do this. Unexplained so far, and a better candidate for what makes `before` look
worse than anything else we have measured.

### The game's post chain clips how much of the model reaches the screen

Our hook is at the NGX evaluate, before the interface is drawn and before the tonemapper. Film
grain, chromatic aberration, `r.Tonemapper.Sharpen`, bloom, depth of field and lens flare are all
applied **on top of** what the model just wrote. Lumen, ray tracing, ambient occlusion, textures and
LOD are upstream of it and are the model's raw material instead.

Measured in The Blood of Dawnwalker at DLAA, same camera across both rounds (cross-correlation
0.982), by how far the one-pass to four-pass ladder opens:

| | post chain on | post chain off |
|---|---|---|
| `after` | ×1.470 | **×1.714** |
| `before` | ×1.453 | ×1.446 |

**At `after`, taking the post chain out delivers about 17% more of the model's work to the screen**,
and the gain is present in all four horizontal quarters of the frame, so it is not just depth of
field sharpening the background. At `before` nothing changes, because its output still has to cross
the upscaler's own temporal reconstruction, and that ceiling is the tighter of the two.

This also revises what an earlier DLAA test concluded. At matched working size the two placements
had looked equivalent; they were equivalent because the post chain was clipping both to the same
level. With it out of the way, `after` gains +994.6 of texture over the neutral base at four passes
against `before`'s +649.9.

**So: if you run at `after`, turn the game's post-processing off.** In this game that is
`r.FilmGrain=0`, `r.Tonemapper.Sharpen=0`, `r.BloomQuality=0`, `r.DepthOfFieldQuality=0`,
`r.SceneColorFringeQuality=0` in `Engine.ini`. You get roughly a third more of what the
milliseconds already bought.

Two honest limits. The metric counts invented grain the way it counts recovered detail, and removing
film grain changes the high-frequency floor of everything — the ratio test is built to survive an
additive floor, and the `before` control, which did not move at all, is what rules that explanation
out. And turning these off changes the look the game's art direction chose; more of the model
arriving is not the same claim as a better picture.

### RenoDX's Upscaled hook and Frame Generation crash the GPU

Not our bug, published because it cost us an afternoon. With that add-on's hook point set to
*Upscaled* and DLSS-G active, the game died about two seconds after startup with an Unreal
`GPUCrash` — no Windows fault event, no Streamline minidump, a device removal. Its log runs clean
until `captured first loaded-module D3D12 reconstruction evaluation (slot 0)`, the moment the hook
latches onto the DLSS output, and stops about 100 ms later. Its own log says `replace_source=true`:
it writes into the buffer Frame Generation consumes. **With Frame Generation off, the same
configuration ran 3.5 minutes and 19,000 evaluations clean.**

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

**Model resolution above 100% costs more and delivers less.** The slider goes to 200% and the guess
was that a finer raster gives the model more structure to work with. Measured against a frozen frame
with a pixel-identical base, at `after` on a native 4K output:

| model resolution | the model's raster | ms | texture gained | texture per unit of grain |
|---|---|---|---|---|
| **100%** | 3840×2160 | 6.99 | **+125.8** | **0.48** |
| 145% | 5568×3132 | 15.06 | +43.5 | 0.26 |
| 175% | 6720×3780 | 22.31 | +41.6 | 0.29 |
| 200% | 7680×4320 | 29.92 | **+28.0** | 0.23 |

At 200% it costs 4.3× and returns 22% of the gain, and the ratio of texture to grain gets worse as
well — it is not less of the same, it is worse distributed. The mechanism is the trip home: the model
writes detail at 7680×4320 and Lanczos3 samples it back to 3840×2160, which destroys exactly the fine
structure it just added. It is drawing above the frequency the output can carry.

Cost itself is honest — it tracks area almost exactly (11.83/15.08/22.33/29.95 measured against
11.97/14.90/21.66/28.32 predicted). You are simply paying it for nothing.

**This says nothing about supersampling up to native.** At `before`, where the model runs on the
game's render resolution, 145% of a 2558×1439 render is 3709×2087 — just under a 4K output, not above
it. That is a different question and it is untested here.

**The aggregate hid the shape.** For three days this file said both mods *wash shadows* — the
darkest 15% of the frame goes from about 10 to about 30, so it looked like a black-level lift. It is
not. Split those same pixels by distance from the nearest lit pixel and the deep occlusion does not
move at all; the whole lift sits in the contact-shadow transition. One number over a large mask can
be perfectly correct and still describe the wrong mechanism. Decompose before you name a cause — and
the decomposition here was suggested by the eye, which had already said the model was softening a
contact shadow rather than lifting black.

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
- **We shipped a fix for something the model cannot receive.** v0.5.1's headline was that the jitter
  was now being sent to the model. Pulling every parameter name out of the model's own binary shows
  it has no jitter input at all — the string does not occur in 165 MB. The name came from RenoDX's
  add-on and was taken on trust. It was reported as making no visible difference, which was true and
  should have been the clue rather than the footnote.
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
| Jitter | `Jitter` | sends `DLSSNR.JitterOffsetX/Y` | **this model build has no such parameter** — the switch does nothing, see above |
| One history | `SharedHistory` | every full-size pass on one model feature, sharing one history, instead of one each | true from v0.5.2 — a reduced pass keeps its own, so the ladder still works |
| Tone on every pass | `ToneEveryPass` | send Local tone to every pass, not only the first | true from v0.5.2 — halve the value against what you used on one pass |
| Later passes see no motion | `StillMv` | the passes that share the history are given zero motion, because between them the frame has not moved | true from v0.5.2 — only applies to passes that share; a reduced pass keeps its own history and its own vectors |
| Scale test | `ScaleTest` | diagnostic: asks at feature creation whether the model takes an input below its output, on a throwaway feature, and logs it | 0 — the answer is in the section above and it is no |

**Recommended look**, the button beside the Intensity slider, fills the four strengths and the three
switches at once: Intensity **0.67**, Local structure **1.26**, Local tone **0.25**, Skin structure
**1.09**, with One history, Tone on every pass and Later passes see no motion all on. *Model default*
beside it puts the four back where NVIDIA ships them (1, 1, 1, and −1 for skin, which means follow
local structure rather than a strength of zero) and leaves the switches alone.

Intensity sits below NVIDIA's own default of 1 on purpose. Everything through v0.5.1 shipped 1.5,
and measured against the same scene at 1.0 the higher setting added more of *everything* — more
texture where there was detail, and as much more grain where the wood was flat. Local structure goes
the other way because it carries detail without the brightness lift that washes shadow. Local tone is
0.25 rather than the 0.5 that felt right with tone on the first pass only, because with tone on every
pass it enters once per pass: 0.5 across two sits about where 0.8 across one did, which at three
passes puts 0.5-on-one near 0.23 each.

Style is taste per scene: Natural for realism, Cinematic for punch, Default for "no filter".

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
