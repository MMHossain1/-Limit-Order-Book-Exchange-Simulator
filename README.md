# Limit Order Book & Exchange Simulator

### A Deterministic Limit Order Book Built in Python

A Python-based exchange simulator designed to explore order matching, market microstructure, data structures, correctness, and performance engineering.

The project implements an event-driven limit order book with price-time priority, partial executions, order cancellation, deterministic event replay, and a rigorous testing and benchmarking framework.

The goal is not just to build a working matching engine, but to understand **why it works, how efficiently it operates, and how to prove its correctness.**

---

## Overview

A limit order book maintains outstanding buy and sell orders at different price levels. When compatible orders arrive, the matching engine executes trades according to the exchange's matching rules.

This project models the process in a reproducible environment, allowing experiments with matching algorithms, data structures, event streams, and performance bottlenecks.

### Core objectives

- Implement a correct price-time priority matching engine.
- Model order lifecycles and partial executions.
- Design efficient data structures for price levels and individual orders.
- Validate correctness using unit tests, property-based testing, and differential testing.
- Measure throughput, latency, and memory usage under different workloads.
- Investigate performance bottlenecks through profiling and controlled experiments.
- Make every simulation deterministic and reproducible.

## Features

> Implementation status: features should be marked complete only after they have been implemented and tested.

| Feature | Description |
|---|---|
| Limit orders | Orders that execute only at an allowed price or better. |
| Market orders | Orders that seek immediate execution against available liquidity, subject to defined order constraints. |
| Price-time priority | Better prices execute first; orders at the same price follow FIFO arrival order. |
| Partial fills | Orders can execute across multiple counterparties or remain partially unfilled. |
| Order cancellation | Remove eligible resting orders while maintaining book consistency. |
| Trade generation | Record execution price, quantity, maker order, and taker order. |
| Market depth | Query aggregated quantities at individual price levels. |
| Best bid and ask | Efficiently inspect the highest bid and lowest ask. |
| Event replay | Reproduce a simulation from the same ordered event stream. |
| Correctness testing | Validate matching behavior and accounting invariants. |
| Performance benchmarking | Measure throughput, latency distributions, and memory consumption. |

## How the Matching Engine Works

The matching engine follows price-time priority.

For buy orders:
- Higher bid prices have priority.
- At the same price, earlier eligible orders have priority.

For sell orders:
- Lower ask prices have priority.
- At the same price, earlier eligible orders have priority.

A trade can occur when the incoming order is compatible with the best resting order on the opposite side.

### Example

Suppose the order book contains:

| Side | Price | Quantity |
|---|---:|---:|
| Ask | 101 | 7 |
| Ask | 102 | 12 |
| Bid | 99 | 10 |
| Bid | 98 | 15 |

An incoming buy limit order arrives with a limit price of `101` and a quantity of `10`.

The matching engine executes:

1. Buy 7 units at 101, consuming the best ask.
2. The remaining 3 units cannot execute at 102 because that price exceeds the buyer's limit.
3. The remaining buy quantity rests at 101, assuming the order's time-in-force permits resting.

The resulting book must preserve the applicable matching rules and all quantity invariants.

## Architecture

The project is organised around distinct responsibilities.

```text
limit-order-book/
├── src/
│   └── orderbook/
│       ├── __init__.py
│       ├── models.py
│       ├── order_book.py
│       ├── matching_engine.py
│       ├── events.py
│       ├── trades.py
│       ├── market_data.py
│       ├── replay.py
│       └── benchmarks.py
├── tests/
│   ├── test_matching.py
│   ├── test_order_lifecycle.py
│   ├── test_price_time_priority.py
│   ├── test_partial_fills.py
│   ├── test_invariants.py
│   ├── test_replay.py
│   └── test_properties.py
├── examples/
│   └── basic_simulation.py
├── benchmarks/
│   └── benchmark_engine.py
├── docs/
│   └── design.md
├── pyproject.toml
├── LICENSE
└── README.md
```

### Component responsibilities

- **`models.py`** — Defines orders, sides, order types, statuses, and execution records.
- **`order_book.py`** — Maintains bid and ask price levels, queues of resting orders, and order lookup structures.
- **`matching_engine.py`** — Processes incoming orders and generates executions according to the matching policy.
- **`events.py`** — Defines the events that modify the simulated exchange state.
- **`trades.py`** — Represents executions and maintains trade records.
- **`market_data.py`** — Exposes best prices and aggregated market depth.
- **`replay.py`** — Replays a recorded event stream to reproduce the resulting state.
- **`benchmarks.py`** — Provides repeatable workloads and performance measurements.

## Design Decisions

### 1. Integer price representation

Prices are represented using integer ticks rather than binary floating-point values.

For example, if the minimum price increment is `0.01`, a price of `101.25` can be represented as `10125` ticks.

This avoids floating-point rounding issues in price comparisons.

### 2. Explicit order lifecycle

Each order has a unique identifier, side, limit price where applicable, original quantity, remaining quantity, and status.

The engine must define how orders transition between states such as active, partially filled, filled, and cancelled.

Invalid state transitions should be rejected or handled according to documented rules.

### 3. Data structures

The initial implementation prioritises clarity and correctness. Subsequent implementations can compare alternative approaches.

Potential structures include:

- Dictionaries for direct order-ID lookup.
- FIFO queues for orders at each price level.
- Heaps for selecting extreme price levels, with appropriate handling of stale entries.
- Ordered price-level structures for efficient price traversal and deletion.

The optimal design depends on the workload and required operations. The project aims to measure these trade-offs rather than assume one structure is always best.

### 4. Deterministic execution

Given the same initial state, configuration, and ordered event stream, the engine should produce the same executions and final state.

