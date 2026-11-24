# Repository Context: cheron2000/PS2

> Reviewed on 2026-11-24 from the public GitHub repository. The recursive tree contains 140 entries and is not truncated. Static inspection covered the project specification, research/build protocol, model, losses, preprocessing, dataset adapters, geospatial pairing, inference, training, metrics, benchmark/evaluation utilities, Streamlit UI, setup scripts, and test inventory. No test suite, model training, real-data acquisition, GPU run, or benchmark was executed during this review.

## 1. Executive summary

**PS2 is an uncertainty-aware Sentinel-2 super-resolution research prototype for SIH26142**, the NTRO problem statement on deep-learning super-resolution mapping from medium-resolution satellite imagery.

The intended task is 10 m Sentinel-2 L2A imagery to a 2.5 m output (4×) with four bands (blue, green, red, NIR), a per-pixel predictive uncertainty product, geospatially correct outputs, and validation against real high-resolution reference imagery. Its intended differentiator is India-specific validation using ISRO Cartosat-2S/3, plus downstream land-cover utility—not merely visually sharper images.

This repository is unusually candid about the difference between code plumbing and scientific validation. It contains a meaningful implementation base:
- CNN feature extraction, simplified second-order channel attention, fixed-window transformer blocks, sub-pixel upsampling, and mean/log-variance heads.
- L1/L2 reconstruction and heteroscedastic Gaussian NLL losses with validity masks.
- Sentinel-2 preprocessing, normalization, patch tiling, and cloud/invalid-pixel masking.
- Array-based SEN2NAIP/SEN2Vénus/Cartosat pairing with NCC registration/QC and geospatial-grid contracts.
- Checkpointed training, safe CLI checkpoint loading, GeoTIFF inference, tiled inference, uncertainty calibration helpers, scene-disjoint split utilities, provenance manifests, and benchmark gating.
- Streamlit visualization UI and a script-based test suite.

**Overall verdict:** credible research and evaluation-infrastructure prototype, but not yet a real-data-validated super-resolution system. The key blockers remain actual product ingestion, sensor harmonization, scene-level split integration, trained checkpoint evidence, Cartosat acquisition/validation, and calibrated uncertainty/downstream results.

### Highest-priority findings
1. The main CLI inference loader now uses PyTorch's safer `weights_only=True`, but the Streamlit app and a legacy evaluation script still load checkpoints with `weights_only=False`. Those paths retain a pickle-based arbitrary-code-execution risk for untrusted checkpoints.
2. Training currently performs a seeded random split by sample index, while a scene-disjoint split protocol exists separately. The CLI does not consume that scene protocol; geographic leakage remains possible if patches from the same scene/AOI are split across train and validation.
3. The official SEN2NAIP release is TACO-format, while the active loader expects paired local `.npy` arrays. A real-format converter is not present.
4. Cartosat pairing is array-first. Portal download, orthorectification, CRS reprojection, and actual product schema are external prerequisites, not implemented ingestion.
5. The tracked data directory contains a WorldCover GeoTIFF, but no SEN2NAIP/SEN2Vénus training pairs or trained checkpoint are present in the repository tree.
6. The newer benchmark/evaluation contract is a positive design improvement, but the older `scripts/run_evaluation.py` path remains a separate, less rigorous evaluator and should not be used to make current scientific claims.

## 2. Repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/PS2` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 8,206 KB |
| Recursive tree | 140 entries; complete |
| Problem statement | SIH26142 — NTRO |
| Domain | Remote sensing / geospatial deep learning |
| Main language | Python |
| Model framework | PyTorch |
| Array/geospatial stack | NumPy, SciPy, rasterio |
| UI | Streamlit |
| Evaluation | NumPy metrics, scene protocol, benchmark contract |
| Test inventory | 21 `test_*.py` scripts across root and `tests/` |
| CI workflows | No `.github/workflows/` entries found |

The tree includes a single ESA WorldCover tile under `data/worldcover/`. No paired SEN2NAIP/SEN2Vénus LR/HR array directories, Cartosat sample set, or trained model checkpoint are tracked.

## 3. Problem and intended scientific contribution

The official problem statement asks for a pipeline that enhances medium-resolution satellite imagery while retaining spectral/geospatial consistency, validates against high-resolution references, improves analytical utility, and accounts for uncertainty.

