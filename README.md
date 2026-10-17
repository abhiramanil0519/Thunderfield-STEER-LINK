# STEER-LINK
### Adaptive Quantum Circuit for PCM Telemetry Decoding

STEER-LINK is a six-qubit quantum receiver architecture designed to decode 6-bit Pulse Code Modulation (PCM) telemetry words using adaptive quantum operations, mid-circuit measurements, and classical feed-forward.

The architecture combines nearest-neighbour CNOT gates with measurement-conditioned $R_y(\theta)$ rotations. Rotation angles are selected from a classical lookup table (ROM), allowing subsequent quantum operations to depend on previously measured bits.

The project implements a Qiskit-based receiver, a NumPy reference model, ideal and noisy simulation workflows, IBM Quantum hardware execution, and performance visualization.

## Overview

Deep-space probes and remote sensing systems transmit telemetry over communication channels that may have limited signal energy and noisy observations. Efficient reconstruction of digital telemetry is therefore an important receiver-design problem.

STEER-LINK investigates whether adaptive quantum processing can improve the decoding of six-bit residual words.

The residual word is defined as

$$
e_t = w_t \oplus w_{t-1}
$$

where:

- $w_t$ is the current six-bit telemetry word.
- $w_{t-1}$ is the previous telemetry word.
- $\oplus$ denotes bitwise XOR.
- $e_t$ is the residual word processed by the receiver.

For slowly varying telemetry, consecutive words may have structured residual distributions. The implementation estimates these statistics from simulated sensor data and uses the learned distribution as a prior for decoding.

STEER-LINK is evaluated against a conventional Hadamard-based receiver called the Green Machine.

## Problem Statement

The objective is to investigate a quantum receiver that uses local entangling operations, adaptive single-qubit rotations, and intermediate classical measurements to decode six-bit telemetry residuals.

The experiment evaluates:

1. Ideal decoding performance at different photon levels.
2. The number of active rotation angles after ROM pruning.
3. Ideal predictions compared with saved experimental results.
4. The effects of noise on decoding performance.
5. The resource requirements of the complete circuit.

The project provides an experimental framework for investigating these questions. It does not assume that STEER-LINK outperforms the baseline under all conditions.

## Proposed Architecture

STEER-LINK consists of two levels.

**Level 1 — STEER-LINK Cell**

A reusable cell performs a nearest-neighbour CNOT operation, selects an angle from the ROM, conditionally applies an $R_y(\theta)$ rotation, and measures the current qubit.

**Level 2 — Complete Receiver**

Six cells are chained after a six-qubit Hadamard layer. The cells process qubits in the order $q_5$ through $q_0$, using earlier measurement results to control subsequent rotations.

The complete simulated experiment also includes preparation of the received quantum state.

## Circuit Diagram

The diagram illustrates the two-level STEER-LINK architecture, including the CNOT link, adaptive rotation, mid-circuit measurement, classical feed-forward, and six-cell receiver.

<p align="center">
  <img src="circuit.png" width="80%" alt="STEER-LINK two-level quantum circuit architecture">
</p>

## Level 1: STEER-LINK Cell

Each cell combines three principal operations.

### 1. CNOT Link

A Controlled-NOT gate couples neighbouring qubits:

$$
\operatorname{CNOT}(q_i,q_{i-1})
$$

The operation introduces a conditional interaction between adjacent qubits when a lower-indexed neighbour exists.

### 2. Adaptive Rotation

The cell applies the rotation

$$
R_y(\theta)=
\begin{bmatrix}
\cos(\theta/2) & -\sin(\theta/2)\\
\sin(\theta/2) & \cos(\theta/2)
\end{bmatrix}
$$

The angle is selected from the classical lookup table based on the measurement history available to that cell.

### 3. Mid-Circuit Measurement

The current qubit is measured before the entire circuit finishes. The resulting classical bit becomes available to subsequent operations.

### Classical Feed-Forward

The measured bits form a growing prefix. The next cell uses the prefix to select the appropriate rotation angle from the ROM.

This creates an adaptive sequence in which later quantum operations depend on previously obtained measurement results.

## Level 2: Complete Receiver

### Received Quantum State

The implementation constructs a six-qubit state representing the received residual word:

$$
|\xi_e\rangle = Z^e |s\rangle
$$

Here, $|s\rangle$ is the reference amplitude state and $Z^e$ represents the phase operations determined by the residual word.

The state is prepared mathematically using a quantum circuit. This emulates the received state rather than implementing a physical optical receiver.

### Hadamard Layer

A Hadamard gate is applied to each of the six qubits:

$$
H^{\otimes 6}
$$

This provides the initial basis transformation for the receiver.

### Chained Adaptive Cells

The six cells process the qubits in the order

$$
q_5 \rightarrow q_4 \rightarrow q_3
\rightarrow q_2 \rightarrow q_1 \rightarrow q_0
$$

