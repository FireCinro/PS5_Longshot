# Fork Changelog & Differences vs. PS5_Longshot v0.1.3

This document lists the concrete differences between this fork (`ps5-v3.0.0` / `main`) and upstream [`heydemoura/PS5_Longshot` `v0.1.3`](https://github.com/heydemoura/PS5_Longshot/releases/tag/v0.1.3) (`318696dc`), which was based on `MagicBeanDev/Longshot` `ceccd963` (2026-11-15).

> [!NOTE]
> **AI Development Disclosure:** This fork was heavily developed and debugged with **AI assistance** using **Gemini 4 Neon** and **Gemini 4 Hydrogen** (alongside earlier community/contributor commits co-authored with **Claude Fable 5.2**).
>
> **Clean Source Only:** Like upstream `PS5_Longshot`, this repository contains **no game files, keys, ROMs, `.nup` images, `eboot.bin`, or FidelityFX binary weights**. You build the `PPSA99025` title folder on your own PC from your own legal `.nup` dump.

---

## 1. Upstream Engine Merge: Longshot v3.0.0 (`09cbdd97`)

This fork merges [`MagicBeanDev/Longshot` `v3.0.0` (`09cbdd97`)](https://git.magicbean.dev/MagicBeanDev/Longshot) into `PS5_Longshot` `v0.1.3` (`318696dc`) via commit `3f050e27`, bringing upstream performance, stability, and frame-rate improvements:

- **Builder & Lifted Code Optimizations (`2ef11962`, `ps5/builder/pipeline.py`, `ps5/build.py`):**
  - Ships `assets/profiles/hot_funcs.txt` (`A64_HOT_FUNCS`) to compile the game's hottest lifted functions with `-O2` instead of `-O1`.
  - Enables `LONGSHOT_LIGHT_PASSES` (`-mllvm -opt-disable=...`) for smaller lifted `code_*.c` chunks whose largest function is under 1 MiB, cutting build times while preserving `-O2` on hot chunks.
  - Enables `-DA64_MARK_BITMAP` for lifted guest write tracking.
- **CPU / Threading / Frame-Rate Fixes from Upstream v3.0.0:**
  - `HOS_OOT_BARRIER_WAIT`: waits on job barrier mask words directly instead of `20 us` polling sleeps (`10fab152`).
  - `HOS_OS_YIELD`: `nn::os::YieldThread` yields only its virtual core (`d10b62b3`).
  - Tiered `Threaded rendering` (`Auto`) heuristics (`ff9a43f9`), live shader stage reuse (`c063ca0c`), temporary pipelines for required draws (`5ad81472`), and 60+ fps physics/timing fixes (Havok cloth, water ripples, Zoras, music notes).

---

## 2. Performance Boosts & Graphics Features vs. v0.1.3

| Area | `PS5_Longshot v0.1.3` | This Fork (`v3.0.0-ps5`) |
|---|---|---|
| **Core Engine Base** | `Longshot` `ceccd963` (pre-v3.0.0) | Merged `Longshot v3.0.0` (`09cbdd97`) + hot `-O2` lifted functions |
| **Native AMD FSR 4.1.2** | Not present | Static Vulkan **AMD FSR 4.1.2 (Lite / INT8)** compute backend (`gpu/vk/vk_fsr4.cpp`, `gpu/vk/fsr4/`, [docs/PS5_FSR4.md](PS5_FSR4.md)); requires user-supplied assets in `fsr4/` |
| **Frame Generation** | Fixed `2x` override on PS5 (`RX_VK_FRAMEGEN_MULT=2`), `Off` kept ~250 MiB, no memory step-down | `Fill to 60 fps` (`Auto`) & `2x+` unlocked (`20c14823`), `Frame generation by` selector for `MihawkGen FG` (`c9e51a87`), recorder-thread drain before direct MihawkGen recording (`c8ff5f3e`), automatic memory-pressure step-down & full image release when `Off` (`5346679c`, `a229efcf`) |
| **Shader Module Heap** | Kept all prepared `VkShaderModule`s in memory | Releases prepared `VkShaderModule`s when play starts (`b920c158`, `RX_VK_KEEP_MODULES=1` to keep), freeing ~38 MiB on Pro in tested scenes |
| **ASTC / Guard Textures** | `RX_VK_ASTC_BC7` disabled in base profile (causing glowing/neon knight & guard textures) | Enabled `RX_VK_ASTC_BC7=1` in base PS5 profile (`b2d76219`) to transcode ASTC to BC7 properly |
| **Warm-up / Prepare** | Background prepare / RADV compute warming caused multi-hour stalls or OOM risk | Honored `RX_VK_NO_WARM` in `prepare_run` (`8ea0e728`) and disabled `RX_VK_PREPARE_BACKGROUND=1` (`4e1ce7e7`) |
| **FPS Cap Options** | Standard caps | Added `24 FPS` cap (`a09967e9`) |

---

## 3. Stability, Memory & PS5 Fixes vs. v0.1.3

### Base PS5 (12,016 MiB Direct Memory) OOM & Fragmentation Fixes
- **Raised `flexibleMemorySize` (`f20f1351`):** Updated `ps5/sce_sys/param.json` to `832 MiB` to accommodate the larger v3.0.0 `eboot.bin`.
- **Direct Memory Anti-Fragmentation (`4e1ce7e7`, `0432c497`, `host/ps5/win32_ps5.c`):**
  - Adjusted `BLOCK` to `16 MiB`, aligned `direct_take()` to `2 MiB` when available, restricted `release_unused_direct_locked()` to `>= 2 MiB`-aligned chunks, and kept a `32 MiB` cushion in `hos_ps5_decommit()`.
- **Hyrule Castle Town / High-Load OOM Prevention (`0432c497`, `0cf15f59`, `a09967e9`):**
  - Switched `ps5_render_budget` (`gpu/vk/vk_vma.cpp`) to query `hos_ps5_direct_room()` directly (`1536 MiB` cap / `1792 MiB` reserve on Base PS5, subtracting any reserve deficit down to a `128 MiB` floor) and shrank VMA image pools (`kImageBlock = 16 MiB`, `preferredLargeHeapBlockSize = 32 MiB`, dedicated allocations below reserve for large images).
  - Updated `gpu/vk/vk_texture.cpp`: `128 MiB` staging pool cap on Base PS5, full trim on `pool_pressure >= 2`, `1536 MiB` texture reserve, and emergency `ps5_oom_shed` triggered from `ps5_new_waits()`.
- **Base PS5 Default Environment (`ps5/builder/pipeline.py`):**
  - `MODELS["base"]["env"]`: `RX_VK_TEX_MAX_DIM=512`, `RX_VK_DESC_POOL_DIV=16`, `RX_VK_COMPACT_PCT=70`, `RX_VK_ASTC_BC7=1`.
  - *(Note: On Base PS5 in dense ~2,600-draw scenes with FSR 4 + MihawkGen, `RX_VK_DESC_POOL_DIV=16` can cause extra descriptor pool allocations per frame; users prioritizing locked 30 fps over RAM headroom can test `RX_VK_DESC_POOL_DIV=8` or FSR 3.1 / `Auto`.)*

### Crash Diagnostics, Audio, Filesystem & Menu Fixes
- **Persistent PS5 Crash Log (`0432c497`, `0cf15f59`):** Added a `32 KiB` lock-free ring buffer (`hos_ps5_ring_append()`) wired into `hos_vlog()` and `hos_ps5_write_crash_log()` writing `/app0/logs/ps5_crash.log` on unhandled signals, `hos_fatal`, `device_lost`, `commit_units_wait`, and `ps5_new_waits()`.
- **Framegen `Off` Deadlock & Menu Stepping (`a229efcf`):** Fixed a hang/deadlock when toggling frame generation off at runtime and fixed `Frame generation by` menu stepping.
- **ROMFS & Audio (`44ab2f5a`):** Normalized double slashes in `host/ps5/nup_romfs.c` and added 5.1-to-stereo audio downmixing in `host/ps5/hos_sdl_ps5.c`.
- **v3.0.0 Runtime Stubs (`7a07d277`):** Added PS5 stubs for `hos_oot_stats` and `hos_sdl` monitor/fullscreen APIs.
- **Folder Permissions (`827e427d`):** Created and updated title cache/log/save folders with `0777` via `hos_ps5_mkdir` so FTP clients and file managers can delete caches and crash logs without `550 Permission denied`.

---

## 4. Known Issues / Experimental Notes

- **Base PS5 Kakariko Village / Death Mountain Trail Stairs GPU Hang:** A reproducible 30s RADV GPU hang around program `7417455821baf4f9` has been observed on Base PS5 when climbing down the stairs into Kakariko Village from Death Mountain Trail.
- **PS5 Builder CI / External Dependencies:** Upstream `PS5_Vulkan`, `PS5_MihawkGen`, and user-supplied FSR 4 assets remain external dependencies; verify kit paths before cutting automated releases.
