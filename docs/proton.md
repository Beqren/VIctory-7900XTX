# Running on Linux through Proton

The layer is Windows code. It also runs on Linux under Wine: Proton (Wine plus
vkd3d-proton, Direct3D 12 on Vulkan) on an AMD card with Mesa's RADV driver.
This page says what that needs, how to build and run, and what goes wrong
there, with the cause and the switch for each fix.

**Measured on one machine only**: CachyOS (kernel 7.3), Radeon RX 9070 16 GB,
Mesa RADV 26.3.1, Proton Experimental 11.0-20261112 (its vkd3d-proton is
3.2.0, build 2576c0b4) with the patched wineserver (below), 2026-11-25/26;
some results also with vkd3d-proton master (71cae09, 2026-11-25) built from
source and placed beside `victory.exe` (`WINEDLLOVERRIDES` with
`d3d12,d3d12core=n`). Nothing here has been tried on another Linux machine,
another card or another driver.

The changes below are on by default only under Wine (ntdll exports
`wine_get_version`), each with a switch: `0` turns it off, `1` turns it on
on any host. That is a choice, not a finding: the compute dispatch wait is
for a cause that is not Wine-specific. None of them has been tried on
Windows or on other cards. The null-blob pipeline library is the one change
that is the same on every host.

## Requirements

- **Proton Experimental** (or a Wine of the 11 series with vkd3d-proton). Not
  GE-Proton11-12: its vkd3d-proton turns some translated shaders into invalid
  SPIR-V and RADV 26.2 dies on a null pointer at the first world pipeline (exit
  code 3).
- **For speed, a wineserver with the view tree** (next section). Stock works,
  about three times slower.
- **Microsoft's `d3dcompiler_47.dll`, build 10.0.26100**, in the prefix's
  `system32`, with the override `WINEDLLOVERRIDES=d3dcompiler_47=n`. The layer
  compiles its own helper shaders at run time with `D3DCompile`, some as
  `cs_5_1`. Wine's built-in `d3dcompiler_47` refuses some of them (for example
  `Method 'InterlockedMax' is not defined`), and so does the older Microsoft build
  winetricks installs (6.3.9600). Without them the world's indirect draws are
  dropped and the screen is mostly black.
- **cargo-xwin** and a Rust toolchain with the `x86_64-pc-windows-msvc` target,
  for the Windows build on Linux.
- A launcher that runs a Windows program with Proton outside Steam, such as
  umu-run (`GAMEID=0`, `PROTONPATH=<the Proton>`, `WINEPREFIX=<a prefix of its
  own>`).

## Building

    cargo xwin build --release -p victory --target x86_64-pc-windows-msvc

The binary is `target/x86_64-pc-windows-msvc/release/victory.exe`. A
diagnostics build is `--profile diag --features diagnostics`, under
`target/x86_64-pc-windows-msvc/diag/`.

The shader crate's tests run natively (`cargo test -p victory-shader --lib`).
`victory-gpu` and `victory-hle` are Windows-only: build their tests with
`cargo xwin test --no-run` and run the test executables under Proton.

## The patched wineserver

wineserver keeps a process's memory views in a linked list and walks it on
every view call: `map_view` checks the address against every view and
`unmap_view` searches from the head. The title keeps about 125,000 views and
the layer remaps 1,000 to 4,000 single 64 KiB pages a frame, and ntdll holds
its process-wide lock across each server call, so every `VirtualProtect`,
`VirtualQuery` and `GetWriteWatch` of every thread waits behind it. One page
remapped cost 23.6 us with 100 views alive and 552 us with 125,000.

The fix is a published wineserver patch: Wine bug 60868, CachyOS/wine-cachyos
pull request 50. It keeps an rb-tree of views by base address beside the list,
so each lookup is O(log n), and touches only `server/` (`file.h`, `mapping.c`,
`process.c`, `process.h`). Only `wineserver` differs from stock.

