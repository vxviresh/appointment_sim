# Appointment Booking Policy Simulation

This repository defines a Monte Carlo simulation for one decision:

`If a site adds a daily carton cap and raises slot capacity, how much operational protection does it gain and what happens to merchant booking outcomes?`

The simulator is intentionally directional in v1. It uses configurable synthetic demand and booking behavior now, but is structured so real site data can replace those assumptions later.

See [`simulate_v2.md`](simulate_v2.md) for the implementation-ready spec.

## Problem

Today the site books inbound appointments using only a slot constraint:

- each time slot has a maximum number of trucks
- a truck consumes exactly 1 unit of slot capacity in a slot
- the system does not directly limit total cartons booked for a day

That means a day can look safe on truck count while still blowing past the site's operational carton plan if too many large trucks book on the same day.

## Policy Being Tested

The simulation compares:

- **Baseline**
  - current `max_trucks_per_slot`
  - no daily carton cap

- **Candidate policies**
  - higher `max_trucks_per_slot`
  - configurable `daily_carton_cap`

This is not just a binary before/after model. The simulator should sweep multiple slot increases against one or more carton-cap values to find the tradeoff surface.

## What the Simulation Measures

### Metric 1: Operational protection
- `% of days over carton limit`
- tracked for both booked cartons and actual-arrival cartons

### Metric 2: Merchant booking outcome
- `% of merchants booked within 5 days of desired date`
- formula: `booked_date - desired_date <= 5`

### Metric 3: Merchant improvement vs baseline
- `% of merchants booked earlier than baseline`
- formula: `booked_date_candidate < booked_date_baseline`
- this matters because higher slot capacity may help smaller loads get in earlier even with a carton cap

### Supporting metrics
- `% of merchants pushed more than 5 days`
- `% of merchants with no appointment found within the search window`
- mean, median, and p95 push-out days

## How the Simulation Works

For each Monte Carlo run:

1. Generate truck demand for each day.
2. Assign cartons to each truck.
3. Assign each truck a desired appointment date and booking attempt timing.
4. Run the same demand through:
   - baseline policy
   - each candidate policy
5. Compare outcomes at the truck/request level and aggregate metrics across runs.

The key design choice is paired comparison: the exact same synthetic demand sample is replayed through every policy being compared.

## Key Inputs

### Static inputs
- `slots_per_day`
- `baseline_max_trucks_per_slot`
- `day_of_week_multipliers`
- carton-size distribution parameters
- booking lead / desired-date assumptions
- no-show rate
- simulation horizon
- number of Monte Carlo runs
- search horizon beyond desired date

### Policy inputs
- `candidate_max_trucks_per_slot`
- `daily_carton_cap`

### Demand scenario inputs
- `normal_load_multiplier`
- `peak_load_multiplier`

Volume fluctuation is a core part of the model. The simulator should support at least:

- **Normal load**: baseline demand level
- **Peak load**: supports two modes:
  - **Flat**: uniform `1.8x` multiplier across all days (simple, backward-compatible)
  - **Clustered** (default): places multi-day surge windows (e.g., 2-4 consecutive days at `2.5x`) within the horizon, with non-surge days at a lower base (e.g., `1.2x`). This models real peak dynamics where consecutive heavy days create cascading pushout pressure that a flat multiplier understates.

## Output Shape

For each scenario and policy, the model should produce:

- one summary row per policy
- baseline deltas
- merchant outcome distributions
- operational protection distributions
- a policy frontier showing how much slot capacity must increase to offset merchant harm under a given carton cap

## Important Notes

- v1 should not hardcode what "acceptable" means
- the model should show tradeoff curves first, then humans can decide what is acceptable
- desired-date behavior is synthetic and must remain configurable so real data can replace it later
- single-site only in v1
- carton count is the operational proxy for now
