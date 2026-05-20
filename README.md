# Explosive Synchronization and Flash Crash in Networked Oscillators

This repository models synchronization transitions in networks of coupled oscillators. It covers two dynamical systems (Kuramoto and Stuart-Landau), two network models (ER and BA), and the analysis needed to detect abrupt changes in collective behavior such as flash-crash style drops in coherence.

The workflow is split into a fast C++ simulator and a Python analysis/plotting layer. Data is exported as simple CSV/NPY files so the steps are easy to inspect and reproduce.

## Explosive Synchronization

Explosive synchronization is a first-order phase transition in networked oscillators where the order parameter $R$ jumps discontinuously from near-zero to near-one as coupling $K$ crosses a critical value. Unlike the continuous (second-order) Kuramoto transition on homogeneous networks, the explosive transition exhibits hysteresis: the forward and backward critical couplings differ. This requires frequency-degree correlation on heterogeneous networks (Gómez-Gardeñes et al., PRL 2011).

## What this project adds

- A clean separation between simulation (C++) and analysis (Python)
- Network generation for ER and BA graphs with consistent parameters
- Order-parameter tracking and early-warning indicators
- Flash-crash style coupling drops to study abrupt desynchronization
- Finite-size scaling data and phase-diagram visualization
- Hysteresis analysis to verify explosive synchronization
- A minimal test suite and CI build for reliability

## Model overview

Networks:

- ER (Erdos-Renyi): each pair of nodes connects with probability $p$
- BA (Barabasi-Albert): preferential attachment with $m$ edges per new node

Kuramoto dynamics (phases $\theta_i$):

$$
\dot{\theta}_i = \omega_i + \frac{K}{d_i} \sum_{j \in \mathcal{N}_i} \sin(\theta_j - \theta_i)
$$

Stuart-Landau dynamics (complex states $z_i$):

$$
\dot{z}_i = (\alpha + i \omega_i - |z_i|^2) z_i + \frac{K}{d_i} \sum_{j \in \mathcal{N}_i} (z_j - z_i)
$$

Order parameter (coherence):

$$
R(t) = \left| \frac{1}{N} \sum_{j=1}^N e^{i\theta_j} \right|
$$

The Python implementations match the C++ coupling normalization ($K/d_i$), so results are consistent across both layers.

## Repository layout

```
explosive-sync-flash-crash/
├── README.md
├── cpp/
│   ├── CMakeLists.txt
│   ├── src/
│   │   ├── network_gen.cpp/hpp
│   │   ├── kuramoto.cpp/hpp
│   │   ├── stuart_landau.cpp/hpp
│   │   ├── flash_crash.cpp/hpp
│   │   ├── order_param.cpp/hpp
│   │   └── main.cpp
│   └── include/
├── python/
│   ├── analysis/
│   │   ├── generate_networks.py
│   │   ├── run_kuramoto.py
│   │   ├── run_stuart_landau.py
│   │   ├── finite_size_scaling.py
│   │   ├── early_warning.py
│   │   ├── flash_crash_sim.py
│   │   ├── hysteresis_sweep.py
│   │   ├── lyapunov_analysis.py
│   │   └── percolation_analysis.py
│   ├── plots/
│   │   ├── plot_order_param.py
│   │   ├── plot_finite_size.py
│   │   ├── plot_early_warning.py
│   │   ├── plot_phase_diagram.py
│   │   ├── plot_flash_crash.py
│   │   ├── plot_degree_distribution.py
│   │   └── plot_percolation.py
│   └── utils/
│       └── helpers.py
├── data/
└── figures/
```

## Setup

Requirements:

- Python 3.10+
- CMake 3.15+ and a C++17 compiler

Install Python dependencies:

```
pip install -r requirements.txt
```

## End-to-end workflow

1) Run a simulation to generate an order-parameter time series.

```
python python/analysis/run_kuramoto.py --model er --n 200 --p 0.05 --k 2.0 --tmax 50 --dt 0.05
```

For explosive synchronization on BA networks, use degree-weighted frequencies (e.g., `--freq-mode degree-weighted --omega-mean 1.0`).

2) Compute early warning indicators from the saved series.

```
python python/analysis/early_warning.py --input data/kuramoto_order.csv --window 50
```

3) Plot both series and indicators.

```
python python/plots/plot_order_param.py --input data/kuramoto_order.csv --out figures/order_param.png
python python/plots/plot_early_warning.py --input data/early_warning.csv --out figures/early_warning.png
```

4) For phase-diagram style summaries, sweep sizes and couplings.

```
python python/analysis/finite_size_scaling.py --n-list 50,100,200,400,800 --k-list 0.0,0.5,1.0,1.5,2.0 --n-replicas 5
python python/plots/plot_finite_size.py --input data/finite_size.csv --out figures/finite_size.png
python python/plots/plot_phase_diagram.py --input data/finite_size.csv --out figures/phase_diagram.png
```

