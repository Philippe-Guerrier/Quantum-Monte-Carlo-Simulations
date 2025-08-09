# Quantum Monte Carlo Simulations — Estimating π with Qiskit

This project demonstrates a **Monte Carlo estimation of π** using **quantum-generated pseudo-random numbers** built from simple Qiskit circuits. The notebook sweeps a parameterized circuit to sample points `(x, y)` in `[0, 1]^2`, counts how many fall inside the unit quarter-circle, and estimates π via `π ≈ 4 * inside / total`.

The code runs on the **Qiskit Aer** simulator and includes multiple circuit sizes (`nqubits ∈ {3, 5, 6}`), execution styles (`execute` and `backend.run`), and plots (histograms, Bloch vectors, scatter plots with a reference circle, and convergence traces).

---

## Overview

- **Goal**: Use quantum circuits as a randomness source for a classical Monte Carlo estimator of π.
- **Idea**: Prepare qubits in superposition and light entanglement, measure bitstrings, and map them to `[0, 1]` real numbers by normalizing the integer value of the measured string (base-2) with `2^nqubits`.
- **Notebook**: `Quantum Monte Carlo Simulations Pi Complete.ipynb` runs end-to-end and produces intermediate visualizations and the final π estimate.

---

## Method

### Quantum Random Number Generation (QRNG)
1. Start with `nqubits` qubits in `|0⟩`.
2. Apply `H` gates to create superposition and `CZ` entangling gates for correlation:
   - Example pattern for 3 qubits:
     ```
     H on all qubits → RX(θ) on all qubits → CZ(0,1), CZ(1,2) → optional final H on all qubits → measure
     ```
3. Parameterize a rotation `RX(θ)` and **sweep θ** during the sampling loop to decorrelate samples across iterations.
4. Measure once per coordinate (`shots=1`) to get bitstrings for `x` and `y` independently.

### Mapping Bitstrings to [0, 1]
- Convert the measured bitstring (e.g., `"101"`) to an integer, divide by `2^nqubits`, and treat as `x` (repeat for `y`).

### Monte Carlo Estimator
- For each iteration `i`:
  - Sample `x` from a bound circuit with `θ = π*i/num_points`.
  - Sample `y` from a bound circuit with `θ = π*(i+0.5)/num_points`.
  - If `x^2 + y^2 ≤ 1`, increment `inside`.
  - Append `π_i = 4 * inside / (i+1)` to a trace for plotting convergence.

---

## Repository Structure

```
Quantum-Monte-Carlo-Simulations/
├─ Quantum Monte Carlo Simulations Pi Complete.ipynb   # main notebook
└─ README.md
```

---

## Requirements

- Python ≥ 3.9
- Qiskit (Aer + Terra)
- NumPy, Matplotlib

Install (legacy API compatible with the notebook as written):

```bash
pip install "qiskit==0.43.*" "qiskit-aer==0.12.*" numpy matplotlib
```

If you prefer Qiskit ≥ 1.0, see the execution note below.

---

## How to Run

### Option A — Jupyter
1. Create and activate a virtual environment (optional).
2. Install dependencies.
3. Open the notebook and run all cells. The notebook prints intermediate π estimates and shows plots (histogram, Bloch multivector, scatter with circle, convergence curve).

### Option B — Script (Optional)
You can extract the logic into a Python script that:
- Builds the base circuit for a chosen `nqubits`.
- Sweeps `θ` across iterations.
- Samples `x` and `y`, updates the running π estimate.
- Produces the final scatter and convergence plot.

---

## Key Parameters and Knobs

- `nqubits`: number of qubits (resolution of mapping to `[0,1]` is `1/2^nqubits`). The notebook explores 3, 5, and 6 qubits.
- `num_points`: number of Monte Carlo samples (larger is better but slower).
- `shots`: fixed at `1` per sample to maximize throughput.
- `θ` schedule: uses two phase offsets per iteration (`i` and `i+0.5`) for `x` and `y` to reduce temporal correlation.
- `max_time`: optional wall-time cap in one variant (e.g., 6 hours) to stop long runs automatically.

---

## Example Snippets

### Circuit Template with Parameter
```python
from qiskit import QuantumCircuit
from qiskit.circuit import Parameter

nqubits = 3
theta = Parameter("θ")
base_qc = QuantumCircuit(nqubits, nqubits)

# Superposition
base_qc.h(range(nqubits))

# Parameterized rotation
base_qc.rx(theta, range(nqubits))

# Light entanglement (CZ chain)
for i in range(nqubits-1):
    base_qc.cz(i, i+1)

# Optional: extra H to mix further
# base_qc.h(range(nqubits))

# Measurement
base_qc.measure(range(nqubits), range(nqubits))
```

### Mapping a Measurement to [0, 1]
```python
def bitstring_to_unit_interval(counts, nqubits):
    # counts is a dict like {"101": 1}
    key = next(iter(counts))
    return int(key, 2) / (2**nqubits)
```

### Legacy Execute API (as in the notebook)
```python
from qiskit import Aer, execute
from qiskit.visualization import plot_histogram

backend = Aer.get_backend("qasm_simulator")
qc_bound = base_qc.bind_parameters({theta: 1.234})
job = execute(qc_bound, backend, shots=1)
result = job.result()
counts = result.get_counts(qc_bound)
x = bitstring_to_unit_interval(counts, nqubits)
```

### Qiskit ≥ 1.0 API
```python
from qiskit_aer import AerSimulator
from qiskit import transpile

backend = AerSimulator()

def run_counts(circuit, backend, shots=1):
    compiled = transpile(circuit, backend=backend)
    job = backend.run(compiled, shots=shots)
    return job.result().get_counts()

qc_bound = base_qc.bind_parameters({theta: 1.234})
counts = run_counts(qc_bound, backend, shots=1)
x = bitstring_to_unit_interval(counts, nqubits)
```

---

## Results and Plots

The notebook prints running estimates and final values of π. Example outputs observed during runs:
- `Final estimated value of Pi: 3.50088` (small `nqubits`, modest samples)
- `Final estimated value of Pi: 3.2076` (more qubits, limited samples with time cap)
- `Final estimated value of Pi: 3.104` (fewer samples)

Plots include:
- **Convergence trace** of the running π estimate.
- **Scatter** of sampled points with a reference quarter-circle overlay.
- **Measurement histogram** for selected circuits.
- **Bloch multivector** visualization of the statevector (when using `BasicAer` statevector simulator on a circuit without measurement).

---

## Limitations and Notes

- **Sampling variance**: Monte Carlo convergence is `O(1/√N)`. Expect noisy estimates for small `num_points`.
- **Resolution**: With `nqubits` qubits, the finest grid step in `[0,1]` is `1/2^nqubits`. Higher `nqubits` increases resolution but also circuit width and transpilation time.
- **Circuit bias**: The measurement distribution depends on circuit design (gates, entanglement, θ schedule). The approach is **illustrative**, not a certified QRNG.
- **Performance**: Using one shot per sample reduces overhead; increasing `shots` lowers randomness variance per sample but slows execution.
- **Hardware vs simulator**: Real devices introduce noise (decoherence, SPAM, gate errors). The notebook targets the simulator for reproducibility.

---

## Acknowledgements

- Qiskit (IBM Quantum) for quantum circuit simulation.
- Matplotlib for plotting.
