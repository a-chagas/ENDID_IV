# Replication Package: PPML First Stage + IV Second Stage (Two-Stage Bootstrap)

This project estimates:

1. **First stage (PPML):** bilateral trade (or flows) with fixed effects using `fixest::fepois`, producing predicted bilateral flows and a **country-year exposure instrument** (e.g., `WD_row_hat`).
2. **Second stage (IV):** country-year outcomes (e.g., `ln_production`) on the endogenous exposure `WD_row_obs`, instrumented by `WD_row_hat` using `fixest::feols` IV syntax.
3. **Inference:** a two-stage bootstrap that (i) draws PPML coefficients from a multivariate normal approximation and (ii) re-samples country clusters, then re-estimates stage 2 and stage 3.

A machine-readable summary of required inputs and key parameters is in `replication_config.csv`.

## 1) Software requirements

- R (>= 4.1 recommended)
- Packages: `data.table`, `fixest`, `MASS`, `foreach`, `doParallel`, `doRNG`

Install:

```r
install.packages(c("data.table","fixest","MASS","foreach","doParallel","doRNG"))
```

## 2) Folder structure

Place the script in the project root and organize data as:

```
<project_root>/
  data_article/
    dt_bilateral_wheat.rds
    df_country_year_wheat.rds
  output/
  your_script.R
```

The script creates `output/` automatically (if not present).

## 3) Input data

### 3.1 Bilateral data: `data_article/dt_bilateral_wheat.rds`

Bilateral panel at the `(exporter i, importer j, year t)` level used in the PPML stage.

Minimum required fields include:

- Identifiers and FE: `isoi`, `isoj`, `year`, `iso_pair`
- PPML clustering variables: `cl_iy`, `cl_jy`
- Dependent variable: `value`
- Shock variable: `speij_d_crop07` (or whatever you pass as `shock_var`)
- All regressors in `rhs_ppml`

### 3.2 Country-year data: `data_article/df_country_year_wheat.rds`

Country-year panel with one row per `(iso, year)` used in the IV stage.

Minimum required fields include:

- Merge key: `iso`, `year`
- Fixed effects: `isoid`, `year` (or your chosen FE)
- Endogenous regressor: `WD_row_obs`
- Outcomes: `ln_production` (and optionally `ln_area`, `ln_yield`)
- Controls in `rhs_stage2` (e.g., `temp`, `temp2`, `precip`, `precip2`, `irrigation2`, `spei_d_crop07`)

## 4) How to run

From the project root:

```r
source("your_script.R")
```

Key printed output is the bootstrap-based coefficient table, e.g.:

```r
summary(resrob$ln_production$stage3$coeftest)
```

## 5) Main objects

### 5.1 First stage (`mrob`)
- `mrob$model`: PPML `fixest` model
- `mrob$dt_for_W`: data used to recompute the instrument inside the bootstrap
- `mrob$beta_hat`, `mrob$vcv`: coefficient vector and clustered vcov
- `mrob$W_hat`: instrument at `(iso, year)` (column name `WD_row_hat`)

### 5.2 Second stage (`resrob`)
For each outcome (e.g., `ln_production`):
- `resrob$ln_production$stage3$model_point`: point IV estimate (`fixest` object)
- `resrob$ln_production$stage3$se_boot`: bootstrap standard errors
- `resrob$ln_production$stage3$coeftest`: printable coefficient table

## 6) Customization

- Outcomes: edit `dep_vars` in `run_three_models(...)`.
- Controls: edit `rhs_stage2`.
- PPML RHS: edit `rhs_ppml` (and ensure variables exist in `dt`).
- Bootstrap: edit `B`, `n_cores`, `chunk_size`, and `seed`.

## 7) Reproducibility

The bootstrap uses `doRNG`, so results are reproducible given the same seed, data, and software versions (minor floating point differences across platforms may remain).

## 8) Troubleshooting

- If bootstrap standard errors are `NA`, too many draws likely failed (collinearity / not enough variation within sampled clusters). Try increasing `B`, simplifying the model, or verifying that `cluster_id` has many clusters.
- If you get “cluster id(s) missing”, confirm those columns exist in `df_iv`.

