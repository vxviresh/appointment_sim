# Monte Carlo Appointment Simulation v2

## Objective

Build a single-site Monte Carlo simulator that compares:

- **Baseline policy**: current slot-only booking
- **Candidate policies**: higher slot capacity plus a configurable daily carton cap

The simulator must quantify the tradeoff between:

- operational protection from daily carton overload
- merchant booking outcomes when demand is normal and during peak

The simulator should not encode an "acceptable" merchant threshold in v1. It should report the tradeoff surface clearly enough for a human decision after results are reviewed.

## Inputs

The model must print or save a run configuration summary that clearly separates static assumptions, policy settings, and scenario settings.

### Static inputs

These define the site and synthetic demand behavior. They do not change within a policy sweep unless the user explicitly changes them.

- `num_days`
  - number of simulated desired-arrival days in one run
- `num_runs`
  - number of Monte Carlo runs per scenario
- `random_seed`
  - base seed used to generate reproducible paired demand
- `slots_per_day`
  - number of appointment slots available each day
- `baseline_max_trucks_per_slot`
  - current-state truck capacity per slot
- `day_of_week_multipliers`
  - relative demand by weekday
- `carton_mean`
  - mean cartons per truck for synthetic generation
- `carton_std`
  - standard deviation of cartons per truck
- `carton_min`
  - lower clip bound for cartons per truck
- `carton_max`
  - upper clip bound for cartons per truck
- `booking_lead_distribution`
  - probability distribution for how many days before desired date a merchant tries to book
- `max_search_days`
  - maximum days after desired date the merchant is willing to search for an appointment
- `no_show_rate`
  - probability that a booked truck does not actually arrive

### Scenario inputs

These control volume conditions. They are configurable and should be passed separately from policy settings.

- `normal_load_multiplier`
  - default `1.0`
- `peak_load_multiplier`
  - default `1.8`
- optional future scenarios may be added later, but v1 only requires normal and peak

### Policy inputs

These are the variables being tested.

- `daily_carton_cap`
  - configurable operational cap on total booked cartons per day
- `candidate_max_trucks_per_slot`
  - slot capacity per slot under a candidate policy

### How each input is used

- `slots_per_day` and `baseline_max_trucks_per_slot` define baseline truck booking capacity
- `candidate_max_trucks_per_slot` tests whether raising slot capacity offsets the merchant friction introduced by a carton cap
- `daily_carton_cap` blocks a booking when booked cartons for a day plus the candidate truck cartons would exceed the cap
- carton distribution parameters determine truck size and therefore ops stress and cap friction
- booking lead distribution determines desired-date pressure and merchant timing
- `max_search_days` defines when a merchant effectively fails to get an acceptable appointment
- `no_show_rate` allows comparison of booked operational stress versus actual-arrival operational stress
- load multipliers change total demand and represent normal versus peak booking volume

## Entities

### 1. Site

Represents one warehouse site.

Fields:
- `site_id`
- `slots_per_day`
- `baseline_max_trucks_per_slot`
- `daily_carton_cap`

### 2. Truck request

Represents one merchant appointment request in the simulation.

Fields:
- `request_id`
- `desired_date`
- `booking_day`
- `cartons`
- `scenario_name`

Derived outcome fields per policy:
- `booked_date`
- `booked_slot`
- `pushout_days`
- `found_appointment`
- `showed_up`

### 3. Policy

Represents one booking rule set.

Fields:
- `policy_name`
- `max_trucks_per_slot`
- `daily_carton_cap`
- `is_baseline`

### 4. Daily site state

Represents booking state for one day under one policy.

Fields:
- `day`
- `slot_truck_counts`
- `booked_cartons`
- `actual_arrival_cartons`
- `booked_request_ids`

## Booking Logic

### Demand generation

For each run and scenario:

1. For each simulated day, generate truck request volume using:
   - `Poisson(base_rate * day_of_week_multiplier * load_multiplier)`
2. For each request, generate:
   - `cartons` from a configurable distribution
   - `desired_date`
   - `booking_day = desired_date - sampled_lead`, clamped to the simulation horizon
3. Use the same generated requests for every policy in that run.

### Baseline booking rule

For each request ordered by `booking_day` and a deterministic tie-breaker:

1. Start with `desired_date`
2. Search candidate days through `desired_date + max_search_days`
3. For each candidate day, scan slots in a deterministic order
4. Accept the first slot where:
   - `slot_truck_count < baseline_max_trucks_per_slot`
5. If no slot is found in the search window:
   - mark request as not booked

### Candidate policy booking rule

Use the same request order and search process, but the booking must satisfy both:

1. `slot_truck_count < candidate_max_trucks_per_slot`
2. `booked_cartons_for_day + request.cartons <= daily_carton_cap`

If both pass, book the truck into the first qualifying slot.

### No-shows

After bookings are complete:

