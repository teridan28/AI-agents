# cyanospectra

Open-source Python library for analysing and deconvoluting *in vivo* absorption spectra of cyanobacteria
(parsing → QC → preprocessing → deconvolution → derived parameters → visualisation / classification).

## Install
```bash
pip install -e .            # core
pip install -e ".[dev]"     # + pytest
pip install -e ".[agent]"   # + anthropic SDK (optional AI agent)
pip install -e ".[cnn]"     # + torch (only to TRAIN the 1D CNN; inference is pure NumPy)
pytest                      # 31 tests
```

## Quick start
```python
import cyanospectra as cs

spectra = cs.parse_table("data/example_spectra.xlsx")      # or cs.parse_pasted(clipboard_text)
result = cs.analyze({
    "spectra": spectra,
    "preprocessing": {"smoothing": "savgol"},               # optional, logged
    "deconvolution": {"method": "both"},                    # 'nnls' | 'gaussian' | 'both'
    "uncertainty": {"method": "bootstrap", "n": 30, "seed": 0},
})
result["ratios"]    # {sample: {"PC/Chl": 0.52, "APC/PC": 0.26, "Car/Chl": 0.59, "PBS/Chl": 0.66}}
result["qc"]        # structured findings: code, severity, message
result["deconvolution"][sample]["gaussian"]["bands"]   # centre, FWHM, height, area, share
cs.viz.plot_deconvolution(result["deconvolution"][sample]["nnls"])   # returns a Matplotlib Figure
open("out.xlsx", "wb").write(cs.export_excel(result))
```
`analyze()` output is plain JSON-serializable (`cs.to_json(result)` = session file).
Note: `result["ratios"]` is keyed by sample, and comes from `result["default_method"]`
(per-method values: `result["ratios_by_method"]`).

Run `python examples/demo.py` for a complete synthetic example.

## Modules
| Step | Module | Notes |
|---|---|---|
| Synthetic generator | `synthetic` | seeded; known areas/centres/ratios; noise, power-law scattering, band jitter, overlap |
| Parsing | `parsing` | xlsx / delimited text / pasted text / bytes; delimiter + decimal comma auto-detected; 1 nm grid |
| QC | `qc` | saturation, negatives, NaN, narrow range, noise, low signal, outlier replicates |
| Preprocessing | `preprocessing` | scattering (power law or offset), Savitzky–Golay, normalisation, replicate averaging; logged |
| Reference library | `reference`, `data/reference_library.json` | versioned; every record documents origin/solvent/method/source; add own spectra (JSON/CSV) |
| Deconvolution | `deconvolution` | `GaussianDeconvolver` (bounded LSQ, centre ±5 nm, FWHM bounds, AIC/BIC band selection) and `NNLSDeconvolver`, one interface |
| Uncertainty | `uncertainty` | residual bootstrap or Monte Carlo, seeded |
| Derived parameters | `derived` | band area/centre/FWHM/share, PC/Chl, APC/PC, Car/Chl, PBS/Chl (PE/PC if PE present) |
| Visualisation | `viz` | spectra, overlay, difference, 2nd/4th derivative, deconvolution + residuals, benchmark |
| Classification | `classification` | rule-based interpreter; PLS-DA/logreg on raw spectrum vs. mechanistic features; grouped CV; NumPy CNN inference |
| Benchmark | `benchmark` | error vs noise, overlap, scattering error |
| Entry point | `pipeline.analyze` | pure function: dict in, dict out |
| AI agent (optional) | `agent` | Claude tool-use definitions + loop; offline summary without any model |

Ratios are ratios of **integrated absorption (band areas)** of pigment contributions, not concentrations.
Scattering model: `A = a500·(λ/500 nm)^-b` (`a500` = scattering at 500 nm; equivalent to `a·λ^-b`).