The repository's solution summary identifies the India-specific Cartosat route as the intended novelty:
- Primary 4× benchmark: Sentinel-2 10 m to NAIP 2.5 m using SEN2NAIP.
- Secondary 2× benchmark: SEN2Vénus 10 m to 5 m.
- India holdout: Sentinel-2 paired with ISRO Cartosat-2S/3.
- Downstream utility: land-cover classification comparison using ESA WorldCover labels.
- Uncertainty: predictive variance, Gaussian NLL, calibration/coverage evaluation.

The research notes explicitly acknowledge existing Sentinel-2 super-resolution work such as ESA OpenSR and SEN2SR/SEN2SRLite. The proposed differentiator is not a novel claim for the architecture itself; it is a rigorously acquired, registered, provenance-sealed Indian high-resolution validation route and evidence of downstream usefulness.

The Cartosat route remains a target, not a completed result.

## 4. Architecture and module map

### Model — `src/model.py`
- NCHW tensor convention.
- Four-band input and output by default.
- Shallow convolutional feature stem.
- High-order attention blocks using channel-wise spatial mean and variance as a low-cost second-order proxy; this is not a full covariance implementation.
- Fixed, non-shifted local-window multi-head self-attention plus MLP.
- Cross-stage feature fusion.
- PixelShuffle sub-pixel upsampling at 2× or 4×.
- Separate mean and log-variance output heads.

The build log records CPU PyTorch smoke-test verification for a 4× output and an odd-sized input requiring window padding/cropping. No CUDA performance benchmark is recorded. The fixed-window attention simplification may create boundary artifacts; it is explicitly documented as a future ablation/inspection item.

### Losses — `src/losses.py`
- L1 or L2 reconstruction loss.
- Heteroscedastic Gaussian NLL from predicted log variance.
- Optional broadcastable validity mask.
- Configurable reconstruction/NLL weighting and bounded log variance.
- Empty valid masks are handled without returning NaN.

The bug-fix log says the default NLL weight was reduced from 0.1 to 0.01 after a mean-collapse issue in which the model could reduce uncertainty loss while producing near-constant imagery. The reported 35% validation-error reduction is documented in the repository, but this review did not independently reproduce that experiment.

### Preprocessing — `src/preprocessing.py`
- Sentinel-2 Scene Classification Layer validity masking.
- Reflectance, percentile, and z-score normalization with reversible statistics.
- Patch tiling, overlap support, and minimum-valid-fraction filtering.
- Invalid pixels can be excluded from normalization/training/evaluation.

The SCL class policy follows documented Sentinel-2 classes but the code's own notes say it has not been checked against a real Sentinel-2 SAFE product in this environment.

### Dataset adapters
- `src/datasets/sen2naip.py`: local LR/HR NumPy pair loader, pair registration using normalized cross-correlation, overlap crop, and low-quality pair rejection.
- `src/datasets/sen2venus.py`: analogous 2× secondary benchmark loader.
- `src/datasets/cartosat_pairing.py`: array-first Cartosat pairing, integer scale checks, NCC alignment, optional grid metadata checks, mask handling, affine reflectance harmonization, and provenance metadata.
- `src/datasets/worldcover.py`: reads ESA WorldCover GeoTIFFs, remaps raw class codes to 11 consecutive labels, and reprojects categorical labels with nearest-neighbor resampling.
- `src/datasets/geospatial.py`: CRS/affine/pixel-size contracts, scale-aligned registration shifts, transform-preserving crops, and nodata/validity-mask propagation.
- `src/datasets/provenance.py`: SHA-256 file hashes and a sealed canonical JSON manifest.

### Training — `src/train.py`
- PyTorch Dataset/DataLoader training loop.
- AdamW optimizer, optional cosine scheduler, gradient clipping.
- Reproducible Python/NumPy/PyTorch seeding and RNG state checkpointing.
- Validation-driven best checkpoint selection when `--val-fraction` is nonzero.
- Atomic checkpoint writes, run manifest, dataset fingerprint, resume support, and JSONL telemetry.
- Safe checkpoint resume with `weights_only=True` plus narrowly registered NumPy types.

### Inference — `src/infer.py`
- NumPy CHW or rasterio-readable input.
- Optional reflectance normalization and validity mask.
- Model checkpoint/config reconstruction.
- Preflight bounds for input pixels and raw bytes.
- Single-pass or tiled inference.
- SR mean and variance output; variance is scaled by the square of the reflectance divisor when output units are converted.
- GeoTIFF CRS and affine transform propagation with scale-adjusted pixel size, or NumPy output.
- Atomic file writes and optional JSONL telemetry.