| views alive | stock, one page remapped | patched |
|---:|---:|---:|
| 10,000 | 113.6 us | 11.5 us |
| 50,000 | 249.8 us | 12.0 us |
| 125,000 | 520.9 us | 12.2 us |

To use it with Proton, build `server/wineserver` from Wine's source at exactly
the commit that Proton release's `wine` submodule names, with the patch applied,
and put it in a copy of that Proton (`files/bin/wineserver`). It must match
that Proton's own `ntdll.so`: the protocol version is checked at start, so
never put it into another Proton release, and rebuild it when Proton updates.
Give the copy a prefix of its own. A world frame of 2.9 to 3.2 s on stock fell
to about 1 s in the first runs, before the fixes below.

## Running

    WINEPREFIX=<prefix> PROTONPATH=<Proton> GAMEID=0 WINEDLLOVERRIDES=d3dcompiler_47=n \
      umu-run victory.exe run 'Z:\path\to\game_scarlett_master.exe' --execute

Paths given to the layer are Windows paths (`Z:` is the Linux root). Set
`VICTORY_GAME_DIR`, `VICTORY_CACHE_DIR` and `VICTORY_DATA_DIR` (or
`victory.toml`) to Windows paths too, and `VKD3D_SHADER_CACHE_PATH`, since
vkd3d-proton otherwise writes its cache into the working directory. Proton
does not pass a console program's output on: run it under `cmd /c` with the
output redirected to a file.

The launch settings of `scripts/play.ps1` apply on Linux as on Windows.
`VICTORY_KEYS` presses the pad's A on the start screen in every build, so an
unattended run gets past it (it does nothing unless set). On a vkd3d-proton
older than 71cae09, which includes the 3.2.0 of current Proton (below), add
`VICTORY_PIPELINE_WAIT_MS=0`.

## Problems seen under Proton

Each entry gives what was seen, the cause, and what to do about it.

### Pipelines were never kept between sessions

Seen: "no host pipeline library: pipelines are not kept" at every start; no
early list was written, so every session made its pipelines from nothing.

