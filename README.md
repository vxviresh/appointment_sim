# Appointment Booking Simulation

A Monte Carlo simulation comparing two warehouse appointment booking systems to understand the trade-off between operational protection and merchant experience.

## The Problem

Warehouse sites book inbound truck appointments by the hour. Today, the only constraint is **max trucks per slot** (e.g., 2 trucks/hour). This controls congestion at the dock but does nothing to control workload — if all trucks happen to be large, a site rated for 5,000 cartons/day can easily receive 15,000+ cartons with no system-level guardrail.

## The Proposed Change

Add a second constraint: a **daily carton cap**. A truck's booking is rejected if its carton count would push the site over the daily limit, even if a slot has truck capacity available.

Two constraints must now both pass for a booking to succeed:
1. The target slot has remaining truck capacity
2. The day's running carton total + this truck's cartons ≤ daily carton cap

## What We're Simulating

| | BEFORE | AFTER |
|--|--------|-------|
| Slot constraint | max trucks/slot | max trucks/slot |
| Carton constraint | none | daily carton cap |

The question: **what does the carton cap cost merchants, and how much does it protect operations?**

When a truck can't get its desired date, it gets pushed to the next available day (up to 7 days out). If nothing is available in that window, the truck gives up — lost demand.

## Metrics

**Operational experience**
- % of days where actual cartons received exceeds the daily plan (BEFORE is bad here)
- Daily carton utilization distribution

**Merchant experience**
- Average push-out days — how far merchants get bumped from their desired delivery date
- % of trucks that can't find any appointment in 7 days (lost demand)

## Scenarios

- **Normal** — demand roughly matches capacity
- **Peak** — demand at ~1.8x capacity
- **Ramp** — demand grows from 0.5x to 2x over 30 days (shows when push-out starts hurting)
- **Demand sweep** — steps from 0.5x to 2.5x capacity to find the exact inflection point where each system breaks down

## Simulation Details

- Monte Carlo: 500 runs × 30 simulated days per scenario
- Truck arrivals: Poisson-distributed daily with day-of-week weights
- Carton sizes: configurable Normal distribution, clipped to [500, 10,000]
- Booking behavior: trucks try to book 1–7 days before their desired arrival date
- No-show rate: ~12% (slot is wasted, cartons are released)
- Booking is first-come-first-served

See [`simulate.md`](simulate.md) for the full design spec and implementation plan.
