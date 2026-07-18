# Wideband SVD and DM-RS Massive MIMO Simulink Testbench
## Project Status

**Status:** Completed executable testbench and verification framework  
**Implementation:** MATLAB and Simulink  
**Model type:** Level-2 MATLAB S-function orchestration testbench  
**System profile:** 64 transmit antennas, 8 receive antennas, 4 spatial layers  
**Author:** Md Moklesur Rahman  
**Location:** Oulu, Finland

# Executable Simulink Testbench for the 64 × 8 PCC-SVD Massive MIMO-OFDM Link-Level Simulator

## Project Overview

This repository contains the **Simulink integration, execution control, verification workflow, and model-based testbench** for a 64 × 8 Massive MIMO-OFDM downlink simulator using **Pilot-Compatibility-Constrained SVD (PCC-SVD)** precoding.
The testbench executes the verified MATLAB numerical engine from an active Level-2 MATLAB S-function. 
- launches the verified MATLAB engine from Simulink;
- reruns the complete deterministic verification suite;
- supports a finite smoke-test mode;
- supports a complete analysis mode;
- reports execution status and key metrics through a fixed six-element interface;
- logs results to the MATLAB workspace;
- rebuilds the Simulink model programmatically;
- supports deterministic MATLAB–Python shared-vector verification.


The Simulink model and the MATLAB numerical engine have been executed successfully. The reported verification and smoke-test values come from completed runs.

## Technical Purpose

Frequency-selective SVD precoding creates a trade-off between:

- dominant-subspace capture;
- frequency continuity of the effective channel;
- compatibility with sparse DM-RS-based channel estimation.

Per-subcarrier SVD can maximize local spatial gain but may produce rapid basis changes across frequency. These changes can make the effective channel difficult to reconstruct from sparse reference signals. Wideband SVD produces a smoother effective channel but can sacrifice local subspace gain.

PCC-SVD addresses this trade-off by selecting among aligned SVD candidates with bundle sizes:

```text
144 subcarriers
48 subcarriers
1 subcarrier
```

The selection uses:

- pilot-reconstruction NMSE;
- dominant-subspace capture;
- an SNR-adaptive reconstruction-error budget;
- a deterministic minimum-reconstruction-error fallback rule.

The Simulink testbench exists to make this verified numerical chain executable and observable in a model-based environment.

## Relationship to the Main Numerical Simulator

This repository is the **Simulink testbench layer** of the larger PCC-SVD Massive MIMO-OFDM project.

The complete transmitter, channel, precoding, estimation, equalization, and metric calculations remain in the verified MATLAB numerical engine.

The Simulink model:

- calls the MATLAB engine;
- does not reimplement all 64 × 8 matrix operations block by block;
- exposes testbench status and numerical outputs;
- guarantees that the model and the numerical engine use the same implementation.

This design avoids maintaining two independent implementations of the same physical-layer algorithm.

## System Configuration

| Parameter | Executed value |
|---|---:|
| Transmit antennas | 64 |
| Receive antennas | 8 |
| Spatial layers | 4 |
| FFT size | 256 |
| Active subcarriers | 144 |
| OFDM symbols | 14 |
| Subcarrier spacing | 30 kHz |
| Modulation | Normalized Gray-coded 16-QAM |
| DM-RS-like OFDM symbols | 4 and 11 in MATLAB indexing |
| Layer-separated pilot comb spacing | 4 |
| Channel model | Five-tap frequency-selective TDL |
| Tap delays | 0, 2, 5, 9, 14 samples |
| Relative tap powers | 0, -2.2, -4, -6, -8.2 dB |
| Equalizer | Gain-corrected MMSE; ZF available |
| SNR values | -5, 0, 5, 10, 15, 20 dB |
| PCC-SVD bundle candidates | 144, 48, 1 |
| Fixed-frame testbench campaign | 6 frames per SNR point |
| Adaptive-limit parameters | epsilon_max = 0.008, epsilon_inf = 0.002, beta_PR = 0.03 |
| Reconstruction budget at 10 dB | 0.005 |
| Reconstruction budget at 20 dB | 0.0023 |

## Mathematical Signal Model

For active subcarrier \(k\), the received signal is

