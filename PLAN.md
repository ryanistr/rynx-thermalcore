# PLAN.md — ML Thermal Mitigation: dataset-first roadmap

Goal: a thermal mitigation controller that delivers sustained performance when
demand exists and hard-caps heat when it doesn't, learned from real device
data instead of hand-tuned heuristics.

Status of this document: living plan. Stage gates are binding — no stage
starts until the previous gate passes.

---

## Why this direction (context)

The current "learning" in the daemon (`src/prediction.rs`) is an online linear
regression trained on raw, unscaled features (temp ~310 vs load ~0.5) with a
single learning rate, retrained every tick, with 50% authority over the
actuator (`src/monitor.rs:302`). That design cannot do anything but chase
noise; the observed lags are compounded by two cascaded smoothing filters
(EWMA on the sensor in `utils.rs:106`, `SMOOTHING_ALPHA=0.3` on every cooling
write in `cooling.rs:133`) plus a deadband that wipes the PID integral
(`policy_manager.rs:49`).

Conclusion: no model choice fixes a loop with no dataset and no benchmark.
Hence: collect first, model second, control third.

## ML route (decided)

- Stage 1 model: gradient-boosted trees (GBDT, shallow depth, early stopping
  on a chronological split) predicting ΔT over 5s/15s/45s horizons, trained
  OFF-device in Python, exported as flat threshold arrays embedded in the
  daemon (inference is a few hundred comparisons — no ONNX runtime).
  Conservative variant: quantile loss (p80), so we plan against the
  pessimistic forecast.
- Control layer: model predictive control (receding horizon) on top of the
  learned plant model, with the existing PID as fallback/baseline.
- RL: only as Stage 3, offline only (contextual bandit / offline RL on the
  logged dataset), deployed as frozen weights, always shadowed against MPC.
  No online exploration on hardware, ever.
