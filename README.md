# Automated Warehouse — AI Planning (PDDL)

**Course:** AI for Robotics II — University of Genova  
**Planner used:** Temporal Fast Downward (TFD)  
**PDDL requirements:** `:typing :durative-actions :numeric-fluents :universal-preconditions :equality`

---

## Overview

This project models an automated warehouse using PDDL temporal planning. Two mover robots transport crates from their storage locations to a loading bay, where one of two loader robots places them onto a conveyor belt. The domain captures:

- **Crate types** — light (≤ 50 kg), heavy (> 50 kg), and fragile
- **Transport modes** — single-robot for light crates, dual-robot for heavy/fragile or for faster light transport
- **Loader separation** — primary loader handles heavy crates; secondary handles light ones
- **Battery management** — each mover has 20 power units; recharge is available at the loading bay
- **Group ordering** — crates in the same group must be loaded in a specified sequence
- **Relay strategy** — for crates that cannot be reached in a single trip due to battery limits, transport is split via an intermediate midpoint

Transport time formulas:
- Single robot: `distance × weight / 100`
- Two robots: `distance × weight / 150`
- Empty robot travel: `distance / 10`

---

## Repository Structure

```
warehouse-pddl/
│
├── domain/
│   └── domain.pddl                   # Single domain file (all extensions included)
│
├── problems/
│   ├── problem0.5.pddl               # 2 crates — basic scenario, group ordering
│   ├── problem1.pddl                 # 3 crates — fragile crate, group ordering
│   ├── problem2.pddl                 # 4 crates — two groups, battery management
│   ├── problem3.pddl                 # 4 crates — relay strategy for unreachable crate
│   └── problem4.pddl                 # 6 crates — multiple groups, all fragile, battery
│
├── docs/
│   ├── warehouse_pddl_report.pdf  # Full project report
|   └── Assignment.pdf #Description of the project requirements
│
└── README.md
```

---

## Domain Extensions Implemented

| # | Extension | Description |
|---|-----------|-------------|
| 1 | Crate groups | Crates in the same group are loaded in a fixed order via `group-next` |
| 2 | Dual loaders | Primary handles heavy; secondary handles light/fragile-light |
| 3 | Battery management | Battery decreases with movement; recharge at loading bay (5 time units) |
| 4 | Fragile crates | Always require 2 movers; loading takes 6 time units instead of 4 |

---

## Problem Scenarios

| Problem | Crates | Max Dist | Fragile | Groups | Makespan | Plan Steps |
|---------|--------|----------|---------|--------|----------|------------|
| 0.5     | 2      | 20       | 0       | A      | 22.06    | 7          |
| 1       | 3      | 20       | 1       | A      | 32.77    | 11         |
| 2       | 4      | 20       | 1       | A, B   | 69.19    | 20         |
| 3       | 4      | 30       | 1       | A      | 97.31    | 32         |
| 4       | 6      | 30       | 5       | A, B   | 55.52    | 27         |

Problem 3 required a relay strategy: crate3 (60 kg, 30 units away) exceeded the battery budget for a direct trip (21 units needed vs. 20 available), so transport was split via an intermediate midpoint at `loc15`.


## Key Design Decisions

**Loader role separation:** The primary loader is restricted to `(is-heavy ?c)` crates; the secondary to `(not (is-heavy ?c))`. This prevents the planner from trivially assigning all crates to the primary loader and ensures realistic load balancing.

**Relay mechanism:** `move-crate-duo-relay` transports a heavy crate from its origin to a midpoint; `move-crate-duo-to-final` completes the second leg after robots recharge. Currently implemented for heavy crates only.

**Loading bay exclusivity:** The predicates `loading-in-progress-primary` and `loading-in-progress-secondary` prevent concurrent loading on the same arm. Additionally, no crate can be delivered to the bay while loading is in progress.

---

## Heuristic (Part II)

A domain-specific admissible heuristic `h: S → ℕ` estimates remaining plan cost from any state:

```
h(state):
  h_value = 0
  for each crate c not yet loaded:
    dist   = distance(c)
    weight = weight(c)
    if weight > 50:
      time = dist * weight / 100
    else:
      time = min(dist * weight / 100, dist * weight / 150)
    h_value += ceil(time)
  return h_value
```

For Problem 0.5: `h = ceil(7) + ceil(2.67) = 7 + 3 = 10` vs. actual makespan 22.06.  
The gap is expected — the heuristic ignores loading times, recharge delays, and resource contention, so it always underestimates (admissible).

---

## Pattern (Part II)

Minimal symbolic pattern for Problem 0.5:

```
{ at(robot1, loc10), at(robot2, loc10),
  crate-at(crate1, loc10), crate-at(crate2, loc20),
  crate-loaded(crate1), crate-loaded(crate2) }
```

Estimated bound ≈ 22.06, matching the TFD makespan exactly. For larger problems the bound increases with crate count and distance; the pattern should be extended with `battery-level`, intermediate positions, and `group-next` predicates to remain tight.

---

## Authors
- AGUENAROUS Mohamed
- Ferdjallah Cheima
- Mokrani Lisa


