# cyanospectra: Implementation Report

*Cyanobacteria Absorption Spectra: Analysis & Deconvolution. What was built, how it was built, and how it relates to the project.*

## 1. Purpose

The project overview describes an open-source Python library that recovers pigment contributions from in vivo whole-cell absorption spectra of cyanobacteria. It then derives physiological parameters with quantified uncertainty, validated against synthetic ground truth. This report summarises what was implemented from that specification, what the delivered package contains, and how each part maps back to the project text.

## 2. What was done, step by step

1.  **Reading the specification.** Three things were taken from the overview: the processing chain (parsing, quality control, preprocessing, deconvolution, derived parameters, visualisation and classification), the two required deconvolution methods (Gaussian decomposition and NNLS unmixing) and the rule that every method is verified against synthetic spectra with known composition.

2.  **Project skeleton.** A Python package with tests and packaging metadata was created. A first attempt created the directories incorrectly; this was noticed when file writes failed, and the skeleton was rebuilt.

3.  **Core modules.** The pigment band table of the overview was converted into code, together with a seeded synthetic spectra generator (known band areas, centres and pigment ratios; noise; power-law scattering; optional band jitter and widening) and the pigment-ratio calculation.

4.  **Data input and cleaning.** Parsing of Excel, delimited text and text pasted from a spreadsheet (delimiter and decimal separator auto-detected, resampling to a common 1 nm grid); quality control returning structured findings; logged preprocessing (scattering correction, Savitzky-Golay smoothing, normalisation, replicate averaging).

5.  **Deconvolution.** A versioned reference library, Gaussian decomposition (bounded least squares, band centres limited to +/-5 nm, AIC/BIC band selection), NNLS unmixing on the reference library, and bootstrap / Monte Carlo uncertainty, all behind one common interface.

6.  **First end-to-end test and a key finding.** Both methods worked on synthetic data. Estimating scattering only from a short near-infrared window (710-750 nm) and extrapolating it badly distorted the Gaussian fit (it even selected a spurious phycoerythrin band), whereas fitting the scattering jointly with the bands recovered the ratios accurately.

7.  **Benchmark.** The estimation error was measured against ground truth as a function of noise, band overlap and scattering-correction error. The results determined the default settings (Section 5).

8.  **Remaining modules.** Figures (spectra, overlays, differences, 2nd/4th derivatives, deconvolution with residuals, benchmark); classification (rule-based interpreter, PLS-DA, logistic regression, grouped cross-validation, NumPy CNN inference); the single entry point analyze() with JSON and Excel export; and an optional AI-agent layer.

9.  **Tests and bug fixes.** A suite of 31 automated tests was written. Running it exposed a real bug in quality control (with three replicates an outlier was not flagged, because it was compared with the median of the others and pulled the reference towards itself; it is now compared with the group median). It also showed that a 15 % per-spectrum tolerance was unrealistic: single ratios such as APC/PC can be off by about 20 % in individual spectra. The correctness test was therefore replaced by a strict test on noise-free data plus a statistical test (median error over 8 realistic spectra).

10. **Verification and documentation.** A demo was run, the deconvolution figure was inspected visually, and the packaging (including the reference-library data file) was checked by building a wheel. English and Slovak READMEs were written, and the raw-spectrum vs. mechanistic-feature comparison was run and documented with an explicit caveat (Section 6).

## 3. Contents of the delivered package

The archive cyanospectra.zip contains the following.