### Evaluation and reproducibility
- `src/metrics.py`: PSNR, SSIM, spectral-angle error, integer-shift NCC alignment diagnostics, masked metrics, and ERGAS/bandwise helpers.
- `src/eval_downstream.py`: confusion matrix, class IoU, overall accuracy, and NDVI-threshold classifier comparison across SR/bicubic/HR.
- `src/scene_protocol.py`: deterministic scene-disjoint splits, manifest validation, scene-level bootstrap confidence intervals, WorldCover schema validation, downstream metrics, and uncertainty calibration.
- `src/evaluation_contract.py` + `configs/evaluation_contract.json`: machine-checked frozen policy for dataset routes, band order, scale, masks, baselines, metrics, uncertainty coverage, improvement gates, and reproducibility.
- `src/benchmark.py`: controlled per-scene evaluation of nearest/bicubic/SR, and a data-status gate that refuses to label a benchmark verified-real unless the required metadata/manifests are supplied.

## 5. Main data flow

```text
Sentinel-2 L2A input
  -> SCL / nodata / geospatial validity mask
  -> band order + reflectance normalization
  -> paired LR/HR arrays
  -> registration and pair QC
  -> patch dataset
  -> SRModel
       -> reconstructed mean
       -> predicted log variance
  -> loss: reconstruction + heteroscedastic NLL
  -> checkpoint + run manifest + telemetry
  -> inference (single pass or tiled)
  -> SR GeoTIFF + uncertainty GeoTIFF
  -> scene-level metrics / downstream utility / uncertainty calibration
```

The implemented pipeline is currently connected around array-based datasets. Real product download and conversion are not automatic parts of the data flow.

## 6. Critical gaps and risks

### P1 — Unsafe checkpoint loading remains in UI and legacy evaluation paths
The main `src/infer.py:load_checkpoint()` uses `torch.load(..., weights_only=True)` and documents why it must not silently fall back to unrestricted pickle loading.

However:
- `app.py:get_model()` directly calls `torch.load(..., weights_only=False)`.
- `scripts/run_evaluation.py:load_model()` also calls `torch.load(..., weights_only=False)`.

This is an inconsistent security boundary. Loading a malicious or untrusted checkpoint through either path can execute pickle payload code. The comment in `app.py` that this is safe because the file was trained locally is not a reliable control if the checkpoint path is user-configurable or the artifact is replaced.

**Fix:** route every UI/evaluation load through the safe shared checkpoint loader; remove all unrestricted `torch.load` calls; test that malformed/untrusted checkpoint payloads fail closed.

### P1 — Training split is sample-index random, not scene-disjoint
`src/train.py:split_dataset()` uses PyTorch `random_split` over dataset indices. The code comment explicitly acknowledges that it does not persist a scene/AOI-level split manifest.

A scene-disjoint split implementation exists in `src/scene_protocol.py`, and the frozen evaluation contract declares scene-disjoint splits as policy, but the training CLI does not use it. If the dataset contains many crops from the same geographic scene, patches from that scene may land in both training and validation.

**Impact:** validation loss may overstate generalization because train/validation samples share spatial context, acquisition conditions, and geography. It also means the training split does not implement the frozen evaluation policy.

**Fix:** require scene_id/sample_id metadata, create and persist the scene split manifest before training, consume that manifest in the CLI, and verify no scene/sample overlap. Keep the final test scenes excluded from checkpoint selection.

### P1 — Official dataset formats are not integrated
The SEN2NAIP loader expects:
```text
data-root/
  lr/<id>.npy
  hr/<id>.npy
```
The repository explicitly documents that the official SEN2NAIP/SEN2NAIPv2 release is TACO-format and that no TACO-to-NumPy converter has been built or tested against a real archive. The SEN2Vénus loader similarly uses the documented local array convention, not a verified official release adapter.

**Impact:** the user cannot point the current loaders directly at the official dataset releases and expect them to work. A data conversion and verification stage is still needed.

**Fix:** implement and test official-format adapters/converters, pin the upstream dataset release/version, validate band order/units/metadata, and create sealed provenance manifests from the original source assets and derived patches.

### P1 — Cartosat is not a complete acquisition-to-validation path
`src/datasets/cartosat_pairing.py` accepts already extracted arrays. Its docstring assigns portal download, orthorectification, band mapping, CRS reprojection, and common-grid preparation to the caller. No real Cartosat source data is included.