$$
\mathbf{y}[k]
=
\mathbf{H}[k]\mathbf{W}[k]\mathbf{s}[k]
+
\mathbf{n}[k].
$$

Here:

- \(\mathbf{H}[k]\) is the \(8\times64\) physical MIMO channel;
- \(\mathbf{W}[k]\) is the \(64\times4\) precoder;
- \(\mathbf{s}[k]\) contains the four layer symbols;
- \(\mathbf{n}[k]\) is circular complex AWGN.

The receiver observes the effective channel

$$
\mathbf{G}[k]
=
\mathbf{H}[k]\mathbf{W}[k],
$$

with dimension \(8\times4\) per active subcarrier.

All precoders are semi-unitary:

$$
\mathbf{W}^{H}[k]\mathbf{W}[k]
=
\mathbf{I}_{4}.
$$

This keeps the total transmit-energy convention consistent across the compared precoding strategies.

## PCC-SVD Compatibility Metrics

### Pilot-Reconstruction NMSE

For candidate bundle size \(b\), the noise-free effective channel is sampled at the implemented pilot locations and reconstructed using the same interpolation procedure as the practical receiver.

The pilot-reconstruction NMSE is

$$
\eta_{\mathrm{PR}}(b)
=
\frac{
\sum_k
\|\mathbf{G}[k]-\mathbf{G}_{\mathrm{rec}}[k]\|_F^2
}{
\sum_k
\|\mathbf{G}[k]\|_F^2
}.
$$

This metric isolates structural pilot-sampling and interpolation error without AWGN.

### SNR-Adaptive Reconstruction Limit

The allowed reconstruction error is

$$
\varepsilon_{\mathrm{PR}}(\rho)
=
\min
\left(
\varepsilon_{\max},
\varepsilon_{\infty}
+
\frac{\beta_{\mathrm{PR}}}{\rho}
\right).
$$

The limit is looser at low SNR, where AWGN dominates, and tighter at high SNR, where deterministic interpolation error becomes visible.

### Selection Rule

Let the candidate set be

$$
\mathcal{C}
=
\{144,48,1\}.
$$

The feasible set contains candidates satisfying the active reconstruction limit.

If at least one candidate is feasible, PCC-SVD selects the feasible candidate with the largest subspace-capture ratio.

If no candidate is feasible, it falls back to the candidate with the minimum pilot-reconstruction NMSE.

In plain language:

```text
Feasible candidate exists:
    select the feasible candidate with maximum subspace capture

No feasible candidate:
    select the candidate with minimum pilot-reconstruction NMSE
```

## Simulink Testbench Architecture

The generated model is:

```text
Wideband_SVD_DMRS_Executable_Testbench.slx
```

The active execution path contains:

1. **Run Trigger**  
   Activates the numerical engine once at simulation time zero.

2. **Level-2 MATLAB S-Function**  
   The block `simulink_sfunction` executes the testbench interface.

3. **Status Register**  
   A Unit Delay block stores the six-element result vector.

4. **Demultiplexer**  
   Separates the six status-word values.

5. **Display Blocks**  
   Show:
   - status;
   - passed checks;
   - total checks;
   - BER;
   - effective-channel NMSE;
   - selected bundle size.

6. **To Workspace Block**  
   Logs the status word as:

```text
simulink_status_log
```

7. **Scope**  
   Provides graphical observation of the output status vector.

8. **Implemented Algorithm Workflow Subsystem**  
   Documents the complete processing order:
   - configuration;
   - frame and DM-RS generation;
   - TDL channel generation;
   - SVD candidate construction;
   - PCC-SVD selection;
   - digital precoding;
   - MIMO propagation and AWGN;
   - DM-RS LS estimation;
   - CSI selection;
   - ZF or MMSE equalization;
   - demapping and metric calculation.

The workflow subsystem is documentation by construction. It is not a second independent numerical implementation.

## Six-Element Status Interface

The S-function returns a six-element vector:

```text
[status, checks_passed, total_checks, BER, NMSE, selected_bundle]
```

The corresponding values are:

| Position | Meaning |
|---:|---|
| 1 | Overall execution status |
| 2 | Number of passed verification checks |
| 3 | Total number of verification checks |
| 4 | Final BER |
| 5 | Effective-channel NMSE |
| 6 | Selected or mean bundle size |

The runner also returns a structured summary containing fields such as:

```text
mode
status
passCount
totalCount
finalBer
finalNmse
finalBundle
message
```

## Execution Modes

The Simulink layer supports two execution modes.

### Testbench Mode

Testbench mode executes:

- all nine deterministic verification gates;
- a deterministic 10 dB PCC-SVD smoke test;
- status-word generation;
- display and workspace logging.

Run it with:

```matlab
summary = run_simulink_only('testbench');
```

### Analysis Mode

Analysis mode executes:

- the complete fixed-frame six-scenario MATLAB campaign;
- all configured SNR points;
- result-table generation;
- figure regeneration;
- final PCC-SVD output reporting.

Run it with:

```matlab
summary = run_simulink_only('analysis');
```

## Main MATLAB and Simulink Files

### Simulink-Specific Files

```text
build_simulink_testbench.m
simulink_sfunction.m
simulink_runner.m
run_simulink_only.m
prepare_simulink_model.m
reset_and_build_simulink.m
force_rebuild_simulink.m
stop_simulation_if_needed.m
validate_local_path.m
Wideband_SVD_DMRS_Executable_Testbench.slx
```

### Main Numerical-Engine Dependencies

```text
massive_mimo_ofdm_simulation.m
matlab_testbench.m
run_main.m
run_complete_simulation.m
run_high_statistics_analysis.m
run_all.m
```

### Shared-Vector Verification Files

```text
shared_vectors.mat
shared_vectors_manifest.json
verify_shared_vectors.m
```

Use the exact original filenames. Model callbacks, function resolution, and execution scripts may depend on them.

## Block-to-Function Traceability

| Technical operation | MATLAB stage |
|---|---|
| Layer mapping and digital precoding | `buildFrame`, `applyPrecoder` |
| TDL channel generation | `generateTdlChannel` |
| Covariance-SVD candidate bank | `covarianceSvdCandidate` |
| Unitary Procrustes alignment | `alignSvdBases` |
| PCC-SVD selection | `precoderChannelEstimatorCompatibilitySvd` |
| Effective-channel formation | `trueEffectiveChannel` |
| DM-RS LS estimation | `estimateEffectiveChannelLs` |
| ZF and gain-corrected MMSE detection | `equalizeLayers` |
| Metric and CSV generation | Metric functions and CSV writers |

This mapping ensures that the report, algorithm description, MATLAB implementation, and Simulink execution interface refer to the same numerical stages.

## Verification Methodology

The testbench runs nine deterministic verification gates.

| Verification gate | Technical purpose | Result |
|---|---|---|
| Wideband semi-unitarity | Verify equal-power precoder normalization | PASS |
| Aligned per-subcarrier semi-unitarity | Verify that alignment preserves basis norm | PASS |
| PCC-SVD semi-unitarity | Verify the selected precoder remains semi-unitary | PASS |
| Effective-channel dimensions | Verify \(144\times8\times4\) dimensions | PASS |
| 16-QAM round trip | Verify mapper and demapper ordering | PASS |
| Noiseless flat-channel LS | Verify exact pilot division and interpolation | PASS |
| Noiseless end-to-end recovery | Verify mapping, equalization, ordering, and demapping | PASS |
| Alignment reduces pilot error | Verify Procrustes alignment reduces reconstruction error | PASS |
| PCC-SVD feasibility or fallback | Verify feasible selection and fallback logic | PASS |

A failed gate invalidates the corresponding simulation campaign. The model does not report a passing status when any required gate fails.

## Executed Smoke-Test Results

The deterministic 10 dB testbench-mode smoke test produced:

| Metric | Executed value |
|---|---:|
| Verification gates passed | 9 of 9 |
| BER | \(3.18287\times10^{-3}\) |
| Effective-channel NMSE | \(2.42407\times10^{-2}\) |
| RMS EVM | 0.151627 |
| Exact capacity | 28.0944 bit/s/Hz |
| Selected-candidate pilot-reconstruction NMSE | \(3.809\times10^{-3}\) |
| Active reconstruction budget | \(5.000\times10^{-3}\) |
| Selected bundle | 48 subcarriers |