| **Path**                             | **Content**                                                                                  |
|--------------------------------------|----------------------------------------------------------------------------------------------|
| pyproject.toml                       | Packaging and dependencies (optional extras for the agent layer and CNN training).           |
| README.md, README_SK.md              | Documentation, roadmap status and known limitations (English / Slovak).                      |
| examples/demo.py                     | Complete example on synthetic data: analysis, text summary, figure, Excel export.            |
| tests/test_all.py                    | 31 automated tests, including correctness against synthetic ground truth.                    |
| cyanospectra/synthetic.py            | Seeded synthetic spectra generator with known truth; labelled synthetic datasets.            |
| cyanospectra/parsing.py              | Excel, text and pasted-text parsing; metadata; common 1 nm grid.                             |
| cyanospectra/qc.py                   | Quality control: saturation, negatives, narrow range, noise, low signal, outlier replicates. |
| cyanospectra/preprocessing.py        | Scattering correction, smoothing, normalisation, replicate averaging, with a log.            |
| cyanospectra/reference.py, data/     | Versioned reference library with documented origin; user-extensible (JSON/CSV).              |
| cyanospectra/deconvolution.py        | Gaussian decomposition and NNLS unmixing behind one interface.                               |
| cyanospectra/uncertainty.py          | Residual bootstrap and Monte Carlo uncertainty (seeded).                                     |
| cyanospectra/derived.py              | Band parameters and physiological ratios.                                                    |
| cyanospectra/benchmark.py            | Systematic comparison against ground truth.                                                  |
| cyanospectra/viz.py                  | Figures; each function returns a Matplotlib Figure.                                          |
| cyanospectra/classification.py       | Rule-based interpreter, PLS-DA, logistic regression, grouped CV, CNN (NumPy inference).      |
| cyanospectra/pipeline.py             | analyze() entry point, JSON session, Excel export as bytes.                                  |
| cyanospectra/agent.py                | Optional AI-agent layer: tool definitions, tool-use loop, offline text summary.              |
| cyanospectra/pigments.py, \_utils.py | Pigment band table; shared helpers.                                                          |

## 4. How it relates to the project

| **Project overview**                                                                   | **Implementation**                                                                                                                                                                                                                 |
|----------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Figure 1: spectrum, deconvolution, parameters                                          | analyze() in pipeline.py performs the whole chain in one call.                                                                                                                                                                     |
| Pigment table (Chl a 435/678 nm, Car 450/478/500 nm, PC 620 nm, APC 650 nm, PE 565 nm) | pigments.py; used as initial band positions and as reference spectra.                                                                                                                                                              |
| Light scattering, A = a \* lambda^-b                                                   | preprocessing.py (NIR fit or offset) and a joint baseline fit in deconvolution.py. The model is written as a500\*(lambda/500)^-b, which is equivalent.                                                                             |
| Gaussian decomposition, centre +/-5 nm, bounded widths, AIC/BIC                        | GaussianDeconvolver with bounded least squares and information-criterion band selection.                                                                                                                                           |
| NNLS unmixing on a documented reference library                                        | NNLSDeconvolver and reference.py; each record documents origin, solvent, method and source.                                                                                                                                        |
| R2, RMSE, residual spectrum, bootstrap / Monte Carlo                                   | fit_stats() in deconvolution.py; uncertainty.py.                                                                                                                                                                                   |
| PC/Chl, APC/PC, Car/Chl, PBS/Chl                                                       | derived.py (PE/PC added when PE is present).                                                                                                                                                                                       |
| Validation against synthetic ground truth, pytest assertion                            | synthetic.py, benchmark.py and tests/test_all.py (ground-truth tests run on every pytest run).                                                                                                                                     |
| Classification: raw spectrum vs. mechanistic features                                  | classification.py: PLS-DA, logistic regression and CNN on raw spectra, versus models on deconvolution features; cross-validation grouped by culture.                                                                               |
| Design principles                                                                      | Pure functions, JSON-serializable results, structured QC findings, logged preprocessing, seeded randomness, figures without show()/savefig(), NumPy CNN inference, single analyze() entry point. The tests check several of these. |
| Quality control list in the roadmap                                                    | qc.py implements saturation (A \> 1.5), negative values, narrow range, noise level and outlier replicates.                                                                                                                         |

Two items of the overview are intentionally not addressed: the web layer of the cyano.tools platform (stated as outside the scope of the repository) and the license (stated as to be determined).

## 5. Main results

### 5.1 Benchmark on synthetic data

Median relative error of the four ratios (6 spectra per condition, seed 0).