**Impact:** the project's central India-specific differentiator is not yet demonstrated. An array alignment utility is not equivalent to a reproducible Cartosat acquisition/orthorectification pipeline or validated Indian ground truth.

**Fix:** acquire an authorized real Cartosat-2S/3 scene; record license/order metadata; build rasterio/GDAL ingestion and orthorectification/reprojection; harmonize sensor response and spectral bands; preserve source CRS/affine metadata; create sealed source/pair manifests; evaluate on held-out Indian scenes.

### P1 — Cartosat reflectance harmonization is a prototype statistical adjustment
The current Cartosat pairing code applies per-band affine mean/std matching between the upsampled LR and HR arrays. Its own docstring says this is not a substitute for a calibrated sensor-response model.

**Risk:** matching means and standard deviations can conceal radiometric/spectral differences and does not model spectral response functions, atmospheric correction, acquisition timing, illumination, or sensor-specific radiometry. It can also remove genuine scene differences if the overlap is not carefully controlled.

**Fix:** document the harmonization as a provisional alignment aid; establish sensor/product-specific calibration and compare results with/without harmonization. Do not call this calibrated reflectance equivalence.

### P1 — No real-data result or trained checkpoint is evidenced in the repository
The README explicitly says all tests use synthetic data and no real Sentinel-2/NAIP/Cartosat/WorldCover downloads were performed in the build environment. It also says GPU-scale training has not been run. The tracked tree includes one WorldCover GeoTIFF, but no training pair directories or trained checkpoint.

The achievement report's description of a "fully functional" pipeline should therefore be read as code-path/prototype completeness, not proof of an end-to-end trained model evaluated on real scenes. No real benchmark metrics, uncertainty calibration result, or Indian holdout result can be independently checked from the tree reviewed here.

**Fix:** separate software smoke-test status from real-data/model validation status in all demos and presentations. Publish checkpoint provenance, data manifests, training logs, held-out per-scene results, uncertainty coverage, and downstream metrics before making performance claims.

### P2 — Legacy evaluation script is not the frozen benchmark protocol
`scripts/run_evaluation.py`:
- Uses a sorted pair list and a simple last-20%-of-pairs split.
- Does not group by scene/AOI or read the scene split manifest.
- Does not use the frozen evaluation contract.
- Crops arrays to their minimum common dimensions rather than requiring a verified common grid.
- Does not apply a validity mask.
- Uses simplified global SSIM and its own PSNR/SAM/NDVI routines rather than the canonical metrics module.
- Reports pair-level aggregate means rather than scene-level bootstrap intervals.
- Does not evaluate uncertainty calibration or downstream class IoU.
- Loads checkpoints with `weights_only=False`.

**Fix:** deprecate it or refactor it to call the canonical benchmark/evaluation contract, scene manifests, masks, and safe checkpoint loader. Do not use its output as the official SIH26142 validation report.

### P2 — Streamlit app does not use the robust inference path
`app.py` loads the model/checkpoint itself rather than using `src.infer.load_checkpoint()` and its safe deserialization, preflight validation, geospatial I/O, tiled inference, and telemetry path. It reads pre-materialized `.npy` pairs from `data/sen2naip`, divides LR input by 10,000, and runs `predict()` on the selected patch.

**Fix:** make the UI call the shared inference API; add file upload only with explicit input-size and file-format safeguards; make the UI's data units and normalization contract match the dataset adapter and checkpoint manifest.

### P2 — Model uncertainty is not automatically calibrated by predicting variance
A heteroscedastic Gaussian NLL head produces a variance estimate, but the existence of a variance map does not prove statistical calibration or epistemic uncertainty. The current head primarily models aleatoric/residual uncertainty under its assumed loss/data distribution. The project has an uncertainty calibration implementation and required coverage metrics in the evaluation contract, but no real held-out calibration report is present.

**Fix:** report held-out Gaussian NLL, mean standardized squared error, and empirical coverage at 50/80/90/95%; stratify by land-cover, cloud/edge conditions, and geographic scene; avoid presenting a visualization heatmap alone as calibrated confidence.

