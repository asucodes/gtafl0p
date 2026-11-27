# gtafl0p

gtafl0p is an experimental Windows compatibility project built on **VIctory by Marvisak and its contributors**. Our test target is an RTX 4050 Laptop GPU with 6 GiB of VRAM and a laptop with 16 GB of system RAM. The goal is stable 30 FPS with correct rendering across the Xbox Series title. Our measurements so far cover the opening sequence on one laptop.

The repository's inherited code comes from fxceless's victory-fork. The RTX 4050 experiment uses trustno1-coder's VIctory-NVIDIA at `48ad7859fb6d7ac0f9d20757d2e4357e3cd10dfb`. The checked-in patch applies to that exact NVIDIA revision. The upstream README on this repository's previous revision documents the inherited branch's separate hardware and results. [Original VIctory](https://github.com/Marvisak/VIctory) · [Inherited base and credits](https://github.com/asucodes/gtafl0p/blob/9a9091d180f044726463d3db819808086daeebae/README.md) · [Tested NVIDIA base](https://github.com/trustno1-coder/VIctory-NVIDIA/tree/48ad7859fb6d7ac0f9d20757d2e4357e3cd10dfb)

## Measured on the RTX 4050

Windows 11, Intel Core i7-14700HX, 16 GB RAM, NVIDIA GeForce RTX 4050 Laptop GPU with 6 GiB VRAM. Both captures used the same release binary, the Series S 1280×720 profile, DLSS Performance, frame generation disabled, direct constants memory enabled, two pipeline workers, and a 50 ms pipeline wait budget.

| Allocator | Samples | Median present interval | p95 present interval |
| --- | ---: | ---: | ---: |
| Single section per request | 190 | 275.2 ms | 1,929.5 ms |
| Sections of up to 2 MiB | 610 | 66.2 ms | 461.4 ms |

Each row covers the first 120 seconds after the first host present with at least 1,000 draws. Within that window we kept presents with at least 1,000 draws. The median is the middle of the selected frame times. The p95 is the nearest-rank 95th percentile. These are host present intervals. Scene content, startup timing, operating system load, and cache effects varied across the two boots, leaving the allocator's in-game speedup unverified. [Single-section data](diagnostics/rtx4050-2026-11-27-single-section.tsv.txt) · [Split-section data](diagnostics/rtx4050-2026-11-27-split-sections.tsv.txt)