5) For a flash-crash scenario (abrupt coupling drop):

```
python python/analysis/flash_crash_sim.py --model er --n 200 --p 0.05 --k-high 2.5 --k-low 0.3 --t-drop 20
python python/plots/plot_flash_crash.py --input data/flash_crash.csv --out figures/flash_crash.png
```

## Plot catalog and interpretation

- Order parameter time series: shows growth or decay of coherence; sharp jumps signal explosive synchronization.
- Flash-crash response: shows how coherence collapses after a coupling drop and how quickly it recovers.
- Early warning indicators: rising variance and lag-1 autocorrelation can signal approaching transitions.
- Finite-size scaling: compares steady-state order across $N$ and $K$ to locate sharp transitions.
- Phase diagram: heatmap of steady order parameter across system size and coupling.
- Degree distribution: highlights structural differences between ER and BA networks.
- Percolation: identifies the emergence of the giant connected component in ER graphs.

## Benchmarking and performance

Runtime depends on $N$, integration step size, and network density. Typical timings for a standard scenario ($N=400$, ER network $p=0.05$, $K=2.0$, $T=50$):

| Implementation | Runtime |
| --- | --- |
| C++ (O3) | ~0.15s |
| Python (SciPy solve_ivp) | ~4.50s |

To benchmark locally, time the analysis and plotting steps separately:

```
/usr/bin/time -v python python/analysis/run_kuramoto.py --model er --n 400 --p 0.05 --k 2.0 --tmax 50 --dt 0.05
/usr/bin/time -v python python/plots/plot_finite_size.py --input data/finite_size.csv --out figures/finite_size.png
```

For deeper profiling:

```
python -m cProfile -o profile.stats python/analysis/run_kuramoto.py --model er --n 400 --p 0.05 --k 2.0 --tmax 50 --dt 0.05
```

The C++ CLI is designed for faster parameter sweeps; use the same timing approach:

```
/usr/bin/time -v ./cpp/build/esfc --system kuramoto --network er --n 400 --p 0.05 --k 2.0 --tmax 50 --dt 0.05 --out data/order_param.csv
```

## C++ CLI

Build:

```
cmake -S cpp -B cpp/build
cmake --build cpp/build
```

Run:

```
./cpp/build/esfc --system kuramoto --network er --n 200 --p 0.05 --k 2.0 --tmax 50 --dt 0.05 --out data/order_param.csv
```

Flash-crash scenario (coupling drop):

```
./cpp/build/esfc --system flash-crash --network er --n 200 --p 0.05 --k-high 2.5 --k-low 0.3 --t-drop 20 --tmax 50 --dt 0.05 --out data/flash_crash.csv
```

Options:

- `--system {kuramoto|stuart-landau|flash-crash}`
- `--network {er|ba}`
- `--n`, `--p`, `--m`, `--k`
- `--freq-mode {gaussian|degree-weighted}`
- `--k-high`, `--k-low`, `--t-drop` (Flash-crash)
- `--tmax`, `--dt`, `--seed`
- `--omega-mean`, `--omega-std`
- `--alpha` (Stuart-Landau)
- `--out` for the output CSV
- `--version` to print the CLI version

## Data formats

- CSV time series: `t,order`
- Early warning CSV: `t,order,variance,autocorr`
- Finite-size CSV: `n,k,order_mean,order_std`
- Networks saved as `*.npy` adjacency matrices and `*_degrees.csv`

## Tests and CI

Run tests locally:

```
make test
```

CI runs the test suite, linting (Ruff), type checking (Mypy), and builds the C++ CLI on GitHub Actions across Python 3.10, 3.11, and 3.12.

## Reproducibility

All scripts accept `--seed` to reproduce networks and initial conditions. For sweeps, vary the seed or add replicate loops to estimate variability.

**Note on RNG Incompatibility**: Python uses NumPy's PCG64 RNG while the C++ CLI uses `std::mt19937`. Identical seeds will **not** generate identical networks across languages. For cross-language comparisons, prefer exporting a shared network representation (e.g., Python writes an adjacency `.npy` or an edge-list CSV) and add a C++ reader for that format.

## Data and figures

`data/` and `figures/` directories are used to store generated outputs. We provide a `make data` target to regenerate all standard analysis data files. Please avoid committing generated CSVs or figures to the repository.

## Citations

1. Kuramoto, Y. (1984). *Chemical Oscillations, Waves, and Turbulence*. Springer Berlin Heidelberg.
2. Gómez-Gardeñes, J., Gómez, S., Arenas, A., & Moreno, Y. (2011). *Explosive Synchronization Transitions in Scale-Free Networks*. Physical Review Letters, 106(12), 128701.
3. Barabási, A.-L., & Albert, R. (1999). *Emergence of Scaling in Random Networks*. Science, 286(5439), 509-512.
4. Scheffer, M., et al. (2009). *Early-warning signals for critical transitions*. Nature, 461, 53-59.