### P2 — Data-pair alignment and channel mapping need real sensor validation
- SEN2NAIP registration uses integer-pixel NCC search and pair rejection; it is not a full sub-pixel, sensor-aware co-registration system.
- Cartosat channel matching can repeat a single HR band or truncate extra HR bands to match LR channel count. That is a convenience fallback, not valid spectral band correspondence.
- Grid-aware pairing rejects incompatible CRS/pixel phases when grid metadata is supplied, but array-only callers can still operate without geospatial metadata under shape/registration QC.
- The current pairing code must be tested with real projected scenes and product-specific metadata before claiming geographic accuracy.

**Fix:** require explicit band mapping and grid metadata for real evaluation; use appropriate resampling and sub-pixel co-registration where needed; audit residual alignment visually and numerically on held-out scenes.

### P2 — Benchmark contract enforcement is partial at the call boundary
The contract validator thoroughly checks the declared policy, and `src/benchmark.py` requires verified-real status and both manifest references when `claim_real=True`. However, a caller-supplied metadata flag and non-empty manifest path strings do not by themselves prove that manifests were loaded, verified, and match the exact in-memory scenes being scored.

**Fix:** have the benchmark runner load/verify manifests itself, bind scene IDs and file hashes to the evaluated arrays, and validate split membership against the exact test scenes. Keep the current evidence gate but strengthen it from declarative metadata to verified provenance.

### P2 — Scene protocol manifest validation could be stricter
`validate_scene_split_manifest()` checks schema version, required mappings, cross-split scene/sample intersections, and optional sample coverage. It does not fully prove that each sample ID is assigned to the scene it claims to originate from, nor does it reject duplicate sample IDs repeated within a single split. Add a source-record reconciliation step that validates the complete scene-to-sample mapping, not only cross-split overlap.

### P2 — Runtime/resource controls need deployment validation
- Tiled inference and preflight byte/pixel limits are useful controls, but actual GPU memory and throughput are unverified.
- The model's attention operates on local windows, but full-scene memory still depends on tile size, overlap, bands, and device.
- Streamlit cache/resource lifecycle and stale-output handling should be verified with real model/checkpoint changes and multiple sessions.
- Telemetry is best-effort by default; failures to write telemetry are intentionally swallowed unless strict mode is selected.

## 7. Security and reliability notes

### Positive controls
- Main CLI inference uses `weights_only=True` with a limited NumPy safe-global allowlist.
- Training resume also uses `weights_only=True`.
- Inference validates dimensions and memory limits before tensor allocation.
- Output/checkpoint writing generally uses temp files plus atomic replacement.
- Dataset provenance includes per-file SHA-256 hashes and a sealed manifest.
- Geospatial pairing supports CRS/affine contracts and explicit nodata masks.
- Benchmark code distinguishes development fixtures from verified-real evidence.
- Training uses deterministic seeds and records model/run configuration and dataset fingerprint.

### Remaining controls
- Remove unrestricted checkpoint deserialization from every alternate entry point.
- Treat all external imagery/checkpoints as untrusted inputs; validate archive extraction paths and raster dimensions in any future downloader.
- Keep dataset source/license metadata with provenance records.
- Avoid claiming a sealed manifest makes the source authentic; it detects changes relative to the recorded hashes but does not independently prove who supplied the source or whether it is licensed.
- Use explicit allowlisted band mappings rather than repeat/truncate semantics for scientific sensor correspondence.
- Add CI security checks and dependency scanning when a CI workflow is introduced.

## 8. Tests, execution evidence, and CI

### Test structure
The repository contains 21 executable `test_*.py` scripts, including:
- Model runtime and inference.
- Training and reproducibility.
- Preprocessing and metrics.
- SEN2NAIP/SEN2Vénus/Cartosat pairing.
- Geospatial grid contracts and WorldCover.
- Dataset provenance.
- Evaluation contract, benchmark, and scene protocol.
- Telemetry and audit regression fixes.

`run_tests.py` recursively discovers both root-level and nested test scripts and runs each in its own subprocess, returning non-zero if any fail. This avoids silently skipping nested tests.

The README and build log record many synthetic-data CPU test passes from earlier agent turns. This review did not rerun them, and those historical test claims do not establish real data, GPU, or scientific validation.

### CI
No GitHub Actions workflow was found. The canonical local runner is not automatically enforced by repository CI.

**Recommended CI gates:**
1. Install a pinned Python environment.
2. Run `python run_tests.py`.
3. Add lint/type checks and dependency/security scanning.
4. Add a small CPU model/inference integration test.
5. Verify safe checkpoint loading on all entry points.
6. Validate evaluation contract and manifest integrity.
7. Keep real-data benchmark runs separate and explicitly labeled, using secured datasets and scene-level test manifests.