Randomised workloads should use explicitly recorded seeds.

Determinism makes debugging, regression testing, and performance comparisons easier.

## Correctness and Testing

A matching engine can appear to work on simple examples while failing on edge cases. Correctness is therefore a first-class design requirement.

### Unit tests

Test individual behaviors, including:

- Empty-book behavior.
- Orders that do not cross the spread.
- Exact-price matches.
- Multiple orders at the same price.
- Partial fills and multi-order executions.
- Cancellation of resting orders.
- Orders that are fully filled.
- Orders that remain partially unfilled.
- Invalid order IDs and quantities.

### Property-based testing

Generate many valid and invalid order sequences to uncover cases that hand-written examples may miss.

Useful invariants include:

- Resting orders have positive remaining quantities.
- An order ID cannot identify multiple active orders.
- Executed quantity never exceeds the available quantity of either counterparty.
- Trade quantities are positive.
- Book quantities equal the sum of the remaining quantities of their constituent orders.
- The book does not retain an executable crossed market after matching has completed.

### Differential testing

Build a simple reference implementation and compare its executions and final state with the main engine across the same generated order streams.

This helps detect discrepancies introduced by optimisation.

## Performance Engineering

The project is designed to make performance measurable.

Rather than relying on a single headline number, the benchmark suite should evaluate different workloads and report the conditions under which measurements were collected.

### Metrics

| Metric | Purpose |
|---|---|
| Events per second | Measures processing throughput. |
| Median latency | Measures typical per-event processing time. |
| p95 / p99 latency | Shows the slower tail of the latency distribution. |
| Peak memory usage | Measures memory requirements under load. |
| Operation mix | Describes the proportions of submissions, cancellations, and executions. |
| Correctness checks | Ensures the benchmarked implementation preserves expected behavior. |

### Benchmark methodology

1. Generate a deterministic workload.
2. Record the workload composition and random seed.
3. Warm up the implementation where appropriate.
4. Run multiple trials in a controlled environment.
5. Report latency distributions and throughput.
6. Profile CPU and memory usage.
7. Optimise identified bottlenecks.
8. Rerun identical workloads and compare results.
9. Verify correctness after optimisation.

Benchmark results depend on hardware, Python version, workload composition, and implementation details. Results should include enough context to be reproduced.

### Performance results

*Populate this table after running the benchmark suite.*

| Workload | Events | Throughput | p50 latency | p99 latency |
|---|---:|---:|---:|---:|
| Order submissions | TBD | TBD | TBD | TBD |
| Cancellation-heavy | TBD | TBD | TBD | TBD |
| Match-heavy | TBD | TBD | TBD | TBD |
| Mixed workload | TBD | TBD | TBD | TBD |

No performance claims are made until measurements have been collected.

## Installation

Requires Python 3.12 or later.

Clone the repository:

```bash
git clone https://github.com/YOUR_USERNAME/limit-order-book.git
cd limit-order-book
```

Create and activate a virtual environment:

```bash
python -m venv .venv
```

On macOS or Linux:

```bash
source .venv/bin/activate
```

On Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Install the project and development dependencies:

```bash
python -m pip install --upgrade pip
pip install -e ".[dev]"
```

This installation command assumes development dependencies are configured in `pyproject.toml`.

## Running Tests

Run the complete test suite:

```bash
pytest
```

Run a particular test module:

```bash
pytest tests/test_matching.py
```

Run tests with more detailed output:

```bash
pytest -v
```

## Running a Simulation

Once the example implementation is available, run:

```bash
python examples/basic_simulation.py
```

A simulation should demonstrate order submission, matching, partial execution, trade recording, and a final market-depth snapshot.

## Roadmap

- [ ] Implement order and execution models.
- [ ] Implement bid and ask price levels.
- [ ] Implement price-time priority.
- [ ] Support partial fills and cancellations.
- [ ] Add market-depth queries.
- [ ] Build comprehensive unit tests.
- [ ] Add property-based testing.
- [ ] Implement a reference matcher for differential testing.
- [ ] Add deterministic event replay.
- [ ] Create repeatable performance workloads.
- [ ] Profile CPU and memory usage.
- [ ] Compare alternative data structures.
- [ ] Publish measured benchmark results.
- [ ] Add continuous integration and static analysis.
- [ ] Document design trade-offs and known limitations.

## Scope and Limitations

This is an educational engineering project, not a production exchange or trading system.

Exchange-specific behaviors vary. Features such as self-trade prevention, order modification priority, market-order protection, auction matching, and time-in-force rules must be explicitly defined before claiming support.

The project does not claim exchange-grade latency, production readiness, or profitability from trading strategies.

## Technology Stack

- **Language:** Python
- **Testing:** pytest, Hypothesis
- **Benchmarking:** pyperf or an equivalent controlled harness
- **Profiling:** cProfile, tracemalloc
- **Code quality:** Ruff, mypy
- **Automation:** GitHub Actions

## Why This Project?

This project explores several important systems-engineering challenges:

- Translating formal matching rules into correct state transitions.
- Selecting data structures based on operation costs.
- Designing tests that validate entire sequences of state changes.
- Making event-driven simulations deterministic.
- Measuring performance without confusing throughput with latency.
- Optimising software without sacrificing correctness.

The central principle is simple:

**Correctness first. Measurement second. Optimisation third.**

## Contributing

Contributions and technical discussions are welcome. Bug reports should include a minimal reproduction, expected behavior, and actual behavior.

Proposed optimisations should include benchmark methodology and demonstrate that the engine's correctness tests continue to pass.

## License

This project is distributed under the MIT License. See `LICENSE` for details.

---

**Build it correctly. Measure it rigorously. Optimise it deliberately.**