## Benchmark result (synthetic, 6 spectra per condition, seed 0, median relative error of the 4 ratios)
| condition | Gaussian | NNLS |
|---|---|---|
| noise sd 0.001 / 0.003 / 0.01 | 3.7 / 6.9 / 13.8 % | 2.9 / 2.8 / 3.1 % |
| band widening ×1.0 / 1.15 / 1.3 | 6.8 / 13.3 / 21.1 % | 2.8 / 7.8 / 14.3 % |
| scattering removed with error 0 / +10 % / −10 % / +30 % | 6.8 / 49.4 / 12.1 / 42.0 % | 2.8 / 4.6 / 4.7 / 11.2 % |
| scattering fitted in NIR (710–750 nm) only | 46.7 % | 5.8 % |
| scattering fitted **jointly** with the bands | **3.7 %** | 3.7 % |

Conclusions encoded in the defaults: (1) NNLS is the more robust method overall → `DEFAULT_METHOD = "nnls"`;
(2) extrapolating scattering from a short NIR window is the dominant failure mode, especially for the Gaussian
fit → `analyze()` fits the power-law baseline **jointly** with the bands unless you request a separate
`scattering` correction; (3) both methods degrade when bands overlap more than assumed.
Rerun with `cs.run_benchmark()`; correctness is asserted in `tests/` on every run.

## Known limitations (please read)
* **The built-in reference library is a parametric placeholder** (Gaussian bands at the in vivo positions from the
  project table), not measured spectra, and it uses the same band templates as the synthetic generator. Band
  jitter in the generator mitigates this "inverse crime", but NNLS accuracy on real data depends entirely on
  the quality of the reference spectra: add measured ones (`ReferenceLibrary.add` / `add_csv`).
* Benchmark numbers hold for synthetic data only. Validate on real cultures (e.g. against extracted pigments).
* Bootstrap/Monte-Carlo uncertainty captures **noise only**. Systematic error (reference mismatch, wrong
  scattering) is not included: on jittered synthetic data the error vs. truth is often several times the reported SD.
* Rule-based thresholds (`DEFAULT_THRESHOLDS`) and the synthetic class definitions (LL/HL/N_lim) are illustrative, not calibrated.
* `train_cnn1d` (PyTorch) was **not run** in development (torch not installed); the NumPy inference path is tested.
* The agent loop is tested against a fake client, **not against the live API**.
* Gaussian fit has no negative-baseline term unless `baseline="offset"/"power_law"`; over-subtracted spectra fit badly.

## AI agent (optional)
```python
from cyanospectra import agent
print(agent.offline_summary(result))                         # no model needed
print(agent.run_agent("Is this culture high-light acclimated?", table_text=pasted))  # needs ANTHROPIC_API_KEY
agent.TOOLS; agent.dispatch_tool(name, args)                 # reuse in any agent framework / MCP server
```
The model only plans calls and explains results; all numbers come from the library's tools.

## License
To be determined.

## Roadmap status
Legend: ✅ implemented and tested · 🟡 implemented, with a stated gap · ⬜ not done

- ✅ Synthetic spectra generator (seeded, known composition, noise, scattering; round-trip verified)
- ✅ Data parsing (files, pasted text, delimiter/decimal detection, 1 nm resampling, metadata)
- ✅ Quality control (saturation, negatives, narrow range, noise, outlier replicates)
- ✅ Preprocessing (scattering, Savitzky–Golay, normalisation, replicate averaging; fully logged)
- 🟡 Reference spectral library (versioned, documented, extensible; **shipped spectra are parametric placeholders**, measured spectra still to be added)
- ✅ Deconvolution (Gaussian decomposition and NNLS behind one interface, AIC/BIC, bootstrap/MC uncertainty)
- 🟡 Benchmark (systematic comparison and default-method choice done; **synthetic data only**, real-data validation pending)
- ✅ Derived parameters (band parameters and physiological ratios)
- ✅ Visualisation (overlays, differences, 2nd/4th derivatives, deconvolution with residuals)
- 🟡 Classification (rule-based interpreter, PLS-DA, logistic regression, grouped CV, NumPy CNN inference; **CNN training untested, thresholds uncalibrated, raw-vs-feature comparison only run on synthetic data**)
- ✅ Handover packaging (JSON-serializable outputs, session JSON, Excel bytes, single `analyze()` entry point)
- ⬜ Installation docs beyond `pip install -e .`, license, CI, real example dataset (TBD)