Each cell measures one qubit and updates the classical information available to later cells.

### PCM Word Reconstruction

After all six measurements, the receiver produces a six-bit classical output. A classical decoder interprets the measurement distribution to estimate the transmitted residual word.

## Adaptive Angle ROM

The receiver uses a classical lookup table containing 63 entries.

At cell $k$, there are $2^k$ possible measurement prefixes. Therefore,

$$
N_{\mathrm{ROM}}
=\sum_{k=0}^{5}2^k
=1+2+4+8+16+32
=63
$$

The ROM stores the rotation angle associated with each possible measurement history.

The implementation includes angle tables for the following mean photon levels per code symbol:

| Mean photons per code symbol | Photons per bit |
|---:|---:|
| 0.04 | 0.2133 |
| 0.06 | 0.3200 |
| 0.08 | 0.4267 |

The conversion is

$$
N_{\mathrm{photons/bit}}=\bar n\frac{32}{6}
$$

Angles below the configured pruning threshold can be set to zero, reducing the number of nonzero conditional rotations at the potential cost of decoding accuracy.

## Classical Channel Model

The implementation uses NumPy to simulate the telemetry signal and estimate residual-word statistics.

### Simulated Telemetry

The generated signal contains two periodic components and Gaussian noise:

$$
x_t =
31.5
+28\sin\left(\frac{2\pi t}{300}\right)
+6\sin\left(\frac{2\pi t}{47}\right)
+\epsilon_t
$$

where $\epsilon_t$ is Gaussian noise.

The signal is rounded and clipped to the range from 0 to 63 before conversion into six-bit words.

### Residual Generation

Residual words are generated by XORing consecutive telemetry words. The resulting sequence is divided into training and testing segments.

- **Training segment:** Estimates the residual-word prior.
- **Testing segment:** Evaluates decoding performance on held-out residuals.

Smoothing is applied to the estimated distribution to avoid zero-probability entries.

This is a synthetic channel model. The results are specific to the modeled telemetry statistics and are not measurements from an actual deep-space mission.

### Ideal Reference Model

A NumPy reference model calculates ideal measurement distributions for both the Green Machine and STEER-LINK receivers. These distributions are used to evaluate ideal Word Error Rate (WER) and construct the corresponding decoders.

## Circuit Resources

The nominal receiver architecture contains the following operations:

| Component | Count |
|---|---:|
| Qubits | 6 |
| Hadamard gates | 6 |
| Nearest-neighbour CNOT gates | 5 |
| Adaptive rotation stages | 6 |
| Measurements | 6 |
| Classical angle ROM entries | 63 |

Each adaptive stage can contain multiple conditional rotation instructions, depending on the number of possible measurement prefixes and nonzero angles.

The complete experiment also includes received-state preparation. Consequently, the resource counts above describe the nominal receiver structure, not the total gate count of the complete circuit.

Actual circuit depth, gate counts, physical qubit layout, and conditional-operation counts should be obtained from the transpiled circuits for the selected backend.

## Technology Stack

| Technology | Purpose |
|---|---|
| Python | Main implementation |
| Qiskit | Quantum circuit construction |
| Qiskit Aer | Local ideal and noisy simulation |
| Qiskit IBM Runtime | IBM Quantum backend access and execution |
| NumPy | Numerical calculations and reference model |
| SciPy | Numerical optimization |
| Matplotlib | Circuit visualization and performance graphs |
| Pandas | Tabular result analysis |
| pylatexenc | LaTeX rendering support for circuit diagrams |
| Google Colab | Cloud notebook environment |

## Installation

### Google Colab

Run the following command in a code cell:

```python
%pip install -q qiskit qiskit-aer qiskit-ibm-runtime scipy numpy matplotlib pandas pylatexenc
```

Execute the installation cell before running the implementation.

If installation changes packages that have already been imported, restart the runtime and execute the notebook cells again.

## Configuration

For IBM Quantum hardware execution, configure the following variables in the Python implementation:

```python
IBM_API_KEY = "YOUR_IBM_QUANTUM_API_KEY"
IBM_INSTANCE_CRN = "YOUR_IBM_QUANTUM_INSTANCE_CRN"
IBM_BACKEND = "ibm_marrakesh"
```

Replace the placeholders with the actual API key and instance CRN associated with your IBM Quantum account.

The API key authenticates the account, while the CRN identifies the service instance.

Do not commit a real API key to a public GitHub repository. Keep credentials out of shared notebooks and version control.

A valid API key does not guarantee that a particular backend is accessible or that all required dynamic-circuit operations are supported.

## Execution

The implementation supports four experiment modes through its original command-line interface.

