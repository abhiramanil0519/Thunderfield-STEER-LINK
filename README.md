# STEER-LINK
### Adaptive Quantum Circuit for PCM Telemetry Decoding

STEER-LINK is a six-qubit quantum receiver architecture designed to decode 6-bit Pulse Code Modulation (PCM) telemetry words using adaptive quantum operations, mid-circuit measurements, and classical feed-forward.

The architecture combines nearest-neighbour CNOT gates with measurement-conditioned \(R_y(\theta)\) rotations. Rotation angles are selected from a classical lookup table (ROM), allowing subsequent quantum operations to depend on previously measured bits.

The project implements a Qiskit-based receiver, a NumPy reference model, ideal and noisy simulation workflows, IBM Quantum hardware execution, and performance visualization.

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Proposed Architecture](#proposed-architecture)
- [Circuit Diagram](#circuit-diagram)
- [Level 1: STEER-LINK Cell](#level-1-steer-link-cell)
- [Level 2: Complete Receiver](#level-2-complete-receiver)
- [Adaptive Angle ROM](#adaptive-angle-rom)
- [Classical Channel Model](#classical-channel-model)
- [Circuit Resources](#circuit-resources)
- [Technology Stack](#technology-stack)
- [Installation](#installation)
- [Configuration](#configuration)
- [Execution](#execution)
- [Performance Evaluation](#performance-evaluation)
- [Results and Analysis](#results-and-analysis)
- [Output Files](#output-files)
- [Limitations](#limitations)
- [Future Improvements](#future-improvements)
- [Acknowledgments and Third-Party Resources](#acknowledgments-and-third-party-resources)
- [References](#references)

## Overview

Deep-space probes and remote sensing systems transmit telemetry over communication channels that may have limited signal energy and noisy observations. Efficient reconstruction of digital telemetry is therefore an important receiver-design problem.

STEER-LINK investigates whether adaptive quantum processing can improve the decoding of six-bit residual words.

The residual word is defined as

\[
e_t = w_t \oplus w_{t-1}
\]

where:

- \(w_t\) is the current six-bit telemetry word.
- \(w_{t-1}\) is the previous telemetry word.
- \(\oplus\) denotes bitwise XOR.
- \(e_t\) is the residual word processed by the receiver.

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

A reusable cell performs a nearest-neighbour CNOT operation, selects an angle from the ROM, conditionally applies an \(R_y(\theta)\) rotation, and measures the current qubit.

**Level 2 — Complete Receiver**

Six cells are chained after a six-qubit Hadamard layer. The cells process qubits in the order \(q_5\) through \(q_0\), using earlier measurement results to control subsequent rotations.

The complete simulated experiment also includes preparation of the received quantum state.

## Circuit Diagram

The following diagram illustrates the two-level STEER-LINK architecture, including the CNOT link, adaptive rotation, mid-circuit measurement, classical feed-forward, and six-cell receiver.

<p align="center">
  <img src="circuit.png" width="80%" alt="STEER-LINK two-level quantum circuit architecture">
</p>

## Level 1: STEER-LINK Cell

Each cell combines three principal operations.

### 1. CNOT Link

A Controlled-NOT gate couples neighbouring qubits:

\[
\operatorname{CNOT}(q_i,q_{i-1})
\]

The operation introduces a conditional interaction between adjacent qubits when a lower-indexed neighbour exists.

### 2. Adaptive Rotation

The cell applies the rotation

\[
R_y(\theta)=
\begin{bmatrix}
\cos(\theta/2) & -\sin(\theta/2)\\
\sin(\theta/2) & \cos(\theta/2)
\end{bmatrix}
\]

The angle is selected from the classical lookup table based on the measurement history available to that cell.

### 3. Mid-Circuit Measurement

The current qubit is measured before the entire circuit finishes. The resulting classical bit becomes available to subsequent operations.

### Classical Feed-Forward

The measured bits form a growing prefix. The next cell uses the prefix to select the appropriate rotation angle from the ROM.

This creates an adaptive sequence in which later quantum operations depend on previously obtained measurement results.

## Level 2: Complete Receiver

### Received Quantum State

The implementation constructs a six-qubit state representing the received residual word:

\[
|\xi_e\rangle = Z^e |s\rangle
\]

Here, \(|s\rangle\) is the reference amplitude state and \(Z^e\) represents the phase operations determined by the residual word.

The state is prepared mathematically using a quantum circuit. This emulates the received state rather than implementing a physical optical receiver.

### Hadamard Layer

A Hadamard gate is applied to each of the six qubits:

\[
H^{\otimes 6}
\]

This provides the initial basis transformation for the receiver.

### Chained Adaptive Cells

The six cells process the qubits in the order

\[
q_5 \rightarrow q_4 \rightarrow q_3
\rightarrow q_2 \rightarrow q_1 \rightarrow q_0
\]

Each cell measures one qubit and updates the classical information available to later cells.

### PCM Word Reconstruction

After all six measurements, the receiver produces a six-bit classical output. A classical decoder interprets the measurement distribution to estimate the transmitted residual word.

## Adaptive Angle ROM

The receiver uses a classical lookup table containing 63 entries.

At cell \(k\), there are \(2^k\) possible measurement prefixes. Therefore,

\[
N_{\mathrm{ROM}}
=\sum_{k=0}^{5}2^k
=1+2+4+8+16+32
=63
\]

The ROM stores the rotation angle associated with each possible measurement history.

The implementation includes angle tables for the following mean photon levels per code symbol:

| Mean photons per code symbol | Photons per bit |
|---:|---:|
| 0.04 | 0.2133 |
| 0.06 | 0.3200 |
| 0.08 | 0.4267 |

The conversion is

\[
N_{\mathrm{photons/bit}}=\bar n\frac{32}{6}
\]

Angles below the configured pruning threshold can be set to zero, reducing the number of nonzero conditional rotations at the potential cost of decoding accuracy.

## Classical Channel Model

The implementation uses NumPy to simulate the telemetry signal and estimate residual-word statistics.

### Simulated Telemetry

The generated signal contains two periodic components and Gaussian noise:

\[
x_t =
31.5
+28\sin\left(\frac{2\pi t}{300}\right)
+6\sin\left(\frac{2\pi t}{47}\right)
+\epsilon_t
\]

where \(\epsilon_t\) is Gaussian noise.

The signal is rounded and clipped to the range from 0 to
