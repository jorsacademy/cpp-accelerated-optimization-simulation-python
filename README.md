# C++ Accelerated Industrial Simulation and Optimization

A cross-language scientific-computing project that keeps a transparent scalar Python Monte Carlo implementation as a correctness oracle and accelerates the same trajectory kernel in C++17 through pybind11.

## Problem

The model represents a fleet of industrial assets with four condition states:

```text
0 Healthy
1 Degraded
2 Critical
3 Failed
```

Every simulated day, a maintenance policy chooses which eligible assets can use a limited staffed maintenance capacity. Preventive work can restore degraded/critical assets; failed assets require replacement. Assets not maintained can deteriorate stochastically at the end of the day.

The finite first-stage policy is:

```text
maintenance threshold state in {1, 2, 3}
crew slots per day          in {1, 2, 3}
standby spare units         in {0, 1, 2}
```

There are exactly `3 * 3 * 3 = 27` candidate policies. All candidates are evaluated, so optimization is exact over this declared finite grid, not over every possible maintenance policy.

## Same randomness in Python and C++

Random-number generation is deliberately outside the C++ kernel. NumPy generates two contiguous tensors:

```text
degradation uniforms  [scenario, day, asset]
maintenance uniforms  [scenario, day, asset]
```

The Python reference and the C++ implementation consume the exact same draws. Tests compare the complete per-scenario vectors for cost, lost production, failures, and maintenance events.

This separates performance engineering from changes in stochastic semantics.

## C++ acceleration layer

The extension is implemented in C++17 and exposed with pybind11. The hot simulation loop releases the Python GIL. Packaging uses CMake and scikit-build-core.

The C++ algorithm is also factored into a standard-library-only header (`simulation_core.hpp`). A standalone C++ smoke oracle can therefore be compiled without Python or pybind11, independently checking core state-transition arithmetic.

## Cost / reliability model

Default assumptions include:

```text
Healthy -> Degraded daily probability   0.006
Degraded -> Critical probability        0.018
Critical -> Failed probability          0.070
preventive-maintenance success          0.92
replacement success                     0.98
Degraded productivity                   0.90
Critical productivity                   0.65
```

Economic coefficients cover preventive maintenance, replacement, staffed crew capacity, standby spare capacity, and lost production.

These are stylized industrial reliability assumptions, not parameters estimated from a specific plant.

## Risk-aware optimization

Each policy is evaluated using:

```text
0.75 * mean(cost) + 0.25 * CVaR95(cost)
```

Selection and validation random tensors use different seeds. The baseline repairs only failed assets, staffs one maintenance slot, and has no standby spare.

## Build

Modern Python build:

```bash
python -m pip install -v .
```

Editable development build:

```bash
python -m pip install -v -e .
```

The project requires a C++17 compiler. Build dependencies are declared in `pyproject.toml`.

## Run

Optimization:

```bash
python run_experiment.py
```

Benchmark:

```bash
python run_experiment.py --benchmark
```

Regression tests:

```bash
python -m unittest discover -s tests -v
```

Standalone C++ core oracle:

```bash
g++ -O3 -std=c++17 cpp/core_smoke.cpp -o core_smoke
./core_smoke
```

## Performance reporting discipline

No speedup is hard-coded into the tests or README before it is measured. CI requires numerical equivalence, not a minimum acceleration factor. Hosted-runner timing is hardware- and workload-specific and will be reported only after the GitHub Actions build executes the compiled extension.

## Modeling scope

This repository demonstrates compiled scientific-computing acceleration and finite-grid stochastic policy optimization. It is not a calibrated reliability digital twin. A production application would require condition-transition estimation, maintenance-effectiveness data, production-loss calibration, resource calendars, repair-duration modeling, asset heterogeneity, and model-drift validation.
