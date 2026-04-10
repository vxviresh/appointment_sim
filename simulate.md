# Monte Carlo Appointment Booking Simulation

## Context
The warehouse appointment system currently only constrains by max trucks per time slot. This means a site can get overwhelmed with cartons even when slots aren't "full" (e.g., 16 trucks each bringing 1,000 cartons = 16,000 cartons on a 5,000-carton-capacity day). The proposed change adds a daily carton cap. This simulation compares BEFORE (slot-only) vs AFTER (slot + carton cap) under varying demand levels using Monte Carlo methods.

## Files
- `requirements.txt` — numpy, scipy, pandas, matplotlib, jupyter
- `appointment_simulation.ipynb` — single notebook with everything

## Simulation Design

### Two Systems
| | BEFORE | AFTER |
|--|--------|-------|
| Slot constraint | max_trucks_per_slot (e.g., 2) | same |
| Carton constraint | **none** | daily_carton_cap (e.g., 5000) |

### Config (all in first notebook cell, easily tunable)
- `slots_per_day=8`, `max_trucks_per_slot=2` → 16 truck slots/day
- `daily_carton_cap=5000`
- `base_truck_rate=10/day`, `day_of_week_multipliers=[1.2, 1.0, 1.0, 1.0, 0.8, 0.5, 0.0]`
- `carton_mean, carton_std` (Normal distribution, clipped to [500, 10000])
- `booking_lead_distribution` — 7-element array (% trying to book 7,6,...,1 days before desired date)
- `no_show_rate=0.12`, `max_pushout_days=7` (give up if nothing in 7 days)
- `num_days=30`, `num_runs=500`

### Demand Generation
1. For each of 30 days, draw `Poisson(base_rate x dow_weight x demand_multiplier)` trucks
2. Each truck gets `cartons ~ Normal(mean, std)` clipped to [500, 10000]
3. Each truck gets a booking_day = desired_date - lead (lead sampled from booking_lead_distribution, clamped >= 0)
4. For ramp scenario: demand_multiplier grows linearly from 0.5x to 2.0x over 30 days

### Booking Algorithm
1. Sort all trucks by booking_day (first-come-first-served)
2. For each truck: try desired_date, then desired_date+1, ..., up to +7
3. For each candidate day, scan slots:
   - **BEFORE**: `slot_trucks[day][slot] < max_trucks_per_slot` -> book
   - **AFTER**: same **AND** `day_cartons[day] + truck_cartons <= daily_carton_cap` -> book
4. If no slot found in 7-day window -> lost demand (truck gives up)
5. After all bookings: apply no-shows (remove cartons from day total, but slot remains occupied/wasted)

**Key design choice**: Same demand (same RNG seed) feeds both BEFORE and AFTER systems per run -> paired comparison isolates the effect of the carton cap.

### Scenarios
- **Normal**: demand_multiplier = 1.0 (demand ~ capacity)
- **Peak**: demand_multiplier = 1.8 (demand ~ 1.8x capacity)
- **Ramp**: multiplier grows 0.5 -> 2.0 over 30 days (shows when push-out starts hurting)

### Demand Sweep
- Multipliers from 0.5 to 2.5 in 0.1 steps, 100 runs each
- Shows exactly where each system breaks down

## Metrics

### 1. Appointment Push-Out (days)
- Mean, median, p95 of (booked_date - desired_date) across all trucks and all Monte Carlo runs
- This is the primary "merchant experience" metric

### 2. Operational Experience
- **% days over carton plan**: days where booked cartons > 5,000 (BEFORE will be bad here — this is the problem)
- **% days under-utilized**: actual cartons < 50% of capacity
- **Avg daily carton utilization**: actual cartons / capacity

### 3. Merchant Experience
- **Avg push-out days**: how far merchants get pushed from their desired date
- **% trucks lost**: no appointment available in 7-day window (truck gives up)
- **Push-out distribution**: histogram of push-out days (0, 1, 2, ..., 7)

## Notebook Cell Plan

| # | Cell | Content |
|---|------|---------|
| 1 | Setup | Imports, pip install if needed, matplotlib style |
| 2 | Config | `SimConfig` dataclass with all parameters, scenario instantiation |
| 3 | Demand Gen | `generate_demand()` — Poisson arrivals, Normal cartons, booking lead times |
| 4 | Sim Engine | `simulate_single_run()` — core booking loop (the heart of the simulation) |
| 5 | MC Runner | `run_monte_carlo()` — outer loop with paired BEFORE/AFTER runs |
| 6 | Metrics | `aggregate_metrics()` — compute all summary statistics |
| 7 | Run All | Execute 3 scenarios x 2 systems, print summary table |
| 8 | Demand Sweep | Run sweep from 0.5x to 2.5x capacity |
| 9 | Chart | Push-out comparison: grouped bars (before/after x scenario) with confidence intervals |
| 10 | Chart | Demand sweep curve: push-out vs demand multiplier (line plot with CI bands) |
| 11 | Chart | Carton utilization: heatmap + histogram of daily carton loads |
| 12 | Chart | Lost demand rates: bar chart across scenarios |
| 13 | Chart | Push-out distribution: histogram of push-out days (0-7) |
| 14 | Chart | Confidence intervals: box plots of per-run variance |
| 15 | Summary | Styled pandas DataFrame with all metrics at a glance |

## Key Assumptions
- Wave logic (progressive slot opening) is NOT simulated — it only distributes trucks across slots, which we handle by scanning slots in order
- Real constraints are **trucks per hour** and **working hours** (= slots_per_day x max_trucks_per_slot)
- No-shows: booked trucks that don't show up still consume a slot (wasted) but release their cartons
- Trucks that can't book within 7 days of their desired date give up entirely (lost demand)
- Booking is first-come-first-served based on when the truck tries to book

## Verification Checklist
1. Run notebook end-to-end with default config
2. Sanity: BEFORE system should show high % days over carton plan under peak demand
3. Sanity: AFTER system should show higher push-out and lost demand, but operations are protected
4. Verify paired comparison: same seed produces identical demand for both systems
5. Normal scenario should show minimal difference (both systems handle it fine)
6. Demand sweep curve should show a clear inflection point where push-out jumps
