---
name: mlops-quickstart-overview
description: >-
  Reference for the Databricks MLOps Quickstart repo structure, parameterization
  contract, and Challenger/Champion conventions. Use whenever the user asks
  about repo layout, naming patterns, three-level Unity Catalog references,
  notebook widgets, MLflow experiment naming, or serving endpoint naming, or
  before adapting any notebook, job, or bundle in this repo.
---

# Databricks MLOps Quickstart — Overview

This repository is an end-to-end MLOps template on Databricks. It uses the Iris
classification dataset as a placeholder — it is meant to be adapted to any
dataset and ML problem type (regression, clustering, classification,
forecasting, etc.).

When the user wants to adapt a specific area, load the matching skill:

| Area | Skill |
|------|-------|
| Custom data source / ingestion | `adapt-data-ingestion` |
| Custom algorithm or problem type | `adapt-model-training` |
| MLflow 3 evaluate → approve → deploy pipeline | `adapt-model-deployment` |
| Batch or realtime inference | `adapt-inference` |
| `databricks.yml`, `resources/*.yml`, CI/CD | `adapt-bundle-and-cicd` |
| Python library versions / pip dependencies | `manage-dependencies` |

## Repository Structure

```
notebooks/
  1_data_preprocessing/     → Data ingestion & feature engineering
  2_model_training_and_deployment/
    model_training.ipynb    → Train & register model in Unity Catalog
    model_deployment/       → MLflow 3 deployment pipeline (evaluate → approve → deploy)
  3_inference/
    batch_inference.ipynb   → Batch scoring with the Champion model
    realtime_inference.ipynb → Serving endpoint example
resources/                  → Databricks Asset Bundle job definitions (one YAML per job)
databricks.yml              → Bundle config with dev/prod targets and shared variables
azure_pipelines.yml         → CD pipeline for Azure DevOps
.github/workflows/          → GitHub Actions CD pipeline (alternative)
```

## Parameterization Contract

All notebooks accept parameters via `dbutils.widgets`. Three core parameters
are wired through every notebook and job:

| Parameter | Purpose | Default |
|-----------|---------|---------|
| `catalog_name` | Unity Catalog catalog for all tables and models | Set per target in `databricks.yml` |
| `schema_name` | Schema within the catalog | `iris` |
| `model_version` | Model version (deployment notebooks only) | `1` |

Rules to keep this contract intact:

- **Three-level Unity Catalog references** always use
  `{catalog_name}.{schema_name}.<object>` — never hardcode a catalog or schema
  inside notebook code.
- Job YAMLs in `resources/` declare these as job-level `parameters` sourced
  from `${var.*}` bundle variables, then pass them to notebooks via
  `base_parameters`.
- If a notebook needs a new parameter, add it in **three places**: the widget
  in the notebook, the job `parameters` block, and the task `base_parameters`.

## Key Conventions

- **Challenger/Champion pattern**: newly trained models are registered with
  the `@challenger` alias. After passing evaluation and approval, the
  deployment notebook promotes them to `@champion`. Inference notebooks
  always load the `@champion` alias.
- **MLflow experiment naming**: experiments are scoped per user and catalog —
  `/{user}/{model}_{catalog}` — to avoid collisions during development.
- **Serving endpoint naming**: derived as
  `{catalog_name}-{schema_name}-{model}-endpoint` (dots replaced with dashes)
  to ensure valid endpoint names.
- **Idempotent notebooks**: ingestion and inference notebooks check for table
  existence before deciding whether to create vs. append/overwrite. Preserve
  this behavior when adapting.
- **Default compute**: jobs without an explicit `job_clusters` block run on
  serverless. Dedicated job compute can be enabled by uncommenting the
  `job_clusters` section in the job YAML.
- **Pinned dependencies**: every library version is pinned in
  `requirements.txt` at the repo root. Each notebook installs from it with a
  single `%pip install -r ../../requirements.txt` (or `../../../` for
  deployment notebooks). Never pin a version inline. See `manage-dependencies`.

## Adaptation Workflow

When a user clones this repo to build their own ML solution, work through the
adaptation areas in this order — and load the matching skill for each step:

1. **Data** → `adapt-data-ingestion`
2. **Model training** → `adapt-model-training`
3. **Deployment pipeline** → `adapt-model-deployment`
4. **Inference** → `adapt-inference`
5. **Bundle config + CI/CD** → `adapt-bundle-and-cicd`

Each adapt-* skill is self-contained for its area but assumes the conventions
in this overview hold.