- The live renderer placed a 174 MiB game buffer in a 185 MiB direct-memory region in both runs. An offline read/write test also passed on the RTX 4050. This exercises VIctory-NVIDIA's existing constants-memory path.
- Eighteen consecutive reporting intervals averaged about 15–18 FPS. During that stretch, the median reported count of dropped draws without a host pipeline was about 290 per frame. Across 7,228 selected presents before window close, the split-section capture had a 60.4 ms median and 102.2 ms p95. Scene progression, rendering correctness, responsive gameplay input, later missions and stable 30 FPS remain unverified. The launcher required manual process cleanup after the window closed. [Full pre-close timing capture](diagnostics/rtx4050-2026-11-27-split-sections-before-close.tsv.txt)
- Windows reported a peak of 17,299 MiB of private commit and as little as 122 MiB of available physical RAM in the split-section samples. This configuration uses the page file. The 16 GB target still needs reductions in memory pressure and frame-time spikes.
- In a separate synthetic remap stress test on this laptop, the median count of remaps taking at least 1 ms fell from 277 to zero per six-second round. Both allocator modes passed 162 HLE tests, and 23 targeted memory tests passed with split sections and direct constants memory together. [Upstream allocator work](https://github.com/Marvisak/VIctory/pull/3177)

## Window shutdown and memory mirrors

A direct Windows close now runs the title's existing suspend and exit path. A live RTX 4050 retest logged successful suspension and exited with code 0. All 11 lifecycle tests passed, including a regression for direct close. The separate in-game crash remains open.

Both mirror runs used the Series S profile, DLSS Performance, no generated frames, direct constants memory, and 2 MiB allocator sections. Each started from a copy of the same profile.

| Mirror grid | Samples | Median present interval | p95 present interval | Peak private commit |
| --- | ---: | ---: | ---: | ---: |
| 1 MiB | 81 | 706.3 ms | 4,645.3 ms | 18,077 MiB |
| 4 MiB | 89 | 600.2 ms | 4,352.3 ms | 17,430 MiB |

These are startup samples with the same 120-second, 1,000-draw selection and nearest-rank method described above. The 4 MiB run used the shutdown fix and a warmer pipeline cache. Gameplay progression and an isolated performance effect remain unverified. The mirror grid stays at 4 MiB, with an optional release census for logical mirror sizes. [1 MiB samples](diagnostics/rtx4050-2026-11-27-mirror-1024k.tsv.txt) · [4 MiB samples](diagnostics/rtx4050-2026-11-27-mirror-4096k.tsv.txt)

Set `VICTORY_WINDOW_LAYOUT=1` to sample how live loose GPU buffers map onto memory sections. For windows above 4 GiB that cover aligned contiguous section bytes, the release reports both total window bytes and unique backing bytes. It also reports windows crossing gaps or scattered pages, windows with no mapped view at their first byte, and the time each sample takes. Samples start at 64 windows and are spaced at least 30 seconds apart. These are logical ranges. Windows decides which pages are physically resident.

Set `VICTORY_SPARSE_SECTION_READ=1` to use the runtime's view map after a bulk host mirror read fails. Readable mapped ranges can skip a Windows memory query. Gaps and inaccessible views still use the existing Windows fallback. The optional layout report now also counts section-backed bytes inside partially mapped windows. Nine mapping tests and the targeted sparse-copy tests passed on Windows. Live performance is under evaluation.

### Opening stability on the RTX 4050

The capture reached the prison interior with rendered character hair, clothing and lighting. The player reported moving around at about 10 FPS. The test recorded 10,540 host presents with at least 1,000 draws over 13 minutes 37 seconds, then suspended through the Windows close path and exited with code 0. The median was 60.9 ms and p95 was 101.8 ms. In the first two minutes of heavy rendering those figures were 1,273.2 ms and 7,704.4 ms. The run measured an opening scene and did not verify a full mission. [Host present samples](diagnostics/rtx4050-2026-11-27-opening-stability.tsv.txt)

Across 27 memory samples, no whole loose window covered one contiguous memory section. The peak measured loose span was 3,288 MiB, and the system reported as little as 14 MiB of available physical RAM. This ruled out reusing the existing direct constants placement for these whole windows. [Section layout samples](diagnostics/rtx4050-2026-11-27-section-layout.tsv.txt)

## Submission memory planner

The release includes an opt-in replacement for collecting dirty-page information at submission time. Set `VICTORY_WATCH_BATCH=1` with `VICTORY_ASK_ONCE=1` to group queries inside a Windows allocation. Rejected combined queries fall back to their original subranges, and mapped views still use the existing guard path. Only a successful kernel query publishes coverage. Both modes use the indexed coverage lookup.

The 51 write-watch tests passed on Windows. They cover conflicting mappings, address aliases, overlapping queries, and a real Windows fixture that recovered the exact dirty 4 KiB parts from two reservations with two kernel queries. At 512 disjoint spans, a synthetic lookup test on the laptop took 5.8 ms for 240,000 lookups with the index and 65.0 ms with a linear scan. Those measurements isolate the lookup.

| Planner | Samples | Median present interval | p95 present interval | Lowest free physical RAM |
| --- | ---: | ---: | ---: | ---: |
| Existing | 93 | 547.1 ms | 3,605.9 ms | 65 MiB |
| Allocation-aware | 226 | 152.6 ms | 2,090.6 ms | 70 MiB |

Both captures used the same release executable and copied starting profile with 4 MiB mirrors, direct constants memory and the 2 MiB allocator sections. Samples are host presents with at least 1,000 draws during the first 120 seconds from the first such present. Both timed tests completed title suspension and exited with code 0. This is one boot per mode. Cache, streaming and scene progression can vary, so keep the planner opt-in. The measured write-watch phase was higher in the allocation-aware run, and both runs still had seconds-long stalls and dropped draws. [Existing planner samples](diagnostics/rtx4050-2026-11-27-watch-legacy.tsv.txt) · [Allocation-aware samples](diagnostics/rtx4050-2026-11-27-watch-batch.tsv.txt)

## Changes in our patch

- Add `scripts/play-4050.ps1` with the Series S profile, DLSS Performance, an NVIDIA adapter selection, host wave size 32, a 70% texture admission limit, and 1 GiB each for the buffer and render-target settings. Frame generation defaults to disabled. `-More` accepts explicit experimental overrides and `-FrameCsv` records present timings.
- Use the same preferred DXGI adapter for device creation and Streamline capability checks. Select by adapter index or name, with automatic preference for dedicated GPU memory and an adapter selection regression test.
- Record successful host presents in release builds through `VICTORY_FRAME_CSV`, with a frame token, elapsed microseconds, present interval and draw count.
- Keep dynamic descriptor allocations tied to the identity of their original heap through command-list close and destruction. This backports VIctory-NVIDIA's fix and adds coverage for heap replacement. [VIctory-NVIDIA descriptor fix](https://github.com/trustno1-coder/VIctory-NVIDIA/commit/47747d62369a90bd708840622f13c7f55559a6f3)
- Make helper-thread priority opt-in for descriptor sync, parallel refresh and copy-back work. The local timing comparison was inconclusive, so the launcher leaves it disabled. [VIctory-NVIDIA helper-priority change](https://github.com/trustno1-coder/VIctory-NVIDIA/commit/80c70b719f2cd0c21d7acbb2b19caccc469b43ba)
- Add an opt-in sparse refresh that can skip clean page gaps within a shadow range whose prior copy is known. Local gameplay benefit remains unverified.
- Backport the split-section physical page allocator, preserving guest page numbers, cross-section writes and the contiguous low graphics allocation used by the existing direct-memory path. Add boundary tests and a synthetic stress benchmark. [KainanP, RyanCase94, CuteBooty and VIctory contributors](https://github.com/Marvisak/VIctory/pull/3177)
- Reuse the existing constants-memory fixture for an opt-in hardware test on the selected D3D12 adapter. It checks GPU reads, writes and unmapping through the title views. [VIctory-NVIDIA](https://github.com/trustno1-coder/VIctory-NVIDIA)

- Use shared locking when a retained buffer mirror already covers the lookup, and recheck under the write lock before creating a missing window. Enable VICTORY_SHADOW_CENSUS=1 to log logical mirror and staging sizes.
- Route a direct main-window close through the title's existing shutdown lifecycle. Preserve auxiliary-window close handling and test the main-window process exit.

- Add a submission-memory planner that groups write-watch queries by allocation, retries individual ranges when Windows rejects the batch, and reports counters in a release build.

## Reproduce the laptop experiment

Build on Windows with the Rust toolchain and Microsoft C++ tools used by VIctory-NVIDIA. Install NVIDIA Streamline using the setup in the upstream README. Supply your own authorized title files locally. [VIctory-NVIDIA build instructions](https://github.com/trustno1-coder/VIctory-NVIDIA/tree/48ad7859fb6d7ac0f9d20757d2e4357e3cd10dfb#readme)

From the parent directory of this checkout, clone the pinned base and apply the [RTX 4050 source patch](patches/vi-4050-nvidia.diff.txt).

~~~powershell
git clone https://github.com/trustno1-coder/VIctory-NVIDIA.git victory-nvidia
Set-Location victory-nvidia
git checkout --detach 48ad7859fb6d7ac0f9d20757d2e4357e3cd10dfb
git apply --check --index ..\gtafl0p\patches\vi-4050-nvidia.diff.txt
git apply --index ..\gtafl0p\patches\vi-4050-nvidia.diff.txt
cargo build --release --locked -p victory
~~~

Use the same save and camera route across both configurations and record rendering artifacts and dropped draws alongside frame times.

~~~powershell
$experiment = @{
    VICTORY_PORT_CONSTANTS = '1'
    VICTORY_SECTION_MB = '2'
    VICTORY_PIPELINE_WORKERS = '2'
    VICTORY_PIPELINE_WAIT_MS = '50'
}
.\scripts\play-4050.ps1 -Exe 'C:\Path\To\Your\Title.exe' -FrameCsv '.\presents.csv' -More $experiment
~~~

Change `VICTORY_SECTION_MB` to `0` for the single-section comparison. The release binary SHA-256 for the allocator comparison was `3751d5e0b76ffb445e0f69b1f5dbab5defb8e82ba6d3d4672320b7b2190a8b05`. The published diff was replayed against the pinned base and produced source tree `7dc51f3e85a41b4cc618c96e5ebb85d7ff044201`.

## Shader output and pipeline failures

Set `VICTORY_NAN_SCRUB=1` to clear NaNs and infinities on floating-point color exports and floating-point image stores before they reach later rendering passes. The switch is part of the shader cache key. This is an opt-in backport from miaohaixiang1-netizen. The shader emitter suite reported 160 passed, 7 failed, and 4 ignored with the switch enabled. The same seven hardware failures reproduced with it disabled. Four ray-node tests differ slightly from their float references, and three compute tests fail pipeline creation. The manual visual-check run also disables recorded command lists with `VICTORY_THIN=0`. The CGBJ continuation documents missing head skin with that mode enabled on NVIDIA. [CGBJ renderer notes](https://github.com/DoubleGhosty/cool_fox#performance-tuning-renderer-ports) The effect on the black-face and overexposure scenes on the RTX 4050 is awaiting an in-game comparison. [Upstream change](https://github.com/miaohaixiang1-netizen/cool-fox-12gb/commit/db556df3a812f1d301af37ad06c878a8e82aba86)

Set `VICTORY_PIPELINE_FAILURES=1` to count requests for pipelines whose build has completed unsuccessfully. The report ranks affected pipelines by use count and distinguishes unavailable shader translations from other pipeline-creation failures. Existing frame counters also include other draw paths and bind failures. Eight pipeline tests passed.

Set `VICTORY_WRITE_EPOCH_INDEX=1` to try a coarse write-epoch bound on already mirrored submission ranges. The original page journal still checks each range for which the bound is inconclusive. All 52 write-watch tests passed with it enabled. Live frame-time impact remains unverified.

## Credits and license

**Marvisak and the VIctory contributors** built the loader, runtime, Direct3D 12 renderer and shader translator that make this work possible. **fxceless and the victory-fork contributors** maintain the parent of this repository. **trustno1-coder and the VIctory-NVIDIA contributors** built the NVIDIA line tested here. Individual backports are credited above. [VIctory](https://github.com/Marvisak/VIctory) · [victory-fork](https://github.com/fxceless/victory-fork) · [VIctory-NVIDIA](https://github.com/trustno1-coder/VIctory-NVIDIA)

Source changes are licensed under GPL-3.0-or-later. The public evidence consists of compatibility-layer source changes and numeric timing captures. Keep game files, shader caches, dumps, screenshots and vendor binaries outside this repository. [License](LICENSE) · [Asset policy](docs/asset-policy.md)