The deterministic seeds make this operating point useful for regression testing. Numerical drift in the processing chain changes the smoke-test output.

## Executed Analysis-Mode Result

The complete in-model analysis campaign finished with all nine checks passing.

At the final 20 dB PCC-SVD operating point:

| Metric | Executed value |
|---|---:|
| BER | \(1.20563\times10^{-5}\) |
| Effective-channel NMSE | \(3.5484\times10^{-3}\) |
| Mean selected bundle | 144 |

These values match the corresponding scripted MATLAB campaign outputs at the numerical-engine interface.

## MATLAB–Python Shared-Vector Verification

The deterministic shared-vector workflow verifies numerical conventions across MATLAB and Python without relying on identical random-number generators.

The executed verification includes four precoder families and seven checks per family:

- dimensions;
- effective-channel agreement;
- semi-unitarity;
- stored orthogonality;
- pilot-reconstruction NMSE;
- subspace capture;
- selected bundle.

Total executed result:

```text
28 of 28 shared-vector checks passed
```

Run the verification with:

```matlab
checks = verify_shared_vectors;
```

## Main Fixed-Frame Results

The six-frame-per-SNR testbench campaign is intended for model execution and regression checking rather than final high-statistics publication claims.

Selected BER values are:

| Scenario | -5 dB | 0 dB | 10 dB | 20 dB |
|---|---:|---:|---:|---:|
| Wideband SVD, LS, MMSE | 2.936e-1 | 1.627e-1 | 6.119e-3 | 1.206e-5 |
| Aligned per-subcarrier SVD, LS, MMSE | 1.943e-1 | 7.574e-2 | 2.532e-4 | 1.085e-4 |
| Unaligned per-subcarrier SVD, LS, MMSE | 3.288e-1 | 2.602e-1 | 2.316e-1 | 2.299e-1 |
| PCC-SVD, LS, MMSE | 2.036e-1 | 8.362e-2 | 3.370e-3 | 1.206e-5 |
| Wideband SVD, ideal CSI, MMSE | 1.775e-1 | 7.510e-2 | 2.411e-4 | 0 observed |
| Per-subcarrier SVD, ideal CSI, MMSE | 9.624e-2 | 1.773e-2 | 0 observed | 0 observed |

The unaligned per-subcarrier method exhibits a structural BER floor because the effective channel decorrelates between pilot positions. Alignment removes most of this artificial discontinuity. PCC-SVD transitions from finer bundles at low SNR toward the wideband candidate at high SNR as the reconstruction budget becomes stricter.

## Reproduction Procedure

### Complete Workflow

Run the complete package workflow with:

```matlab
run_all
```

This command is expected to:

- resolve local dependencies;
- execute the MATLAB verification testbench;
- run the scripted simulation campaign;
- regenerate result tables and figures;
- build or refresh the Simulink model;
- execute the Simulink testbench;
- print the completion summary.

### Simulink Testbench Only

```matlab
summary = run_simulink_only('testbench');
```

### Simulink Analysis Campaign

```matlab
summary = run_simulink_only('analysis');
```

### Clean Model Rebuild

```matlab
reset_and_build_simulink
```

or:

```matlab
force_rebuild_simulink
```

### Shared-Vector Verification

```matlab
checks = verify_shared_vectors;
```

## Recommended Repository Structure

```text
Simulink-Testbench-for-64x8-PCC-SVD-Massive-MIMO-OFDM/
├── README.md
├── report/
│   └── Simulink Testbench for the 64 x 8 PCC-SVD Massive MIMO-OFDM Link-Level Simulator.pdf
├── simulink/
│   ├── Wideband_SVD_DMRS_Executable_Testbench.slx
│   ├── build_simulink_testbench.m
│   ├── simulink_sfunction.m
│   ├── simulink_runner.m
│   ├── run_simulink_only.m
│   ├── prepare_simulink_model.m
│   ├── reset_and_build_simulink.m
│   ├── force_rebuild_simulink.m
│   ├── stop_simulation_if_needed.m
│   └── validate_local_path.m
├── verification/
│   ├── shared_vectors.mat
│   ├── shared_vectors_manifest.json
│   ├── verify_shared_vectors.m
│   └── logs/
├── results/
│   ├── csv/
│   ├── figures/
│   └── configuration/
└── docs/
    └── dependency_information.md
```

