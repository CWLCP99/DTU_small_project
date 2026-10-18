# Synthetic EV Charging Fault Simulator

A small Python simulator that generates synthetic AC charging sessions of an electric vehicle, as seen from the car side (onboard charger signals), including three fault types.

It is a minimal, self-contained version of the data side of my master's thesis at BMW, where I analyse EV charging error sequences with unsupervised learning. The real data is confidential, so this project rebuilds the idea on synthetic data whose behaviour is modelled on what I see in real onboard-charger recordings.

## Signals

Each session is sampled once per second with five signals:

Signal ---- Unit / values ---- Notes 
current ---     A         ---- ramps up, hold and ramps down. 
voltage ---     V         ---- car side voltage: 0 V in B and 230 V A
power -----     W         ---- computed Voltage x Current
pilot_status -- B,C,E,F   ----- control pilot states on IEC 61851
status_charging-- idle,charging etc ----- charging_state session

## Session types

Every session is built from phases (waiting, ramp, plateau, …). All phase lengths are drawn randomly per session from fixed ranges.

- **`power_fault`**: the supply fails during the plateau. Current and voltage drop instantly to 0, pilot goes to E, status becomes `error`.
- **`pilot_fault`**: a communication problem. The pilot flaps between C and B several times (current and voltage drop to 0 each time it is B), then the session ends with pilot F and status `error`.
- **`thermal_derating`**: during charging, current steps down from 16 A to 8 A and the session finishes at reduced current with status `complete`. No error status: the fault is only visible in the current shape.

## Code structure

- make_per_phase(length, current_start, current_end, voltage, pilot_status, status_charging) --> builds one phase with all signals. One call corresponds to one row of a phase table. Includes `assert` tests for a constant and a ramp phase.
- simulate_sessions(session_type, rng) --> builds one session from a list of phases and adds the time column.
- generate_dataset(n_per_type, rng) --> generates `n_per_type` sessions per type, adds `session_id` and the ground-truth `session_type`, and returns one DataFrame.

The random generator is created once with a fixed seed (`np.random.default_rng(42)`), so the full dataset is reproducible.

## How to run

```bash
pip install numpy pandas matplotlib pyarrow
jupyter notebook
```

Open the notebook and run all cells top to bottom. It generates 300 sessions (100 per type) and saves them as `sessions.parquet`. The data file is not committed because it can be regenerated exactly from the seed.

## Next steps

- **Feature extraction:** one row per session (duration, mean current, energy, number of pilot changes, whether the session ends in error).
- **Clustering:** k-means on scaled features, choosing k with the silhouette score, and checking against the ground-truth types with the adjusted Rand score.
- **On-change format:** convert the per-second table to the on-change telemetry format used in real vehicle data, where a signal is only recorded when its value changes.
- **Noise and quantisation:** add realistic noise (e.g. voltage varying around 230 V) together with quantisation or a deadband, so the on-change format stays compact.
- **Random number of flaps** in the pilot fault, generated with a loop instead of a fixed list.