Cause: the layer asked for an empty pipeline library with an empty Rust slice,
which has a non-null address. vkd3d-proton returns `E_INVALIDARG` for a
non-null blob of length 0 (`libs/vkd3d/cache.c`,
`d3d12_pipeline_library_init`). The empty library is now asked for with a null
blob (as Marvisak/VIctory#347 by AugustoZFJ, and fxceless/victory-fork#885):
the first start logs "no host pipeline library yet; starting an empty one",
the next one "host pipeline library loaded" and starts the listed pipelines
early. This is the same on every host. Microsoft's page says an empty blob
gives an empty library, not what the pointer may be; it has not been tried on
Windows.

### Every frame waits 500 ms for six compute pipelines (vkd3d-proton 3.2.0)

Seen, with the vkd3d-proton of Proton Experimental 11.0-20261112 (3.2.0,
dxil-spirv 114e664): in the world 285 ms a frame; 55 ms with
`VICTORY_PIPELINE_WAIT_MS=0` (the same scene).

Cause: six compute pipelines are never made (shaders 17829ac71aab4e8d,
32ed072b9f845048, afeea48bbb9913ad, 559d9f0184e15fe3, a1b20bc47e3c9597,
e8c962358d862e3f; each hangs when built alone with `victory translate
--pipelines`). From reading dxil-spirv's source, not from a debugger: a
pre-pass that runs before its structurizer mishandles a loop with two back
edges (a `continue`) and recurses. The pipeline stays "being made", and every
frame that asks for one of them spends the whole pipeline-wait budget
(`VICTORY_PIPELINE_WAIT_MS`, 500 ms a frame) on it. dxil-spirv fixed this on
2026-11-24/25 (694cc67, "Only run fixup passes after structurizer has run",
and 3bc8276); with vkd3d-proton 71cae09, which has them, all six are made.

What to do on 3.2.0: `VICTORY_PIPELINE_WAIT_MS=0`, which avoids the wait for
draws (those whose pipeline is not ready are dropped instead), or a
vkd3d-proton with the dxil-spirv fix beside the executable. Compute dispatches
wait for each of the six once (below), at most half a second a frame: in one
cold start on 3.2.0 with `VICTORY_PIPELINE_WAIT_MS=0` the longest gap between
frames in the first 300 s was 1.05 s, and the heads had skin. A change that
made the translation give every loop one latch also made the six build on
3.2.0, but it changed the shader cache key for every Wine user and is not in
the tree.

### Heads without skin (a floating cap, eyes and teeth)

Seen with a cold driver shader cache (a fresh `MESA_SHADER_CACHE_DIR`), on
vkd3d-proton 3.2.0 and on 71cae09 alike; with a warm cache the heads had
skin.

Cause: the head blend-shape map's compute pipeline (61e54e90adccaf6a) was not
made yet at its one dispatch (made 1 to 21 s later, behind hundreds of
graphics pipelines). A dispatch whose pipeline is not ready is dropped, and
the title issues this one once, so its output is missing for the session. A
timing race, not a Wine one: the fork's issue #75 saw heads without skin on
Windows.

Fix, on under Wine (`VICTORY_COMPUTE_WAIT`: `0` off, `1` on any host): the
first dispatch of a compute pipeline not made yet waits for it, outside the
frame's 500 ms budget, and a worker is kept free to make it next:

- a frame's compute waits end 500 ms after its first one began, however many
  threads wait;
- a pipeline has one wait: once a dispatch has waited 100 ms for it in vain,
  its later dispatches wait as draws do (a pipeline that is never made holds
  one frame once);
- compute pipelines go before the requested graphics pipelines (after the
  passes over the screen), the newest first, and one a dispatch waits for
  goes before them;
- the last free worker (of four or more) takes only such a pipeline or a pass
  over the screen. The head map compiles in 1 to 3 ms; without the kept worker
  its dispatch waited 0.3 to 1.1 s on a cold start for a worker to come free
  (once past the 500 ms: no skin), with it the map was made 5 to 191 ms after
  its dispatch. That narrows the race; it does not close it (the kept worker
  can itself be busy).

The first form of this held each frame up to 10 s and each pipeline up to 10 s
of a worker's time: on a cold start every frame that dispatched pipelines not
made yet was held again (8 and 9 gaps of 1 to 10 s in the first 300 s here; 30
over 1 s and 11.3 s longest in a tester's run on an RX 7900 XT, issue #386).

Measured here (RX 9070, RADV 26.3.1, vkd3d-proton 71cae09, diagnostics
builds), cold starts (empty driver, vkd3d-proton, pipeline and translation
caches; the same checkpoint), the first 300 s after the first frame:

| compute wait | runs | frames/s | longest gap | gaps over 1 s | heads |
|---|---|---|---|---|---|
| off (`=0`) | 1 (+1 that crashed at 57 s) | 15.9 | 1.0 s | 1 | no skin |
| 10 s a frame (before) | 2 | 14.6, 14.5 | 10.2, 10.3 s | 8, 9 | skin |
| this | 2 (one merged with main 014d52d) | 16.6, 15.3 | 1.1, 1.1 s | 1, 2 | skin |

With a warm cache no compute dispatch waited (0 ms in 300 s; longest gap
0.96 s). Nine pipelines dispatched in at most three frames never ran with
this (the same nine with the wait off, none with the 10 s form): their
output, from the bursts of the first world frames, is lost; what it is for
is not known.

## Not known

- Long sessions on Linux with this tree have not been tested.
- vkd3d-proton's converter may fail the same way on programs not yet seen.
- The patched wineserver goes stale whenever Proton updates: rebuild it.
- Nothing here has been run on Windows.
