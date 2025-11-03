## Python Look‑Ahead via Ring Buffer with CODESYS SoftMotion

Exported: 2025-11-03

## Components we’ll use

### controlOS components (explicit)
- **CODESYS project (`codesys/`) [controlOS]**: SoftMotion axes, tasks, PLC logic we extend with a ring‑buffer consumer.
- **Python shared lib (`root/local_src/python-shared/shared/`) [controlOS]**:
  - `shared/codesys_types/*` [controlOS]: IEC↔ctypes structs (`AppCmd/AppFbk`), to be extended for stream header/indices.
  - `shared/app/app.py` [controlOS]: async utilities (`poll`, `task_group`, timing helpers) reused by the Windows app.
  - `shared/log.py`, `shared/utils.py`, `shared/conf.py` [controlOS]: optional helpers.
- **Shared‑memory contract [controlOS pattern]**: Existing `cmd/fbk` mapping + added `codesys_stream` ring buffer.

### Non‑controlOS components
- **CODESYS Control RTE (Windows)**: Real‑time PLC engine executing cyclic tasks.
- **CODESYS SoftMotion**: Motion layer on top of CiA‑402; provides PLCopen MC_* function blocks, blending, cam/gear, limits.
- **Windows named IPC**: Named shared memory (section object) and semaphore (e.g., `Global\\codesys`, `Global\\codesys_stream`).
- **Python app (Windows)**: Your trajectory generator that fills the ring buffer and orchestrates modes via `AppCmd`.

## Data flow (high level)
- **Python → PLC (ring buffer)**: Python keeps 250–500 ms of look‑ahead at period Ts (e.g., 4 ms), writing per‑axis setpoints and advancing `producer_index`.
- **PLC → SoftMotion**: Every PLC cycle, PLC reads one sample using `consumer_index` and feeds SoftMotion’s external setpoint/segment interface; axes stay synchronized by using the same sample index.
- **PLC → Python feedback**: `consumer_index`, stream state (underrun/armed), and motion status in `AppFbk` so Python maintains backlog and adapts.

## Generation, look‑ahead, and blending (detailed)
- **When Python generates data (refill control)**
  - Track backlog samples: `B = producer_index - consumer_index` (mod capacity).
  - Define watermarks: `Blow` (e.g., 200 ms) and `Bhigh` (e.g., 400 ms) mapped to samples via Ts.
  - Refill when `B < Blow`: generate `k = Bhigh - B` samples and append; stop when `B ≥ Bhigh`.
- **PLC time is leading (look‑ahead principle)**
  - The PLC cyclic task (2–4 ms) is the master clock and advances `consumer_index` by 1 per cycle.
  - Python maintains backlog; jitter in Python does not affect motion as long as `B ≥ Blow`.
- **How SoftMotion uses the buffer (blending to new points)**
  - The consumer FB converts each sample to an external setpoint or short queued segment (position/velocity).
  - SoftMotion enforces limits and blends between successive samples/segments with its S‑curve planner.
  - Configure blend parameters (max vel/acc/jerk, corner tolerance) for continuity even with varying curvature.

## Capacity and data‑rate guidance
- **Per‑sample size (positions only)**: `8 bytes × num_axes`.
  - Example: 10 axes → 80 B/sample; pos+vel → 160 B; pos+vel+acc → 240 B.
- **Throughput at Ts = 2 ms**: `sample_size / 0.002`
  - 80 B ⇒ ~40 kB/s; 160 B ⇒ ~80 kB/s; 240 B ⇒ ~120 kB/s (for 10 axes).
- **Look‑ahead memory (0.5 s @ 2 ms ⇒ 250 samples)**
  - ~20–60 kB depending on contents (positions only vs pos+vel+acc), plus a small header.
- **Conclusion**: Shared memory copy and size are trivial on a PC; choose capacity for 0.25–2.0 s look‑ahead without concern.

## Interfaces and layout (concise)
- **Ring buffer header (`codesys_stream`)**:
  - `sample_period_ns: uint32`, `num_axes: uint16`, `capacity: uint32`.
  - `producer_index: uint32`, `consumer_index: uint32` (monotonic; wrap by modulo).
- **Samples**: `float64 samples[capacity][num_axes]` (optionally velocity/accel arrays alongside positions).
- **Cmd/Fbk** (existing) extended fields:
  - Cmd: `stream_arm`, `stream_reset`, mode/enable.
  - Fbk: `consumer_index`, `underrun`, `latency_ms`, motion state/faults.