- Not chosen: random forest as primary regressor (GBDT dominates on tabular
  regression with small data), online learning on device (source of the
  overfitting we're fixing).

---

## STAGE 0 — Dataset collection daemon (current stage)

### Objective

A standalone, zero-interaction collector that runs alongside normal device
usage for days, appending a complete, time-aligned record of the thermal /
power / performance state of the phone.

### Design decisions

- Separate binary, same crate: `src/bin/thermal-collector.rs` (second
  `[[bin]]` in Cargo.toml), reusing `src/lib.rs` modules. Never links the
  actuation path — it must be structurally incapable of writing cooling
  devices or DVFSRC. Observation only.
- Fixed cadence: 1 Hz monotonic + wall-clock timestamps (wall for
  correlating with user events/app usage, monotonic for Δt math). Event
  records (screen on/off, charger connect/disconnect, thermal runaway alert
  conditions) are logged as typed entries with the same clock.
- Append-only JSONL, one record per line, rotated at size cap (default
  64 MB) into timestamped files, bounded total retention (default 1 GB,
  prune oldest). JSONL = trivially parseable by pandas, gap-detectable line
  by line, survives a crash mid-write (last partial line = dropped, rest
  intact).
- No in-file derived labels. Horizon labels (ΔT over next 5/15/45s) are
  computed at training time from raw series — raw logs are re-labelable,
  precomputed labels are not.
- Actuator state is captured by reading sysfs (cooling device `cur_state`,
  `dvfsrc_req_vcore_opp`), so whatever mitigation daemon is running is
  reflected in the data without any IPC. This lets Stage 0 collect both
  baseline (stock stack only) and, later, our own daemon's behavior.

### Schema draft v0 (field groups; final names pinned after device probe)

- clock: `t_mono` (s), `t_wall` (unix s), `seq` (monotonic record counter)
- battery: temp (deci-°C, raw), `current_now` (µA, signed), `voltage_now`,
  `status`/charging flag
- zones: `thermal_zoneN` → {type, temp} for every zone reading >0 and
  ≤200000 (reuse the sentinel filter from `thermal_zones.rs:49`)
- cpu: aggregate + per-core busy% deltas from `/proc/stat` (reuse
  `src/cpu.rs` parsers), per-policy `scaling_cur_freq` + `stats/time_in_state`
  if exposed
- cooling: for each accepted cooling device (reuse `cooling.rs`
  whitelist/blacklist) → `cur_state`/`max_state`
- dvfsrc: current `dvfsrc_req_vcore_opp`
- context: screen on/off (reuse `context.rs` detectors), charger state
- events: `type=screen|charger|mode` entries interleaved in the same stream

### Node probe list (MUST be verified on device before schema freeze —
### these are hypotheses from code reading, not confirmed paths)

- [ ] `/proc/stat` per-policy freq: does this kernel expose
      `/sys/devices/system/cpu/cpufreq/policy*/stats/time_in_state`?
- [ ] GPU load/freq node (MTK: `/proc/gpufreq`/`/proc/mali*` variants differ
      per kernel — probe, don't assume)
- [ ] Is there a usable frame-time source? (`dumpsys SurfaceFinger --latency`
      needs shell/root; `SurfaceFlinger` frame stats, or GFXBench/Manhattan
      scripted runs as a reproducible perf probe). Performance labels are
      the single biggest risk to this plan — if no perf signal is available,
      perf is proxied by load+freq, which is weaker but usable.
- [ ] Do zone temps update at ≥1 Hz or are they latched? (affects useful
      sample rate)
- [ ] Logcat budget: can the collector read `logcat -d -b thermal` style
      entries for vendor-thermal events? (useful context, not required)

### Deliverables

1. `src/bin/thermal-collector.rs` — the collector binary
2. Schema pinned in this file after probe pass (`schema v0` → frozen)
3. Host-side `tools/audit_dataset.py`:
   - gap report (records/day, max gap, clock jumps)
   - unit sanity (temps in plausible range, no sentinel leakage)
   - coverage matrix (charging × screen × load buckets)
   - byte estimate vs 1 GB budget
4. Packaging: how it starts on boot and survives (user's init/rc or Magisk
   module — decide at implementation time; manual start is not acceptable)

### Gate 0 → 1

- ≥ 3 days of continuous collection across ≥ 3 distinct usage scenarios
  (idle overnight, gaming/heavy load, charging while in use)
- audit script passes: max gap < 60 s during uptime, no sentinel temps,
  coverage matrix has entries in every required bucket
- at least one pull via `adb` onto the host, schema parses in pandas/duckdb

---

## STAGE 1 — Shadow-mode plant model (no actuation)

- Feature/label set fixed from Stage 0 data; chronological train/val/test
  split (never random — time-series leakage hides overfitting)
- GBDT baseline predicts ΔT at 5/15/45 s; p50 and p80 quantiles
- Acceptance: beats persistence baseline (predict "no change") and beats a
  linear model on the same split; error reported in °C so it's meaningful
- Ship inference into the daemon in SHADOW mode: model predicts every tick
  and logs its forecast vs what actually happened, zero actuator authority
- Gate: shadow forecast error on-device ≈ offline test error (no
  train/serve skew), collected for ≥ 1 day

## STAGE 2 — MPC controller, PID as fallback

- Receding-horizon optimizer over candidate intensity trajectories,
  objective = overshoot beyond target + performance given up, constraint:
  never exceed Critical (480 deci-°C)
- `PREDICT_BIAS_WEIGHT` stays 0 until MPC beats PID on a replayed dataset
  from Stage 0 (offline replay is the free first test — no device risk)
- A/B on device: logged benchmark scenario, compare time-in-boost vs
  peak temp vs stock/PID
- Gate: MPC ≥ PID on sustained-performance metric at equal or lower peak
  temperature, measured, over ≥ 2 runs

## STAGE 3 — Offline RL (optional, only if MPC plateaus)

- Contextual bandit over the 24 DVFSRC OPP levels trained on logged
  Stage 0-2 outcomes; frozen weights; shadowed against MPC before any
  authority is granted

---

## Risks / open questions

- Performance signal availability (probe list) — biggest single risk; the
  plan degrades gracefully to load+freq proxy but that weakens labels.
- Storage/thermal budget of the collector itself: 1 Hz sampling must not
  itself measurably heat the device (audit: compare battery temp with
  collector on vs off during idle — if it moves the sensor, drop cadence).
- Dataset skew: user's usage defines the distribution; deliberate scripted
  scenarios (benchmarks, charging+gaming) must be part of the 3 days.
- Version/schema drift: the collector stamps `schema_version` in every
  record from day one so later analysis can filter cleanly.

## Explicit non-goals (for now)

- No changes to the actuation logic in `monitor.rs` during Stage 0.
- No thermal HAL integration (IThermal / ThermalService) — revisit only
  after control quality is proven.
- No online training on device, in any stage.
