# Cyanobacteria Absorption Spectra: Analysis & Deconvolution

An open-source Python library for analyzing and deconvoluting *in vivo* absorption spectra of cyanobacteria. 

## Problem: What do you want to exist that does not exist yet?

We want an open-source Python library that takes *in vivo* absorption spectra of whole cyanobacteria cells and derives the contribution of individual pigments — chlorophyll *a*, carotenoids, and the phycobiliproteins (PC, APC, PE) — together with physiologically interpretable parameters.

A whole-cell absorption spectrum is one of the fastest and cheapest measurements you can make on a culture, and also one of the least exploited. It carries information about the state of the light-harvesting apparatus and about acclimation to light and nutrient conditions. The difficulty is that individual pigment bands overlap heavily, and the spectrum is additionally distorted by light scattering on the cells. As a result, routine practice evaluates only a few point ratios (e.g. A680/A620), which extracts a fraction of what is actually in the data.

The intended end state is a standalone, pip-installable package (`pip install -e .`, MIT licensed) that a third party can call without knowing anything about the project — pure functions in, serializable dicts out, no I/O or global state inside the analysis layer. The result is designed to become a future module of the [cyano.tools](https://github.com/Zastupic/cyano-tools) platform (Department of Adaptive Biotechnologies, CzechGlobe); the web layer is out of scope for us, so everything that would make that handover a packaging job rather than a rewrite belongs in the library from day one.

## Data: What goes in?

**Input is tabular absorbance data**: the first column is wavelength, the remaining columns are samples. It arrives in two shapes, both handled by a parsing layer that is strictly separate from the analysis layer:

- `parse_table(...)` — data from a file (Excel / delimited text), with autodetection of the delimiter and of decimal comma versus decimal point.
- `parse_pasted(text: str)` — a string pasted straight out of Excel (tab-separated). This exists as its own function because on the web the clipboard arrives as a string, not as a path.

Parsed spectra are interpolated onto a common 1 nm grid and the wavelength range is unified across samples. An optional sample metadata table (strain, light regime, time point, replicate) can accompany the data; without it, group comparisons are not possible.

**This data is not sensitive.** It is optical density readings of cyanobacterial cultures, plus metadata such as strain name, light conditions, and timestamps. There is no personal data, no patient or human-subject information, and no proprietary or commercially confidential content. **It is safe to send to a model**, and it is safe to commit sample files to a public repository — which matters, because the reference library of pigment spectra and an example dataset are themselves deliverables of the project.

## Correction:

By benchmarking against synthetic ground truth that we generate ourselves.

The core deliverable of the ML/AI part of the project is a **synthetic spectra generator**: given a specified mixture of pigments, it produces a spectrum with **realistic noise** and a **scattering** contribution (a power-law term `a·λ⁻ᵇ` in the NIR, of the kind the correction step is meant to remove). Because we chose the mixture, we know the true answer — the true band areas, centres, and relative pigment ratios — by construction.

That gives us an objective correctness criterion instead of a plausible-looking plot. We run both deconvolution methods through the same interface on the same synthetic data:

- **Gaussian decomposition** — sum of bands initialized at known *in vivo* positions (Chl *a* Soret ~435 nm and Qy ~678 nm; carotenoids ~450/478/500 nm; PC ~620 nm; APC ~650 nm; PE ~565 nm), fit by bounded least squares with constraints on centre (±5 nm) and FWHM.
- **Unmixing onto a reference library** — NNLS or constrained linear regression, yielding relative pigment contents directly.

We then measure how estimation error behaves as a function of **noise level**, **degree of band overlap**, and **error in the scattering correction**, using fit-quality metrics (R², RMSE, residuals, AIC/BIC for choosing the number of bands) and parameter uncertainty from bootstrap resampling of residuals or Monte Carlo perturbation of the noise. Correct means the recovered composition matches the known composition within a stated tolerance — and that tolerance is encoded as a `pytest` assertion, so correctness is checked on every run rather than eyeballed once.

Ground truth also settles the project's main scientific question, which compares a model on the raw spectrum (PLS-DA, 1D-CNN) against a model on mechanistic features extracted by deconvolution: *does mechanistic feature extraction pay off, or can a network learn it directly?*

## Plan:

### Project Milestones

Strictly this order — nothing after the generator can be evaluated without it. Each step ships with its own tests and commits.

1. **Synthetic spectra generator.** A seeded pure function: pigment configuration from a data file in, spectrum on a 1 nm grid out, plus controllable noise and a power-law scattering term. Verified by round-tripping — fit it back and assert the recovered parameters match those that went in.

2. **Data parsing.** `parse_table(...)` for Excel and delimited files, `parse_pasted(text)` for clipboard text, both autodetecting delimiter and decimal comma. Spectra are resampled to a common 1 nm grid, with optional metadata (strain, light, time, replicate).

3. **Quality control.** Returns structured findings — code, severity, message — never printed warnings. Covers A > 1.5, negative values, narrow wavelength range, noise level, saturation and outlier replicates.

4. **Preprocessing.** Optional parameterized steps: scattering correction (A750/A800 or an `a·λ⁻ᵇ` NIR fit), Savitzky–Golay smoothing, normalization, replicate averaging. Every step is logged, so an analysis can be replayed exactly.

5. **Reference spectral library.** Versioned CSV/JSON of *in vivo* and *in vitro* pigment spectra, each documenting solvent, method and citation. Users can add their own without touching the code.

6. **Deconvolution — the core.** Two independent methods behind one interface: bounded least-squares Gaussian decomposition, and NNLS unmixing onto the reference library. Bands, fixed parameters and starting estimates are set through function arguments, not a UI.

7. **Benchmark against ground truth.** Sweep both methods over noise, band overlap and scattering-correction error, scored against known compositions. The result picks the default method and documents where each one fails.

8. **Derived parameters.** Per-band area, amplitude, centre, FWHM and relative share. Converted into physiological ratios: PC/Chl, APC/PC, Car/Chl, PBS/Chl.

9. **Visualization.** Functions return a `Figure` and never call `show()` or `savefig()`. Overlaid and difference spectra, 2nd/4th derivatives, and a deconvolution panel with residuals underneath.

10. **AI classification layer.** Classifies culture state (light regime, N-limitation, strain) with honest cross-validation and an explicit discussion of overfitting. Compares models on the raw spectrum (PLS-DA, 1D-CNN) against models on mechanistic features, with a key-free rule-based interpreter underneath.

11. **Handover packaging.** JSON-serializable outputs, `.xlsx` returned as bytes, versioned session JSON, and one `analyze(payload: dict) -> dict` wrapping the whole pipeline. Any 1D-CNN ships as offline-trained weights with NumPy or ONNX inference — no PyTorch or TensorFlow on the host.