1. Sample `showed_up` for each booked request using `no_show_rate`
2. `booked_cartons` stays unchanged because it reflects planned workload
3. `actual_arrival_cartons` is the sum of cartons for booked requests that actually show up

This separation is required because the business cares about both planned overload and realized overload.

### Paired comparison requirement

Every run must use paired comparison:

- generate one demand sample
- replay it through baseline and every candidate policy

This is required for fair request-level comparisons such as "earlier than baseline."

## Metrics

### Primary metrics

#### Metric 1: percent of days over carton limit

Report both:

- `% days where booked_cartons > daily_carton_cap`
- `% days where actual_arrival_cartons > daily_carton_cap`

Notes:
- baseline has no enforced cap, but this metric still uses the comparison cap to measure how often the site would have exceeded that limit
- candidate policies should materially reduce this metric

#### Metric 2: percent of merchants booked within 5 days of desired date

Definition:

- numerator: requests with `found_appointment = true` and `pushout_days <= 5`
- denominator: all requests

This is the main merchant-experience metric in v1.

#### Metric 3: percent of merchants booked earlier than baseline

Definition:

- numerator: requests booked under candidate policy strictly earlier than under baseline
- denominator: all requests, or booked requests only if the reporting view wants a conditional measure

Default v1 should report both:

- `% of all requests booked earlier than baseline`
- `% of booked requests booked earlier than baseline`

### Supporting metrics

- `% requests pushed more than 5 days`
- `% requests with no appointment found`
- mean pushout days
- median pushout days
- p95 pushout days
- average booked cartons per day
- average actual-arrival cartons per day

## Output Tables

### Table 1: run configuration

One table per execution showing:

- scenario name
- static inputs
- policy sweep inputs
- seed / reproducibility info

### Table 2: scenario summary

One row per policy per scenario:

| scenario | policy_name | trucks_per_slot | carton_cap | pct_days_over_cap_booked | pct_days_over_cap_actual | pct_within_5_days | pct_earlier_than_baseline | pct_pushout_gt_5 | pct_no_appointment | mean_pushout | p95_pushout |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|

### Table 3: policy deltas vs baseline

One row per candidate policy per scenario:

| scenario | policy_name | delta_pct_days_over_cap_booked | delta_pct_within_5_days | delta_pct_earlier_than_baseline | delta_pct_pushout_gt_5 | delta_pct_no_appointment |
|---|---|---:|---:|---:|---:|---:|

### Table 4: policy frontier

This table answers how much higher slot capacity must be to offset merchant harm for a given carton cap.

| scenario | carton_cap | trucks_per_slot | pct_within_5_days | pct_pushout_gt_5 | pct_days_over_cap_booked | frontier_rank |
|---|---:|---:|---:|---:|---:|---:|

The frontier logic in v1 should not declare a winner automatically. It should simply order policies so a reviewer can see the minimum slot increase associated with improving ops protection at each cap level.

## Policy Sweep Design

### Required scenarios

- `normal`
  - load multiplier = `normal_load_multiplier`
- `peak`
  - load multiplier = `peak_load_multiplier`
  - default `1.8`

### Required policies

1. Baseline:
   - `max_trucks_per_slot = baseline_max_trucks_per_slot`
   - `daily_carton_cap = None`

2. Candidate policies:
   - `max_trucks_per_slot = baseline_max_trucks_per_slot + delta`
   - `daily_carton_cap = configured cap`

### Minimum sweep for v1

If baseline trucks-per-slot is `B`, test at least:

- `B`
- `B + 1`
- `B + 2`
- `B + 3`

with one or more configured carton-cap values.

If run time is acceptable, allow a denser sweep.

### Evaluation approach

For each scenario:

1. run baseline
2. run all candidate policies against the same paired demand
3. calculate scenario summary metrics
4. calculate deltas versus baseline
5. produce a frontier view showing the slot increase required at each carton cap

## Assumptions

- single-site only
- carton count is the operational workload proxy for v1
- merchants can search beyond desired date up to `max_search_days`
- merchants are mostly static in v1; adaptive behavior is not deeply modeled
- slot order is deterministic
- booking is first-come-first-served based on `booking_day`
- desired-date behavior is synthetic for now but designed to be replaced with real data-derived assumptions later
- peak demand is higher booking volume, default `1.8x`, and configurable

## Non-goals

The following are intentionally out of scope for v1:

- network routing across multiple sites
- merchant priority classes
- detailed intra-day dock queue simulation
- dynamic merchant gaming or strategic behavior
- learning effects from repeated exposure to the policy
- direct optimization of the "best" carton cap
- hardcoded acceptability thresholds for merchant experience

## Future data integration

The implementation should keep these layers replaceable:

- truck arrival process
- carton size generation
- desired-date generation
- booking lead generation
- no-show process

This allows later calibration using real site slot, date, appointment, and arrival data without changing the policy engine.
