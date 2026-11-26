# BOB Implementation

BOB Implementation (Best of Both Worlds) assembles compatibility work already published by the VIctory community and related projects. Credit for each fix belongs to its original authors. BOB’s own changes are limited to the small amount of glue needed to combine that work.

BOB starts from Belobug’s Radeon build and assembles existing D3D12 compatibility patches for GeForce, with the desktop RTX 5070 as the test target. The fork carries compatibility-layer source under GPL-3.0-or-later. See [LICENSE](LICENSE).

## Radeon baseline

| Belobug branch at intake | Exact revision | Basis for selection |
| --- | --- | --- |
| `a-publier` (default, selected) | [`4eb6cfd`](https://github.com/Belobug/VIctory-Belobug-AMD/commit/4eb6cfdf9e61cb112f1f811344ec0dd309eddd6c) | Contains the RX 9070 XT port and the RDNA4 shader compiler fix from Belobug PR #324. |
| `rx9070xt` | [`cd8dfc0`](https://github.com/Belobug/VIctory-Belobug-AMD/commit/cd8dfc06ace7ccb91a19431e74bc2d0de2a10145) | Ancestor of the selected revision, 49 commits behind. |
| `main` | [`4f62c2a`](https://github.com/Belobug/VIctory-Belobug-AMD/commit/4f62c2a5e27e7beebf3e8d68f77898e43dc7b77c) | Diverged branch with a different graphics and shader-cache implementation. Base for PR #221. |

The reference configuration is the pinned `victory.example.toml`, `scripts/play.ps1` and built-in defaults. The reference launch uses Series X, `diag`, light DRED and `VICTORY_AMBIENT=0.8`, with no `-More` overrides. `-RayTracing`, `-Fsr4K` and `-Safe` are opt-in. The five `VICTORY_PORT_*` renderer features listed below default to off. BOB’s offline verification uses the pinned Rust 1.98.1 toolchain, `--locked`, no external title paths and an explicit adapter selection for hardware tests.

The 44.9 ms RX 7900 XTX result below comes from the inherited [AvatarManz benchmark](https://github.com/avatarmanz/VIctory-7900XTX/commit/c9c932c), with the Series S profile, ray tracing off and renderer patches 1, 2, 5, 6 and 7 enabled. It describes that donor configuration. The RX 9060 XT and RTX 3090 entries below are community reports on the revisions listed there.

## Imported patches

| Purpose | Original author | Source repository | Exact patch | BOB adaptation | Donor evidence | BOB verification |
| --- | --- | --- | --- | --- | --- | --- |
| Honor the existing descriptor reserve override on capped view heaps | trustno1-coder | [VIctory-NVIDIA](https://github.com/trustno1-coder/VIctory-NVIDIA) | [`a9632bc`](https://github.com/trustno1-coder/VIctory-NVIDIA/commit/a9632bc8de343aa20adf4f1123f16445f5b1b21b) | Donor code as published. [BOB glue](https://github.com/Galygious/BOB-implementation/commit/cf328f3) retains the baseline fallback when the heap cap is unknown. | The donor reports 1,100–4,800 refused tables per frame at the default reserve and 0–33 after requesting `524288` on an RTX 5070. | Diagnostics build passed. 37 descriptor tests passed on the desktop RTX 5070 at defaults and 37 again with the override. 37 passed on the Radeon iGPU at defaults. |
| Preserve escaping scalar results from eligible divergent loops in single-wave compute groups on Wave32 hosts | fladit123, with original fixtures by loyodea | [fladit123/VIctory-Belobug-AMD](https://github.com/fladit123/VIctory-Belobug-AMD) | [Belobug PR #366](https://github.com/Belobug/VIctory-Belobug-AMD/pull/366), [`ac445ea`](https://github.com/fladit123/VIctory-Belobug-AMD/commit/ac445ea90803500b610c801df236d98f5bff47eb) | Donor code and fixtures as published. [BOB glue](https://github.com/Galygious/BOB-implementation/commit/a8ed57d) changes the translator cache key for max-wave-32 hosts. | The donor reports matching synthetic results on an RTX 4060 for simple, nested and consecutive loops. [Original fixtures](https://github.com/Belobug/VIctory-Belobug-AMD/pull/299) | Diagnostics build passed. Wave64 suite 32 passed on each adapter. Eight added hardware readback fixtures passed on the RTX 5070, three on the Radeon iGPU. The synchronized path is skipped on the iGPU by capability. Three cache tests passed. |
| Preserve both words of an accumulated Wave64 branch mask in the existing Wave32 retry | ANDREANIADS (Git author IronMan) | [ANDREANIADS/VIctory-Belobug-AMD](https://github.com/ANDREANIADS/VIctory-Belobug-AMD) | [Belobug PR #345](https://github.com/Belobug/VIctory-Belobug-AMD/pull/345), `6ca7b5c` (fixture) and `3dac8db` (fix) | Donor translation code as published. [Cache marker](https://github.com/Galygious/BOB-implementation/commit/3db7d27) and [fixture scope](https://github.com/Galygious/BOB-implementation/commit/c7cb373) follow the existing max-wave-32 retry. | The donor reports 512 matching words on an RTX 3070 over four dispatches. | Diagnostics build and 11 IR tests passed. BOB reproduced the pre-patch refusal at `0x38` on the RTX 5070, then 512 matching words each on RTX 5070 and RTX 3070. The Radeon iGPU gave the same refusal before and after donor code. The final fixture records a capability skip there. |
| Return dynamic descriptors after synchronous whole-list submissions on a capped heap | CwWwWwW (Git author cccw) | [fxceless/victory-fork](https://github.com/fxceless/victory-fork) | [PR #1320](https://github.com/fxceless/victory-fork/pull/1320), [`af23c65`](https://github.com/fxceless/victory-fork/commit/af23c6529ddc242107d2f2b09be268606e963685) | Donor call as published. [BOB glue](https://github.com/Galygious/BOB-implementation/commit/1c110ef) uses the existing 1,000,000-slot capability probe to enable it. | The donor identified a missing reclaim in the `VICTORY_CUT_WAITS=0` path and reports 217 GPU-library tests passed. | Diagnostics build passed. 445 GPU-library tests passed on each of the RTX 5070 and Radeon iGPU, with 34 ignored per adapter. Five queue tests passed with the switch off on the RTX 5070. Live split-submission reclamation remains unverified. |

`VICTORY_DYNAMIC_SLOTS` is opt-in on capped heaps. When unset the reserve is 262,144, as at the baseline. On a 1,000,000-slot heap with one null slot, the upper bound is 499,999. A larger reserve leaves fewer title descriptors mirrored in place. The uncapped sizing path uses Belobug’s existing policy. The reclamation patch is on the existing whole-list wait path (`VICTORY_CUT_WAITS=0`), for capped heaps only.

## BOB verification

| Adapter | Diagnostics build | Shader tests | GPU library tests | Retail rendering | Playable progression |
| --- | --- | --- | --- | --- | --- |
| NVIDIA GeForce RTX 5070, desktop, driver 610.88 | Pass | Full library 316 passed, 4 ignored. Hardware reported 32–32 lanes. | 445 passed, 34 ignored | Pending separate run | Pending separate run |
| NVIDIA GeForce RTX 3070, driver 610.88 | Same binary | Wave64 suite 33 passed, including the branch-mask readback. | Pending | Pending separate run | Pending separate run |
| AMD Radeon(TM) Graphics, integrated, driver 32.0.11024.2 | Same binary | Full library 316 passed, 4 ignored with capability skips. Hardware reported 32–64 lanes. | 445 passed, 34 ignored | Pending separate run | Pending separate run |
| Radeon RX 9070 XT / RX 7900 XTX | Same binary | Pending hardware access | Pending hardware access | Pending hardware access | Pending hardware access |

The adapter names came from `victory translate <empty-directory> --pipelines` using `VICTORY_D3D12_ADAPTER` by name. The empty-directory probe establishes D3D12 device selection. The allocator tests use synthetic descriptors and the shader suite includes hardware readbacks, WARP controls and capability skips.

The pinned baseline shader library reported 303 passes and 4 ignored on both the desktop RTX 5070 and Radeon iGPU. The final suites include those cases and the donor fixtures. Tests with a built-in capability skip can report `ok` in the harness. The per-patch evidence above records which donor hardware fixtures executed. The asset-policy check and Clippy for `victory-gpu` and `victory-shader` with diagnostics passed.

```powershell
cargo build --locked --profile diag -p victory --features diagnostics
$env:VICTORY_D3D12_ADAPTER = 'RTX 5070'
cargo test --locked -p victory-shader --lib -- --test-threads=1 --nocapture
cargo test --locked -p victory-gpu --lib --features diagnostics -- --test-threads=1 --nocapture
cargo xtask policy
```

## Donor review

| Candidate | Disposition on the pinned baseline |
| --- | --- |
| Wave64 compute with local shared memory and multiple console waves, spook40. [Belobug PR #221](https://github.com/Belobug/VIctory-Belobug-AMD/pull/221) | The PR targets `main`. `a-publier` independently splits eligible compute waves, assigns exchange slots per console wave and checks where group barriers can safely run. BOB ran the existing multi-wave and LDS fixtures, including 14 LDS flag dispatches on the RTX 5070. |
| Resource barrier state sanitation, domainawareness420-beep. [Belobug issue #13](https://github.com/Belobug/VIctory-Belobug-AMD/issues/13) | `HostList::transition` legalizes states at insertion and replacement and drops equal-state transitions. The posted snippet duplicates that protection on this branch. |
| Capped descriptor heaps, Vaximous and contributors. [Marvisak issue #747](https://github.com/Marvisak/VIctory/issues/747) | The baseline already keeps 262,144 dynamic slots on a 1M heap and clamps at its null slot. The same reserve increase is in the linked LAKSHYA-cmd-tech, MfDev0, MrEjecutor and [AimLogin layout](https://github.com/AimLogin/VIctory/commit/3e3b8c5841213fbd316a0875c06c5f85cb5200c2) alternatives. The [AimLogin typed guards](https://github.com/AimLogin/VIctory/commit/cc449ca941abea558dc56271c10f7ce0459ab66a) depend on its layout, shader-root and binding changes and would need a port. |
| VItoriaBr and DexNom. [VItoriaBr](https://github.com/uiliannn/VItoriaBr/tree/1361e190b7f46df996f00ea1b540ecf66785127f), [DexNom](https://github.com/DexNom/VIctory/tree/64bd58f5fb794216894bc0e33e03a3a64d19c347) | VItoriaBr’s `e40fba4` is another port of the capped-reserve patch imported above. Its `85615ad` second-plane composition overlaps the baseline’s default composition. Its `a51c1ea` mask-restoration patch and DexNom’s `76749ce` implement the same separate SSA pattern. Their scope includes the Radeon translator and the available recovery evidence is an archive translation. |
| fxceless heap ownership fix. [fxceless PR #1324](https://github.com/fxceless/victory-fork/pull/1324) | The donor reproduced a stale-release problem on WARP. Its heap identity changes span descriptor allocation and draw-state tracking and fail to apply against this branch’s versions of those APIs. |
| Kelvaroq Wave64 and descriptor changes. [Pinned patch series](https://github.com/stellarr-studio/kelvaroq/tree/e062a82e04853bbcae281d166980e0a9b65c975a/native/patches) | Kelvaroq applies the series to fxceless `f85101b`. `0033` conflicts in lowering. `0052` ties paired Wave32 shaders to the donor’s emitter and pipeline host. The `0049` and `0060` bundles also span different runtime APIs. |
| Oversized mesh outputs, Texbirds and eIBart0. [Belobug PR #138](https://github.com/Belobug/VIctory-Belobug-AMD/pull/138) | The donor reports its main-branch port recovered 71 mesh translations and character faces on RX 9070 XT. The exact diff conflicts in the pinned branch’s shader emitter and host pipeline. Porting the mesh and pixel shader link would exceed minimal glue. |

## Compatibility gaps

- Heap-relative bindless reads can still reach slots beyond the title descriptors that fit in place. Safe remapping or typed guards across this branch’s bindings and shader ABI need a separate port. [Descriptor discussion](https://github.com/Marvisak/VIctory/issues/747)
- The synchronized loop fix covers eligible single-wave compute groups on Wave32 hosts. General divergent control flow and the native Wave64 branch-mask refusal reproduced on the Radeon iGPU remain open. [Loop limitations](https://github.com/Belobug/VIctory-Belobug-AMD/pull/366), [branch-mask donor](https://github.com/Belobug/VIctory-Belobug-AMD/pull/345)
- Mesh outputs that exceed D3D12’s 32 KiB budget still need a compatible mesh/pixel link path. The source report observed garbled characters on an RTX 5070 Laptop. Desktop retail behavior remains to be checked. [Belobug issue #125](https://github.com/Belobug/VIctory-Belobug-AMD/issues/125)

## Upstream documentation

<details>
<summary>Belobug’s original README at the pinned revision</summary>

# VIctory on an RX 7900 XTX

A fork of [VIctory](https://github.com/Marvisak/VIctory), a compatibility layer
that runs an Xbox Series title natively on Windows from files the user supplies.

## The goal

**A stable 30 fps or more without ray tracing, then as high a frame rate and as
low frame times as the PC allows**, on an AMD RX 7900 XTX first and on PCs with
32 GB (later 16 GB) of RAM.

The way there is **porting, not emulating**: patching the title's renderer so it
behaves the way a PC renderer does, rather than having the layer watch and copy
the console's shared memory every frame. Shaders are translated ahead of time
into a versioned store that is kept between builds. Ray tracing stays optional
and off by default. The picture comes after the frame rate: texture pop-in and
lighting that is too bright or too dark when walking indoors are next in line.

**This repository contains no game files, code or assets.** You need your own
copy of the game. It is not affiliated with or endorsed by the game's developer
or publisher.

## Where it stands (2026-11-26)

The measurements below come from one Windows 11 PC with an Intel i9-12900K,
64 GB of system RAM and a Radeon RX 7900 XTX (24 GB of dedicated video memory).
Ray tracing was off for these runs.

| | |
|---|---|
| City street, Series S profile, renderer patches 1, 2, 5, 6 and 7 on | about 45 ms a frame (about 22 fps, 44.9 ms in run merge-5) |
| Same, patches 1, 2, 5 and 6 | 45.6 ms, p95 47 ms |
| Same, patches 1, 2 and 5 | 47.3 ms (about 21 fps), p95 49 ms |
| Series X profile, patches 1+2 | about 53 ms (about 19 fps) |
| Same scene before the patches | 57-75 ms (13-17 fps) |
| Shaders the title uses that translate | all of them (25,790 of 25,985; the rest are unused geometry shaders) |
| Memory | about 30 GB of private memory, 10.5 GB of VRAM: **64 GB of RAM recommended** for now |

It started at 280 ms a frame in the same scenes three days earlier. What sets
the frame now is the one thread that submits the title's work to the card
(about 40 ms); the card itself needs about 36 ms and now waits about 7 ms a frame with nothing queued.

### Reports from other GPUs

| GPU and build | What the contributor observed |
|---|---|
| Radeon RX 9060 XT (16 GB), `cd8dfc0` with local changes | In the same Series X interior at 1080p, the game averaged 19.8 fps with ray tracing off and 15.6 fps with it on. Indirect lighting showed white patches and dark smears. [Run settings and results](https://github.com/Belobug/VIctory-Belobug-AMD/issues/313) |
| GeForce RTX 3090 (24 GB), `a-publier@cd8dfc0` | The prison rendered. Two warm Series S runs averaged 545 and 532 ms per frame. The contributor reported 10.0 and 10.2 GB of VRAM in those runs. [Run settings and results](https://github.com/Belobug/VIctory-Belobug-AMD/issues/333) |
| Intel Arc B580 (12 GB), revision unreported | Startup and initial menu navigation worked. Further menu interaction slowed to about 0.7 fps before D3D12 device removal. [Crash report](https://github.com/Belobug/VIctory-Belobug-AMD/issues/98) |

The 10.5 GB of VRAM in the RX 7900 XTX table is the usage reported for that run.
Memory use and frame time depend on the scene and settings. The [ray-tracing notes](docs/fork-ray-tracing.md)
describe which lighting paths run and the remaining rendering issues.

### Renderer patches (off by default while they are tested)

| Switch | What it does |
|---|---|
| `VICTORY_PORT_CONSTANTS=1` | The title's 174 MB constants heap is read by the card where the title writes it: no uploads, no fills |
| `VICTORY_PORT_DESCRIPTORS=1` | The title's descriptor-table stores are patched (byte-checked) to report the slots they write, instead of a scan of ~600,000 slots a frame |
| `VICTORY_PORT_CBV=1` | Translated shaders read constant buffers in place as raw bytes (~22,000 fills a frame to none; -4.7 ms) |
| `VICTORY_PORT_GUARD=1` | The write guards' lock is shared by faults and refreshes, with a claim per 64 KiB block; the submission's planning runs beside it (title threads' waits 29 ms to ~2 ms a frame; -2 ms) |
| `VICTORY_PORT_JOURNAL=1` | The heaps' records are asked for at the title's submit, and the journal's notes are split into 64 parts with locks of their own (the submit thread's waits ~5 ms + 1 ms to ~0.3 ms; -1 to -3 ms) |

### Other switches

| Switch | What it does |
|---|---|
| `--console series-s` | The Series S profile: 1080p output, about 5 ms faster, a softer picture |
| `VICTORY_OUTPUT=1920x1080` | The title renders to that output size itself (Series X profile; about 16 ms less card time) |
| `VICTORY_FRAME_INTERVAL_MS=16.667` | Lets the title go above 30 fps (menus and videos reach 60; the game's clock stays right) |
| `VICTORY_AMBIENT_SETTLE` | How fast light settles after walking indoors (on by default, `250,3`; `0` turns it off) |
| `VICTORY_TEXTURE_STORE=1` | Converted textures kept between runs (about 8 ms a frame for now: off) |
| `VICTORY_KBM_GTA=1` | Optional GTA V PC-style keyboard and mouse mapping on the stand-in pad: `F2` or `` ` `` (`~`) toggles mouse-look and auto-hides the cursor while locked (`Esc`/`P`/focus loss unlocks and restores it), LMB/RMB = RT/LT, MMB/`C` = R3, `Shift` = A, `Space` = X, `R` = B, `F` = Y, `Tab` = LB, `Q` = RB, `Ctrl` = L3, `E` = D-pad Right, `V` = View. Tune with `VICTORY_MOUSE_SENS` (default `1.0`) and `VICTORY_MOUSE_INVERT_Y=1` (or pass via `scripts\play.ps1 -More @{VICTORY_KBM_GTA='1'}`) |

What is known to be wrong:

- Trees, signs and far objects flicker in the distance in some places.
- Lighting is too bright or too dark after walking indoors, until the title has
  run for a while; rooms with little direct light are too dark without `-RayTracing`.
- Textures of what just came into view appear late while the title streams.
- The first run of a new build stutters while the driver makes pipelines
  (`victory warm` makes them ahead).
- Recording a frame in the world makes the title's own watchdog end it.

## How to run it

You need Windows 10/11 x64, a CPU with AVX2, a Direct3D 12 GPU with shader
model 6.6 and mesh shaders, Rust (the toolchain is pinned), the MSVC build
tools, and your own dump of the title.

```powershell
git clone https://github.com/avatarmanz/VIctory-7900XTX
cd VIctory-7900XTX
copy victory.example.toml victory.toml      # then set your paths, outside this folder
cargo build --locked --profile diag -p victory --features diagnostics
scripts\play.ps1 -Exe <path of your copy's executable>
```

The launcher requests light device removal diagnostics (DRED) by default with
`-Build diag`. DRED requires `--features diagnostics`. For a release binary
built with that feature, request DRED with
`-Build release -More @{VICTORY_D3D12_DRED='light'}`.

`scripts\play.ps1 -Safe` leaves out the newest, faster ways; try it first when
something is wrong. `-RayTracing` turns the title's ray-traced indirect light on.

For PC FSR 4.1.2, set `[paths].fidelityfx_dir` in `victory.toml` or
`VICTORY_FIDELITYFX_DIR` to the folder containing AMD's signed
`amd_fidelityfx_upscaler_dx12.dll`. Use the DirectX 12 upscaler in
`Kits/FidelityFX/signedbin`. [AMD FSR SDK 2.4.0](https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK/tree/v2.4.0/Kits/FidelityFX/signedbin)

`scripts\play.ps1 -Exe <path> -Fsr4K quality` selects 2560×1440 input for
the retail 1.0.2.1 Series X 4K SDR profile. `balanced` selects 2258×1270
and `performance` selects 1920×1080. All three output 3840×2160. With
`victory run` directly, set `VICTORY_FSR_4K` to the preset name.
AMD's SDK offers FSR 4.1.2 for some cards only. Where it does not, the
newest FSR 3.1 it offers is used in its place and the log says so; its
picture differs from FSR 4's. Where it offers neither, the run stops
before the title starts and names the versions that were offered.
The log reports the provider at context creation and the actual input and
output sizes at the first successful temporal dispatch. The title profile
and PC FidelityFX bridge were ported from [DexNom/VIctory](https://github.com/DexNom/VIctory/commit/ab34a8b21477c94da2a6cf07249138e8bc84602e).

On a hybrid-graphics PC, the runtime and offline pipeline checks prefer a
high-performance adapter with Shader Model 6.6. The runtime logs its adapter
name, PCI vendor and device IDs, and dedicated VRAM. Set
`VICTORY_D3D12_ADAPTER` to a zero-based DXGI index or part of the adapter's
name to select it explicitly. With the launcher, pass
`-More @{VICTORY_D3D12_ADAPTER='NVIDIA'}`.

Before the first run, ahead of time (both can be stopped and go on later):

```powershell
target\diag\victory.exe shaders <the shader archives>   # every shader translated, minutes
target\diag\victory.exe warm --used                     # pipelines earlier runs drew with, for a new driver or build
```

Commands, debugging switches and upstream's own description:
[docs/README-fxceless.md](docs/README-fxceless.md). Every `VICTORY_...` switch is
described in a comment where it is read.

## What is different here

In short; the reasons and the measurements are in [docs/findings.md](docs/findings.md).

- **List calls are written down, not carried out, on the title's threads**
  (`VICTORY_THIN`): eight threads of ours translate them.
- **Submissions are prepared by a thread of ours and joined**
  (`VICTORY_ASYNC_SUBMIT`, `VICTORY_JOIN_PARTS`), with what can be spread over
  threads spread: refreshes, compares (AVX2), descriptor records, landings.
- **What shaders wrote comes back beside the write guards**, by the dwords that
  changed, on a thread of its own (`VICTORY_RELAY`, `VICTORY_LAND`).
- **Buffer fills that need bytes still on the card are done on the card**
  (`VICTORY_CARD_STEPS`) instead of waiting for it.
- **The title's writes are caught more cheaply**: one view opened at a fault,
  blocks that rest, places in the title's code that look before they write.
- **Textures go up on a copy queue of their own** and a slot shows one only after
  its upload ran, which also ended a device loss that came every 15 minutes or so.
- **Boats float**: textures that shaders write and the title's CPU reads go back
  to title memory (from fxceless and Geckomacabre, on by default here).
- **Frame record and replay** with per-pass times and shader variants compared
  turn by turn ([docs/replay.md](docs/replay.md)).

## Credits

This is a tree of other people's work with more on top. The history is kept, and
commits taken from elsewhere name the commit and its author.

- **[VIctory](https://github.com/Marvisak/VIctory)** by Marvisak and its
  contributors: the loader, the runtime services, the Direct3D 12 back end and
  the shader recompiler.
- **[fxceless/victory-fork](https://github.com/fxceless/victory-fork)**: the fork
  this one started from, and the people whose pull requests it carries.
- **[DexNom/VIctory](https://github.com/DexNom/VIctory)**: ray tracing fixes,
  cube gradients, sparse textures.
- **[Belobug/VIctory-Belobug-AMD](https://github.com/Belobug/VIctory-Belobug-AMD)**:
  the ambient light idea, pipeline workers, and work by romainhedouin.
- **[FQY55kai/ZJOU-VICTORY](https://github.com/FQY55kai/ZJOU-VICTORY)**.
- **[myokex/victory-amd](https://github.com/myokex/victory-amd)**: the guard over
  every view of a page (with Spexivus, joroyt and tightfall), and Joshua Odom's
  reads into title memory.
- **Geckomacabre** and **cxy0072019-sketch**: the texture read-back.
- **[XCL](https://github.com/ivall/xcl)** (GPL-2.0-only) was read for how the same
  title behaves under another layer; no code of it is here.
- This fork's changes were made with [Claude Code](https://claude.com/claude-code).

## License

GPL-3.0-or-later, as VIctory. See [LICENSE](LICENSE); the one file that is not
covered is named in [docs/asset-policy.md](docs/asset-policy.md).

</details>
