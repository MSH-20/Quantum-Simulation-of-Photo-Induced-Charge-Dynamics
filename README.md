# Quantum Simulation of Photo-Induced Charge Dynamics

A VQE–QSE workflow for the ground and excited states of ethylene (C₂H₄), extended to a toy donor–acceptor model of photo-induced charge transfer. Built with Qiskit and PySCF.

The project asks a simple question: can a variational quantum workflow reproduce the π → π* excitation of ethylene, and can its results feed a charge-transfer model like those used in photosynthesis and solar-energy research? The full write-up is in [`Report.pdf`](Report.pdf).

## Pipeline

```
Molecule (PySCF) → Active space (2e, 2o) → Qubit mapping → VQE ground state
      → QSE excited states → Noise + error mitigation → Classical benchmarks
      → Dimer coupling → Marcus rate → Lindblad kinetics → charge-separation yield
```

## Key results

| Topic | Result |
|---|---|
| VQE ground state (STO-3G, 2e/2o) | UCCSD matches the exact active-space energy, −77.116593 Ha |
| π → π* excitation, STO-3G | 14.155 eV (reference: 7.6 eV) |
| π → π* excitation, aug-cc-pVDZ | **7.565 eV**, only 0.035 eV from the reference |
| 90° twisted geometry (STO-3G) | Low-lying excitations drop to 0.084, 3.247 and 9.401 eV |
| Simulated noise (1×) | 177.20 mHa error, reduced to 94.04 mHa with ZNE (about 47% recovered) |
| Real hardware (`ibm_fez`, one job) | 1.57 mHa absolute energy error with resilience level 2 |
| Donor–acceptor toy model | J = 8.559 meV, k_ET ≈ 1.0 × 10¹² s⁻¹, 99.16% charge-separation yield |

### Ansatz comparison (equilibrium geometry)

| Ansatz | Qubits | Parameters | Depth | Error vs exact |
|---|---|---|---|---|
| UCCSD | 4 | 3 | 112 | 0.0000 mHa |
| EfficientSU2 (reps=2) | 4 | 24 | 12 | 0.1853 mHa |
| RealAmplitudes (reps=2) | 4 | 12 | 9 | 0.1056 mHa |

Parity mapping with two-qubit tapering reduces the problem from 4 to 2 qubits.

### Classical benchmarks (STO-3G)

| Method | Ground energy (Ha) | S0 → S1 (eV) |
|---|---|---|
| CASCI(2,2) | −77.116593 | 14.155 |
| CASSCF(2,2), state-averaged | −77.116475 | 14.142 |
| EOM-CCSD | −77.234131 | 11.609 |

## Repository structure

```
.
├── Classical/
│   └── CASCI_CASSCF_EOM-CCSD.ipynb      # Classical reference calculations (PySCF)
├── Quantum/
│   ├── ethylene_vqe_qse.ipynb           # Main workflow: VQE, QSE, geometry/basis study, IBM hardware run
│   ├── ethylene_vqe_qse with noise.ipynb # Noise model, ZNE, readout error mitigation
│   └── toy_demonstration.ipynb          # Donor–acceptor dimer: coupling → Marcus → Lindblad
└── Report.pdf                           # Short report with methods, results and references
```

## Getting started

Each notebook installs its own dependencies in the first cell:

```bash
pip install qiskit qiskit-aer qiskit-nature qiskit-algorithms pyscf matplotlib pandas
```

The hardware section of `ethylene_vqe_qse.ipynb` also needs:

```bash
pip install qiskit-ibm-runtime
```

and a saved IBM Quantum account (`QiskitRuntimeService`). Skip those last cells if you only want simulator results. The hardware job can wait in the IBM queue for hours.

Run the notebooks top to bottom in Jupyter or Google Colab. `ethylene_vqe_qse.ipynb` must run before its hardware cell, because the cell reuses the optimized UCCSD parameters.

## Methods

**Molecule and active space.** Ethylene at its planar equilibrium geometry and at a 90° twist (the two H atoms on one carbon rotated about the C–C axis). The active space is 2 electrons in 2 spatial orbitals, the π and π* frontier orbitals. Basis sets: STO-3G and aug-cc-pVDZ.

**VQE.** Statevector VQE on top of a Hartree–Fock reference, with UCCSD (SLSQP) and hardware-efficient ansätze (COBYLA).

**QSE.** Single and double excitation operators act on the optimized VQE state, `|ψᵢ⟩ = Oᵢ|ψ₀⟩`. The generalized eigenvalue problem `HC = SCE` is solved in this non-orthogonal subspace. Spin multiplicity is checked with ⟨S²⟩ to label singlets and triplets.

**Noise and mitigation.** The simulated noise model uses depolarizing errors (0.1% one-qubit, 1% two-qubit) and 2% readout error, with 8000 shots. ZNE scales the error rates from 1× to 3× and extrapolates linearly to zero noise. Readout mitigation inverts a calibrated 2-qubit confusion matrix.

**Charge-transfer extension.** Two ethylene molecules in a face-to-face stack (3.6 Å) form a minimal donor–acceptor pair with a (4e, 4o) active space. QEOM gives the excited-state splitting, which sets the electronic coupling J. The Marcus rate uses J with λ = 0.25 eV at 298 K:

```
k_ET = (2π/ħ) |J|² (4πλk_BT)^(-1/2) exp[ -(ΔG + λ)² / (4λk_BT) ]
```

A 4-level Lindblad model (donor, acceptor, charge-transfer, ground) with loss and dephasing then gives the charge-separation yield.

## Limitations

- The ethylene dimer is a toy stand-in for a real chromophore pair such as chlorophyll. Parameters like λ, ΔG and the loss and dephasing rates are illustrative, not fitted.
- STO-3G is a minimal basis, which is why its excitation energy is far from experiment. The (2,2) active space also leaves out dynamic correlation.
- ZNE results come from a simulated noise model, and the hardware result is a single energy evaluation, not a statistical study.

## References

1. A. Peruzzo et al., "A variational eigenvalue solver on a quantum processor," *Nature Communications* 5, 4213 (2014).
2. P. F. Kwao, S. P. Sundar, B. Gupt, A. Asthana, "Generalized Eigenvalue Problem in Subspace-Based Excited-State Methods for Quantum Computers," *J. Chem. Theory Comput.* 22(6), 2892–2903 (2026). https://doi.org/10.1021/acs.jctc.5c02010
3. K. Temme, S. Bravyi, J. M. Gambetta, "Error Mitigation for Short-Depth Quantum Circuits," *Phys. Rev. Lett.* 119, 180509 (2017).
4. S. Bravyi et al., "Mitigating measurement errors in multiqubit experiments," *Phys. Rev. A* 103, 042605 (2021).
5. R. A. Marcus, "On the Theory of Oxidation-Reduction Reactions Involving Electron Transfer. I," *J. Chem. Phys.* 24(5), 966–978 (1956).
6. Challenge specification, "Quantum Simulation of Photo-Induced Charge Dynamics: From Ethylene to Photosynthesis," Track 1.
