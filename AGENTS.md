# AGENTS.md — rynx-thermalcore

## What this is

Android thermal management daemon written in Rust. Targets `aarch64-linux-android` (set in `.cargo/config.toml`). Single crate, no workspace.

## Build

Use the `compile-rust` wrapper script (Android arm64-v8a / API 26 via NDK at `$ANDROID_NDK_ROOT` or `/home/ryanistr/android-ndk-r27c`). It builds, strips, optionally UPX-compresses, and copies the binary to the repo root as `rynx-thermalcore`.

```sh
compile-rust -a          # Android build (uses `.cargo/config.toml` target)
compile-rust -a -npx     # Android build, skip UPX
compile-rust             # native build (same wrapper, no UPX)
```

The wrapper runs `cargo build --release --target aarch64-linux-android`. To build for host (development/linting) directly without the wrapper:

```sh
cargo build --release --target $(rustc -vV | grep host | awk '{print $2}')
```

Note: building the bin for a non-Android host requires `-llog`, which is Android-only; the standalone host build fails at link time unless you provide a stub. Prefer the wrapper for anything you intend to run on-device.

## Simulator mode

Enable with `--features simulator`. Run via:

```sh
rynx-thermalcore --simulate <trace_file.json>
```

## Key facts

- Rust **2024 edition** — uses latest language features.
- Release profile: LTO enabled, single codegen unit, symbols stripped.
- No tests, no CI, no linter/formatter config beyond rustfmt defaults.
- Dependencies include `inotify`, `nix`, `signal-hook`, `sysinfo`, `serde`/`bincode` for data persistence.
- The `src/simulator.rs` module is gated behind the `simulator` feature and only compiled when that feature is active.

## Actuation / MTK thermals

This daemon **primarily targets MTK**, complementing the stock thermal stack (thermalloadalgod + thermal HAL + stargazing governor + mtk-dvfsrc) rather than replacing it. It also has **generic fallbacks for non-MTK devices** (mainline cpufreq/Processor cooling, Qualcomm tsens + BCL), so it degrades cleanly where `/sys/kernel/helio-dvfsrc` doesn't exist.

- Actuator mode is selected via `persist.sys.rynx.actuator` (`dvfsrc` | `soft`). Empty/unset triggers **auto-detect on first run**: probe `/sys/kernel/helio-dvfsrc` (OPP table present) → DVFSRC; else soft-safe cooling devices → SOFT; else generic fallback. Logs each choice.
- DVFSRC actuation = write `dvfsrc_req_vcore_opp` (coordinated SoC OPP, CPU+GPU+DDR together). Written only on OPP change to avoid sysfs churn.
- SOFT mode writes only soft-safe cooling devices (`thermal-devfreq`, `cpu_adaptive`, gpu/vcore/devfreq); falls back to broad devices (`cpuNN` etc.) only if no soft devices exist.
- `CoolingDevice::load_device` rejects MTK zombie sensors (negative temo sentinels) and non-performance actors (backlight/cam/shutdown/sysrst). MTK thermal zones read `-127000` when a sensor is absent — always reject `temp <= 0` when reading `/sys/class/thermal/thermal_zone*/temp`.
- Unit mismatch gotcha: `power_supply/battery/temp` is **deci-degrees** (310 = 31.0°C, matching control targets like `TARGET_TEMP_IDLE=360`), but `thermal_zone*/temp` is **millidegrees** (31000 = 31.0°C). Always convert zone temps (`/100`) before blending with battery temp — an earlier bug blew up the control temp to ~2146°C by mixing them raw.
- On MT6893 (X6739): ~137 cooling devices, 34× `cpuNN` hotplug actors, real levers are `thermal-devfreq-0` (GPU) + `cpu_adaptive_*` + dvfsrc. `mtktscpu` (SoC zone) runs ~11°C hotter than battery — the control temp blends this in (`SOC_TEMP_BLEND`).
- Learning/persistence: `ThermalAI` (PID gains + event history) serialized with postcard to `/data/vendor/rynx_thermal_data/learning.dat` (atomic tmp+rename). Gains are clamped on load.
- On-device tooling reports no bait (`/proc/mtkthermal` absent); real MTK sensors are the `thermal_zone*` nodes; dvfsrc debug is under `/sys/kernel/helio-dvfsrc`.
