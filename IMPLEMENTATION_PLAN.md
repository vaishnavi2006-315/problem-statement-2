# IMPLEMENTATION PLAN — Hybrid AI–NWP Adaptive Forecast Blending System

Team BITCODE · SIH 2026 · Problem Statement 26081

## 1. Scope decision for this prototype

This is a from-scratch build (no existing repo found). Full production-scale NWP
ingestion (real GRIB/NetCDF pulls from ECMWF, INSAT, GPM) is out of reach for a
hackathon prototype and not needed to prove the core idea. Instead:

- A **Demo Data Engine** generates deterministic sample fields over India
  (0.25° grid, 61×97 points, lat 6–38N, lon 68–98E) for IFS, ENS(mean+spread),
  AIFS, ERA5, INSAT (convective proxy), GPM IMERG, across 5 named scenarios and
  3 lead times (24/48/72h). Fields are generated with real numpy computation
  (spatial Gaussian bumps + noise + regime-conditioned structure) — nothing is a
  static PNG or a hardcoded number table.
- Every downstream stage (QC, regridding, regime detection, adaptive weighting,
  spatial blending, extreme detection, verification) runs **real code** on these
  arrays. Weights are the output of a trained LightGBM model + a softmax over
  predicted skill, not fixed constants. The U-Net is a real small PyTorch module
  that does a real forward pass over the blended field.
- Verification metrics may be computed internally against the synthetic demo
   reference, but the API marks them unavailable for operational display because
   they are not observations.
- This architecture is designed so demo data sources can be swapped for real
  ECMWF/INSAT/GPM downloads later without changing pipeline code — only
  `app/data/*` ingestion adapters change.

GraphCast is not used anywhere; ECMWF AIFS is the AI model throughout.

## 2. Architecture

```
frontend/ (React+TS+Vite+Tailwind+Recharts+Leaflet)
   |  fetch() JSON
   v
backend/ (FastAPI)
   ├── data/          demo scenario generator + adapters (ForecastField model)
   ├── preprocessing/ QC, alignment, common grid, feature engineering
   ├── ml/
   │     regime.py       rule/clustering-based weather regime detection
   │     weighting.py    LightGBM adaptive gate -> per-pixel model weights
   │     blender.py      weighted blend + safety normalization
   │     unet.py         PyTorch spatial refinement network
   │     extreme.py      extreme-event probability model
   ├── verification/ MAE/RMSE/CRPS/POD/FAR/CSI/F1/Brier vs ERA5/GPM truth
   ├── explainability/ builds the "why this weight" driver object
   ├── database/      SQLite (forecast_runs, model_weights, extreme_events,
   │                   verification_results, pipeline_logs)
   └── api/           REST endpoints (see section 5)
```

## 3. Data flow (per `/forecast/run` call)

1. Load scenario+region+lead_time demo fields (IFS, ENS mean/spread, AIFS, ERA5,
   INSAT, GPM) → `ForecastField` objects.
2. QC: clip physically impossible values, flag/interpolate NaNs, mark source
   availability.
3. Align to common 0.25° grid (already common in demo; interpolation path used
   when a source's native grid differs, so real regridding code exists and is
   exercised in tests).
4. Feature engineering: ENS spread, INSAT convective index, IFS-AIFS
   disagreement, latitude/season/lead-time encodings.
5. Regime detection → {CLEAR, STRATIFORM, CONVECTIVE, CYCLONIC, DISTURBED}.
6. Historical skill lookup (per region×regime×lead_time error table, built from
   a simulated verification history at startup).
7. LightGBM adaptive gate predicts each model's expected error per grid cell →
   converted to normalized trust weights (softmax of negative error).
8. Spatial U-Net takes the 3 weighted fields + features, outputs the refined
   blended field.
9. Extreme-event head: probability + threshold + risk from a logistic function
   of blended intensity, spread, and regime.
10. Confidence score from spread, model agreement, regime certainty, historical
    skill.
11. Verification module scores IFS/ENS/AIFS/fixed-blend/adaptive-blend against
    ERA5+GPM truth for the run.
12. Explainability object assembled (weights + ranked drivers).
13. Everything persisted to SQLite; API returns JSON; frontend renders.

## 4. Database schema

See `backend/app/database/db.py` — tables: `forecast_runs`, `model_weights`,
`extreme_events`, `verification_results`, `pipeline_logs`, plus
`historical_skill` (support table for step 6).

## 5. API design

`/health`, `/sources`, `/ingest`, `/preprocess`, `/regime/detect`,
`/blend/predict`, `/extreme/analyze`, `/forecast/run`, `/verify`,
`/forecast/latest`, `/model-weights`, `/explanation`, `/pipeline/status`,
`/scenarios`, `/regions` — see `backend/app/api/routes.py` for schemas.

## 6. Frontend pages

Landing, Login/Demo, Dashboard (India map + variable/lead controls + key
insights + regime/trust/risk/confidence cards), Location Forecast, Forecast
Maps (6-variable grid), Model Contribution & Explainability, Extreme Event
Alerts, Analytics & Verification, Data Sources, About/Methodology, Settings.
Map rendering uses an SVG India outline + canvas grid-heatmap overlay (no
external tile server) so the demo works fully offline, per the "must run
without internet" requirement.

## 7. Development phases actually followed

1 Project setup → 2 Demo dataset → 3 Ingestion → 4 Preprocessing → 5 Fixed
baseline → 6 LightGBM adaptive blend → 7 Regime detection → 8 Extreme engine →
9 Verification → 10 FastAPI → 11 React dashboard → 12 Maps → 13 U-Net → 14
Explainability → 15 Demo replay/scenarios → 16 Tests → 17 Docker → 18 Polish.

## 8. Known simplifications (disclosed, not hidden)

- Demo fields are synthetically generated, not downloaded from ECMWF/INSAT/GPM
  (no internet access in this environment; architecture supports swapping in
  real adapters later — see `backend/app/data/README.md`).
- The LightGBM gate is trained at startup on a synthetic archive for demo
   execution only. The U-Net is an untrained architecture with no checkpoint.
   Live source adapters and production-trained model artifacts are not configured.
   Demo verification is not presented as real-world skill in the UI.