| **Condition**                                       | **Gaussian**               | **NNLS**                 |
|-----------------------------------------------------|----------------------------|--------------------------|
| Noise sd 0.001 / 0.003 / 0.01                       | 3.7 / 6.9 / 13.8 %         | 2.9 / 2.8 / 3.1 %        |
| Band widening x1.0 / 1.15 / 1.3                     | 6.8 / 13.3 / 21.1 %        | 2.8 / 7.8 / 14.3 %       |
| Scattering removed with error 0 / +10 / -10 / +30 % | 6.8 / 49.4 / 12.1 / 42.0 % | 2.8 / 4.6 / 4.7 / 11.2 % |
| Scattering fitted in NIR window only                | 46.7 %                     | 5.8 %                    |
| Scattering fitted jointly with the bands            | 3.7 %                      | 3.7 %                    |

Consequences for the defaults: NNLS is the default method because it is the more robust one overall; analyze() fits the scattering baseline jointly with the bands unless a separate scattering correction is requested; and both methods degrade when bands overlap more than assumed. These are the conditions under which each method fails, as the overview asks the benchmark to document.

### 5.2 Classification on synthetic data

Grouped cross-validation accuracy: raw spectrum with PLS-DA 0.81 (+/-0.17) and with logistic regression 0.96 (+/-0.08); deconvolution features with PLS-DA and with logistic regression 1.00.

**Caveat:** this is not an answer to the central scientific question of the project. The synthetic classes were defined directly through pigment ratios, so features from deconvolution are favoured by construction. What is delivered is the infrastructure for the comparison; the decisive test requires real data or synthetic classes that differ in other ways than by ratios.

## 6. Status against the roadmap

| **Roadmap item**                                | **Status** | **Note**                                                                           |
|-------------------------------------------------|------------|------------------------------------------------------------------------------------|
| Synthetic spectra generator                     | Done       | Seeded; round-trip fitting verified.                                               |
| Data parsing                                    | Done       | Files, pasted text, delimiter/decimal detection, metadata.                         |
| Quality control                                 | Done       | Structured findings.                                                               |
| Preprocessing                                   | Done       | Every step logged.                                                                 |
| Reference spectral library                      | Partial    | Mechanism complete; shipped spectra are parametric placeholders.                   |
| Deconvolution (both methods)                    | Done       | With AIC/BIC and uncertainty.                                                      |
| Benchmark                                       | Partial    | Default chosen; synthetic data only.                                               |
| Derived parameters                              | Done       |                                                                                    |
| Visualisation                                   | Done       |                                                                                    |
| Classification                                  | Partial    | CNN training untested; thresholds uncalibrated; comparison only on synthetic data. |
| Handover packaging                              | Done       | JSON, session file, Excel bytes, analyze().                                        |
| Installation docs, license, CI, example dataset | Not done   | Marked TBD in the overview.                                                        |

## 7. Known limitations

- The built-in reference library consists of parametric Gaussian placeholders at the in vivo band positions, not measured spectra, and it uses the same templates as the synthetic generator. Band jitter in the generator mitigates this circularity, but NNLS accuracy on real data depends on measured references.

- All benchmark numbers hold for synthetic data. Validation on real cultures, for example against extracted pigments, is still required.

- Bootstrap and Monte Carlo uncertainty capture noise only. Systematic error (reference mismatch, wrong scattering) is not included, so the error against truth is often several times the reported standard deviation.

- Rule-based thresholds and the synthetic class definitions (low light, high light, nitrogen limitation) are illustrative and not calibrated.

- The PyTorch training function for the 1D CNN was not run (PyTorch was not installed); only the NumPy inference path is tested.

- The AI-agent loop was tested against a fake client, not against the live API.

- A Gaussian fit without a baseline term cannot represent over-subtracted (negative) backgrounds.

## 8. Suggested next steps

- Add measured reference spectra of the pigments and re-run the benchmark with them.

- Validate on real spectra of cultures grown under known light and nutrient regimes.

- Evaluate raw-spectrum vs. mechanistic-feature classification on independent, real data; train and test the CNN.

- Calibrate the rule-based thresholds; add license, installation documentation, CI and an example dataset.