## 9. Documentation and project process

The repository includes unusually extensive project history:
- `AGENTS.md`: multi-model research deliberation protocol, closed at solution draft v11.
- `BUILD_AGENTS.md`: sequential agent task-taking/implementation protocol.
- `tasks.md`: implementation task board and verification notes.
- `build-status.md`: build turn history and verification claims.
- `status.md`: research-round consensus state.
- `bugs.md`, `BUGFIXES_APPLIED.md`, `external_gap_audit.md`: audit findings and fixes.
- `research/`, `wide_research/`, and `log/`: research evidence and iterative critique.
- `REPORT.md`, `SIH_Presentation_Script.md`, and `presentation/`: achievement/presentation materials.

**Important process note:** `BUILD_AGENTS.md` says an agent already in its roster must only act when `build-status.md` names it in `next_agent`. The current file names `agent4`. This repository review is documentation/context analysis, not a build-task turn, and did not modify the build task board.

### Documentation inconsistencies to reconcile
- `REPORT.md` (dated 2026-09-25) uses "fully functional" wording; README and gap analysis make clear real-data training/validation are not yet completed. Use the more precise "prototype pipeline implemented; real-data validation pending" framing.
- `BUGFIXES_APPLIED.md` describes the inference checkpoint security fix, but the Streamlit and legacy evaluation entry points still use unrestricted loading.
- `bugs.md` includes findings that later code changes appear to have fixed (for example, masked overlap fraction in `metrics.py`); treat it as a historical audit log, not a current list of open defects.
- `requirements.txt` uses broad lower bounds rather than a lock/pinned reproducibility environment. A Python version is checked by setup scripts, but the dependency set can drift over time.
- Setup scripts install packages and execute the test runner; they do not download real imagery, train a model, or acquire Cartosat data.

## 10. Recommended remediation order

### Immediate engineering
1. Replace `weights_only=False` in `app.py` and `scripts/run_evaluation.py` with the shared safe checkpoint loader.
2. Integrate persisted scene-disjoint manifests into `src/train.py`; ensure validation and final test are distinct scene sets.
3. Build and verify official SEN2NAIP/TACO and SEN2Vénus data adapters.
4. Enforce explicit band mapping, reflectance units, CRS/grid metadata, and masks for all real evaluation inputs.
5. Complete Cartosat acquisition, orthorectification, reprojection, and source/pair provenance workflow.
6. Make the benchmark runner verify manifests and scene IDs rather than trusting caller-supplied flags/paths.

### Scientific validation
7. Train on real paired data and publish reproducible checkpoint/run manifests.
8. Compare against bicubic and appropriate published baselines using the same scene IDs, masks, and preprocessing.
9. Report per-scene confidence intervals for image-quality and downstream metrics.
10. Measure uncertainty calibration on held-out scenes and condition strata.
11. Validate downstream land-cover utility with real WorldCover labels and a defensible classifier/protocol.
12. Run the India Cartosat holdout and clearly separate cross-sensor generalization from same-sensor benchmark results.

### Delivery maturity
13. Add CI and pinned dependency/environment metadata.
14. Test tiled inference on actual full scenes and target hardware.
15. Update demo/presentation claims to distinguish implementation, synthetic test verification, real-data results, and deployment readiness.

## 11. Coding-agent handoff

Preserve these design choices:
- Keep the model's explicit mean and log-variance outputs.
- Keep invalid-pixel masks propagated through normalization, loss, image metrics, downstream metrics, and uncertainty metrics.
- Keep the canonical blue/green/red/NIR band order explicit.
- Keep scene-level splits and test-set exclusion as evaluation invariants.
- Keep SHA-256 source-file provenance and reproducible run manifests.
- Keep geospatial CRS/affine metadata attached through pairing, cropping, and export.
- Keep the verified-real benchmark gate; do not silently relabel synthetic fixtures as real data.
- Prefer shared inference/checkpoint APIs over new one-off `torch.load` implementations.

## 12. Review limitations

- Static source and documentation inspection only.
- Repository tree was complete (140 entries).
- 21 test scripts were identified but not executed.
- No dependencies were installed.
- No model training, checkpoint loading, real dataset download, real Cartosat processing, GPU run, or benchmark execution was performed.
- Findings about actual runtime and scientific performance are limited to documented historical execution evidence and source-level review.