| Mode | Description | IBM credentials required |
|---|---|---|
| `ideal` | Ideal simulation | No |
| `noisy` | Noisy simulation using a fake backend model | No |
| `sweep` | Noise-sensitivity comparison | No |
| `hardware` | Execution through IBM Quantum Runtime | Yes |

### Ideal Simulation

Ideal simulation provides a reference without modeled hardware noise. It is useful for checking circuit logic and evaluating theoretical decoding performance.

### Noisy Simulation

Noisy simulation evaluates performance under a selected simulated backend noise model. Its results depend on the model and supported operations.

### Noise Sweep

The noise sweep varies modeled gate and measurement error rates to investigate how receiver performance changes with increasing noise.

It is a sensitivity analysis, not a calibration of a physical quantum processor.

### Hardware Execution

Hardware execution uses IBM Quantum Runtime to transpile and submit circuits to an available quantum processor.

Before submitting a job, verify the backend, authentication, circuit count, shot count, and available quantum-computing usage.

Hardware results can differ from ideal predictions because of gate errors, measurement errors, coherence limitations, transpilation, and feed-forward constraints.

The hardware path should be considered verified only after a job has successfully completed on the intended backend.

## Performance Evaluation

### Word Error Rate

Word Error Rate measures the probability that the decoded word differs from the transmitted residual word.

For the transmitted residual distribution $p(e)$, measurement distribution $P(n\mid e)$, and decoder $\hat e(n)$,

$$
\operatorname{WER}
=
\sum_e p(e)
\sum_n P(n\mid e)
\mathbf{1}\left[\hat e(n)\neq e\right]
$$

where $\mathbf{1}[\cdot]$ is the indicator function.

A lower WER indicates better decoding performance.

### Green Machine Baseline

The Green Machine applies six Hadamard gates followed by six measurements.

Two decoding strategies are evaluated:

- **Plain ML:** Uses the maximum-likelihood interpretation of the measured word.
- **MAP:** Incorporates the learned residual-word prior in addition to the measurement likelihood.

### STEER-LINK

STEER-LINK applies adaptive measurement-conditioned rotations and uses its ideal measurement model to construct the MAP decoder.

### Comparison Method

The experiment compares:

1. Ideal WER across photon levels.
2. Active ROM entries after angle pruning.
3. Ideal predictions against saved experimental results.
4. Decoding performance under modeled noise.
5. Circuit-resource requirements.

Comparisons should use consistent assumptions and account for the statistical uncertainty of finite-shot results.

## Results and Analysis

The following figure consolidates the receiver performance analysis generated by the notebook.

<p align="center">
  <img src="results.png" width="80%" alt="STEER-LINK performance analysis showing ideal decoding performance, angle ROM usage, and ideal versus saved hardware results">
</p>

### 1. Ideal Decoding Performance

The first graph compares the ideal Word Error Rate of three receiver configurations:

- Green Machine with plain ML decoding.
- Green Machine with MAP decoding.
- STEER-LINK with MAP decoding.

The horizontal axis represents photons per bit, and the vertical axis represents WER.

In the displayed ideal-model results, WER decreases as the photon level increases. STEER-LINK also has a lower ideal WER than the two Green Machine configurations at the plotted photon levels.

These values describe the ideal model, not actual hardware performance.

### 2. Angle ROM Usage

The second graph shows the number of nonzero rotation angles used by each STEER-LINK cell after pruning.

The displayed run uses **49 of the 63 available ROM entries**. The remaining entries have been set to zero by the pruning configuration.

The graph illustrates how active rotation instructions are distributed across the six cells. This helps identify which stages require the largest number of conditional rotation operations.

The number of active angles alone does not determine total execution cost; transpiled circuit depth and gate counts must also be considered.

### 3. Ideal vs Saved Hardware Run

The third graph compares the predicted ideal WER against the measured WER recorded in a saved hardware experiment.

In the displayed graph, the measured WER is substantially higher than the ideal prediction. STEER-LINK also has a higher measured WER than either Green Machine configuration in that run.

This result means the displayed hardware experiment does **not** demonstrate a practical STEER-LINK advantage over the baseline.

Possible contributors include gate and measurement errors, circuit transpilation overhead, limitations of dynamic-circuit feed-forward, and errors in received-state preparation. The graph alone cannot determine the contribution of each factor.

Further repeated experiments, noise-aware analysis, and verification of the transpiled circuits are necessary before drawing stronger conclusions.

The error bars represent the standard errors reported by the experiment. A rigorous comparison should examine the uncertainty of the difference between the receivers, rather than relying on the bar heights alone.

### Overall Interpretation

The results distinguish between theoretical performance and observed execution performance.

The ideal model suggests that adaptive decoding can improve WER for the modeled telemetry distribution. However, the displayed hardware run shows that this improvement does not automatically transfer to a noisy quantum processor.

The main experimental question is whether improvements in circuit implementation, noise management, and feed-forward execution can make the adaptive receiver competitive with the simpler baseline.