## Dependency on the Main MATLAB Numerical Engine

This Simulink repository should not contain a second independent copy of the entire physical-layer implementation unless a self-contained archive is specifically required.

The recommended arrangement is:

```text
Main PCC-SVD MATLAB project:
    complete numerical engine

Simulink testbench project:
    model, builder, S-function, runner, lifecycle scripts, logs, report
```

Before running the model, add both directories to the MATLAB path.

Example:

```matlab
addpath(genpath('../PCC-SVD-MATLAB-Engine'));
addpath(genpath('./simulink'));
```

Update the path to match the actual local folder names.

## Result and Log Locations

The executed package may use:

```text
results_matlab/
figures_matlab/
high_statistics_results_matlab/
high_statistics_figures_matlab/
```

Typical records include:

- verification-gate results;
- simulation configuration;
- per-scenario CSV tables;
- bundle-selection histogram;
- generated figures;
- plot index;
- smoke-test summary;
- Simulink workspace log;
- MATLAB–Python shared-vector results.

## Software Requirements

The testbench requires:

- MATLAB;
- Simulink;
- support for Level-2 MATLAB S-functions;
- the companion MATLAB numerical engine;
- the supplied deterministic verification files.

The exact compatible MATLAB release should be recorded with the released package.

## Model Limitations

The current testbench models:

- a single-cell, single-user digital-baseband downlink;
- four uncoded spatial layers;
- hard-decision 16-QAM;
- ideal time and frequency synchronization;
- a controlled five-tap TDL channel;
- a DM-RS-like reference grid;
- ideal transmitter-side channel knowledge for precoder construction.

The current model does not include:

- channel coding;
- HARQ;
- scheduler behavior;
- practical CSI feedback;
- feedback delay;
- reciprocity mismatch;
- oscillator phase noise;
- carrier-frequency offset recovery;
- timing synchronization recovery;
- nonlinear PA behavior;
- IQ imbalance;
- complete 3GPP conformance;
- hardware-in-the-loop validation;
- over-the-air validation.

The Simulink model is an orchestration testbench, not a native block-level implementation of every matrix operation.

## Future Work

Possible extensions include:

- hierarchical native Simulink stage wrappers;
- complete NR PDSCH and DM-RS mapping;
- channel coding and BLER;
- practical CSI feedback;
- mobility and Doppler;
- spatial correlation;
- RF impairments;
- synchronization recovery;
- hardware-in-the-loop execution;
- real-time testbench operation;
- over-the-air validation;
- additional antenna and layer profiles;
- automated continuous-integration execution.

## Report

The detailed report is:

```text
Modeling, Testbench Design, and Verification of the Executable Simulink Testbench for the 64 × 8 PCC-SVD Massive MIMO-OFDM Link-Level Simulator
```

Recommended repository location:

```text
report/Simulink Testbench for the 64 x 8 PCC-SVD Massive MIMO-OFDM Link-Level Simulator.pdf
```

## Citation

Until a formal publication record is available, the project may be cited as:

```bibtex
@techreport{rahman2026simulinkpccsvd,
  author      = {Md Moklesur Rahman},
  title       = {Modeling, Testbench Design, and Verification of the Executable Simulink Testbench for the 64 x 8 PCC-SVD Massive MIMO-OFDM Link-Level Simulator},
  institution = {Independent Research},
  address     = {Oulu, Finland},
  year        = {2026}
}
```

Replace this entry with the formal publisher citation and DOI after publication.

## License

No open-source license is granted unless a separate `LICENSE` file is included.

Until then, the project should be treated as:

```text
All rights reserved
```

## Author

**Md Moklesur Rahman**  
Independent Researcher  
Oulu, Finland

## Notice

The Simulink model executes the verified MATLAB numerical engine through a Level-2 MATLAB S-function.
The testbench validates execution, exposes status and metrics, and regenerates campaign results inside the model environment. 