## Pause/Resume and error behavior (motion only)
- **Pause motion (not the PLC)**: PLC stops consuming (holds last or decelerates via SoftMotion), publishes current `consumer_index` and motion state. Python may continue filling to maintain look‑ahead, or freeze generation.
- **Resume motion**: PLC re‑arms consumption and blends back in. Use SoftMotion `MC_GearInPos/CamIn` or a ramp to the next buffered sample to ensure continuity.
- **Axis error handling**: On drive/axis fault (following error, limit, CiA‑402 fault):
  - PLC stops consuming, commands SoftMotion `MC_Stop`/safe stop and records fault details.
  - Python halts generation or switches to safe idle; backlog is preserved but ignored.
  - Recovery: clear drive fault (PLC CiA‑402 reset), optionally re‑home, realign indices with `stream_reset`, then re‑arm and resume from the reported `consumer_index` or a re‑synced phase point.

## Where bottlenecks can appear (and how to avoid them)
- **Python trajectory generation**: Complex splines/IK can dominate CPU. Use batched generation, NumPy, precompute segments, and generate in chunks (`k = Bhigh - B`).
- **PLC/SoftMotion load**: Many axes at Ts = 2 ms, heavy blending/cams increase PLC CPU. Start at Ts = 4 ms; profile and tune limits/jerk.
- **Synchronization pitfalls**: Avoid per‑sample semaphore; use lockless indices and only publish `producer_index` after writing a sample.
- **Windows scheduling**: Python threads can jitter; keep refill at 20–100 Hz and sufficient look‑ahead (≥250 ms). Pin PLC and Python to different cores.
- **IPC overhead**: Negligible at these data rates; keep sample struct compact.

## Practical ring‑buffer tips
- Keep samples minimal (positions only) unless you need feedforward; SoftMotion can derive velocity/acc.
- Use power‑of‑two `capacity` and modulo masking; write sample then atomically update `producer_index`.
- Store `sample_period_ns`, `num_axes`, a `version` field in the header; validate on both sides.
- Prefer `float64` for stability; switch to `float32` only if you must halve memory.
- Ensure axes are contiguous per sample (sample‑major layout) for cache‑friendly consumption per cycle.
- Expose diagnostics: backlog (ms), underruns/overruns, PLC cycle deviation.
- On Windows services vs. user sessions, use `Global\\` names for mapping/semaphore.

## Timing targets
- **PLC cycle Ts**: 4 ms to start (2 ms if stable on the PC).
- **Look‑ahead**: 250–500 ms (Blow/Bhigh ≈ 62–125 samples at Ts = 4 ms; double at 2 ms).
- **Python refill**: 20–100 Hz, non‑critical timing.

## Deployment on Windows RTE
- Keep `codesys/controlOS_demo.project` (SoftMotion setup) and extend logic with the consumer FB.
- Export updated `codesys_types` via `codesys/export_types.py` when structs change.
- Implement a Windows IPC adapter in Python for named mapping/semaphore.

## Monitoring & safety
- **PLC**: cycle deviation, underruns, SoftMotion state, CiA‑402 states/faults, interlocks/E‑stop.
- **Python**: backlog (ms), producer rate, overruns/underruns, path generator status.

## Reasoning behind these choices
- **SoftMotion for execution**: Proven planner with constraints, blending, synchronization, and safety—minimizing custom trajectory/safety code.
- **Ring buffer look‑ahead**: Decouples non‑RT Python timing from deterministic PLC execution; simple backlog control yields stable motion.
- **PLC as timebase**: Ensures precise multi‑axis sync and EtherCAT DC alignment; Python remains the content source, not the clock.
- **Windows IPC**: Simple, robust transport mirroring the controlOS Linux pattern while fitting Windows RTE.

## OPTIONAL, POSSIBLE LATER IMPLEMENTATION: Adding Velocity / Acc to ringbuffer

### Benefits of adding velocity

- Smooth continuity at retarget boundaries: SoftMotion can match incoming velocity, avoiding “zero-vel” cusps when you update points (esp. with 20–100 ms updates).
- Lower following error in fast motion: velocity feedforward helps the drive/SoftMotion anticipate where you’re going between samples.
- Better speed shaping from Python: you prescribe tangency along the path rather than letting SoftMotion infer it.

### Benefits of adding acceleration

- Tighter tracking in highly dynamic moves: acceleration feedforward further reduces error during fast ramps.
- Exact boundary conditions for jerk-limited profiles: if your Python generator is the “truth,” providing a(t) lets the PLC reconstruct the intended jerk-limited motion more faithfully.

### Costs/trade‑offs

- More data per sample: pos-only 8·axes bytes; +vel doubles, +acc triples. Still tiny in absolute terms, but nonzero.
- More coupling constraints: your v/a must be consistent with p and sample period; discontinuities cause bumps SoftMotion has to smooth anyway.
- Possible redundancy/conflict: if SoftMotion re-plans/blends, it may partially ignore your derivatives unless configured to use them; misalignment creates “double planning.”
- Extra complexity/testing: derivative continuity across segments, unit/scaling checks, multi‑axis coherence, and drive/SoftMotion feedforward settings.