## Output Files

The experiment can save JSON files containing:

- Experiment configuration.
- Quantum job identifier.
- Selected residual words and their weights.
- Rotation-angle values.
- Transpiled circuit statistics.
- Ideal and measured WER results.
- Measurement counts for individual circuits.

The analysis notebook also saves a performance figure:

```text
steer_link_performance_summary.png
```

The filename of the experiment JSON depends on the configured output path and execution mode.

Keep the result files with the associated code version and configuration for reproducibility.

## Limitations

- The telemetry signal and residual distribution are simulated.
- The received quantum state is circuit-emulated rather than captured by a physical optical receiver.
- Ideal results do not represent actual hardware performance.
- Noisy simulation does not necessarily capture every physical-device error mechanism.
- Dynamic-circuit compatibility depends on the selected backend and runtime.
- Transpilation may increase resource requirements.
- ROM pruning can change decoding performance.
- Results depend on the residual prior, photon level, circuit implementation, shot count, and hardware conditions.
- A single finite-shot experiment cannot establish a general performance advantage.

## Future Improvements

Potential directions include:

1. **ROM optimization:** Improve angle training and quantify the accuracy-versus-resource trade-off of pruning.
2. **Noise-aware training:** Incorporate relevant noise characteristics into angle optimization.
3. **Hardware benchmarking:** Compare receivers on the same device under comparable execution conditions.
4. **Statistical evaluation:** Perform repeated trials and calculate confidence intervals for performance differences.
5. **Circuit optimization:** Investigate transpiled depth, two-qubit gate counts, and dynamic-circuit overhead.
6. **Real telemetry testing:** Validate the model against actual telemetry traces.
7. **Resource analysis:** Compare received-state preparation, measurement, feed-forward, and classical post-processing costs.

These are proposed directions for future investigation, not completed experimental results.

## Acknowledgments and Third-Party Resources

This project uses open-source libraries and their official documentation for quantum-circuit construction, simulation, numerical optimization, and visualization.

### Open-Source Libraries

| Library | Purpose | Official Resource |
|---|---|---|
| Qiskit | Quantum circuit construction and manipulation | [Documentation](https://docs.quantum.ibm.com/) |
| Qiskit Aer | Ideal and noisy circuit simulation | [Documentation](https://qiskit.github.io/qiskit-aer/) |
| Qiskit IBM Runtime | Quantum backend access and circuit execution | [Documentation](https://quantum.cloud.ibm.com/docs/en/guides) |
| NumPy | Numerical calculations | [Documentation](https://numpy.org/doc/) |
| SciPy | Numerical optimization | [Documentation](https://docs.scipy.org/doc/scipy/) |
| Matplotlib | Circuit and performance visualization | [Documentation](https://matplotlib.org/stable/) |
| Pandas | Tabular analysis | [Documentation](https://pandas.pydata.org/docs/) |
| pylatexenc | LaTeX rendering support | [Documentation](https://pylatexenc.readthedocs.io/) |

### External Code and Tutorials

The official resources listed above document the libraries and APIs used by the project. Any external source-code snippets or tutorial-derived implementations incorporated into the repository should be attributed to their original sources, with URLs and applicable license information.

The acknowledgment of a library does not imply authorship of that library or its underlying implementation. Project-specific code should be distinguished from third-party material.

### AI-Assisted Development

AI assistance was used during development to help draft, refine, and debug portions of the implementation and documentation. The project implementation and experimental results should be reviewed and validated by the project maintainers.

### Licensing and Attribution

Third-party libraries remain subject to their respective licenses and attribution requirements. Any externally sourced code incorporated into the repository should retain the required notices and attribution.

## References

- [Qiskit Documentation](https://docs.quantum.ibm.com/)
- [Qiskit Aer Documentation](https://qiskit.github.io/qiskit-aer/)
- [Qiskit IBM Runtime Documentation](https://quantum.cloud.ibm.com/docs/en/guides)
- [NumPy Documentation](https://numpy.org/doc/)
- [SciPy Documentation](https://docs.scipy.org/doc/)
- [Matplotlib Documentation](https://matplotlib.org/stable/)
- [Pandas Documentation](https://pandas.pydata.org/docs/)
- [pylatexenc Documentation](https://pylatexenc.readthedocs.io/)

## Project Summary

STEER-LINK explores an adaptive quantum-circuit approach to six-bit PCM telemetry residual decoding.

Its main mechanism combines nearest-neighbour CNOT interactions, measurement-conditioned $R_y(\theta)$ rotations, and classical feed-forward using a 63-entry angle ROM.

The project integrates a NumPy reference model, Qiskit circuit construction, simulation and hardware execution paths, and WER-based performance analysis.

The results demonstrate why ideal-model improvements must be evaluated against actual execution results before claiming a practical quantum-decoding advantage.
