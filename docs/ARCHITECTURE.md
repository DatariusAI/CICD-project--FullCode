# Architecture -- ML CI/CD Pipeline (Used Car Price Prediction)

## Overview

This project implements an MLOps CI/CD pipeline for predicting used car prices using a Random Forest Regressor. GitHub Actions orchestrates Azure Machine Learning to automate compute provisioning, data registration, environment setup, model training with hyperparameter sweep, and model registration. The codebase is structured as a guided exercise with scaffolded sections for learners to complete.

## Pipeline Architecture

```
+===========================================================================+
|                          GitHub Repository                                |
|   Push / PR to main                                                       |
+===========================================================================+
            |
            v
+===========================================================================+
|              GitHub Actions Workflow (CI/CD Orchestrator)                  |
|              deploy-model-training-pipeline-classical.yml                  |
|---------------------------------------------------------------------------|
|  1. get-config    -- reads config-infra-prod.yml for Azure settings       |
+===========================================================================+
            |
            v  (three parallel jobs)
+-----------------+  +---------------------+  +------------------------+
| create-compute  |  | register-dataset    |  | register-environment   |
| Standard_DS11_v2|  | used_cars.csv as    |  | train-conda.yml as     |
| (0-1 nodes)     |  | used-cars-data      |  | used-cars-train-env    |
+-----------------+  +---------------------+  +------------------------+
         \                    |                       /
          \                   |                      /
           v                  v                     v
+===========================================================================+
|              Azure ML Training Pipeline (newpipeline.yml)                 |
|---------------------------------------------------------------------------|
|                                                                           |
|  +------------------+     +---------------------+     +-----------------+ |
|  |  prep_data       |---->|  sweep_step         |---->|  register_model | |
|  |  (prep.py)       |     |  (train.yml x 20)   |     |  (register.py)  | |
|  +------------------+     +---------------------+     +-----------------+ |
|  | - Read CSV       |     | - Random sampling    |     | - Load best     | |
|  | - Label encode   |     | - n_estimators:      |     |   sweep model   | |
|  |   categoricals   |     |   10, 20, 30, 50     |     | - Register as   | |
|  | - 80/20 split    |     | - max_depth:         |     |   used_cars_    | |
|  | - Save train.csv |     |   choice values      |     |   price_pred-   | |
|  |   and test.csv   |     | - 20 total trials    |     |   iction_model  | |
|  | - Log sizes to   |     | - Minimize MSE       |     | - Write model   | |
|  |   MLflow         |     |                      |     |   info JSON     | |
|  +------------------+     +---------------------+     +-----------------+ |
|                                                                           |
+===========================================================================+
            |
            v
+===========================================================================+
|  Azure ML Model Registry                                                  |
|  - used_cars_price_prediction_model (MLflow format)                       |
|  - Versioned, tracked, ready for deployment                               |
+===========================================================================+
```

## Pipeline Stages in Detail

### Stage 1: Data Preparation (`prep.py`)

- **Input:** Raw `used_cars.csv` dataset
- **Process:**
  1. Reads CSV into a pandas DataFrame
  2. Applies `LabelEncoder` to convert categorical features (e.g., make, model, fuel type) into numerical values
  3. Performs 80/20 train-test split with `random_state=42`
  4. Saves `train.csv` and `test.csv` to output directories
- **Tracking:** Logs `train size` and `test size` metrics to MLflow

### Stage 2: Hyperparameter Sweep (`train.yml` + `train.py`)

- **Algorithm:** `RandomForestRegressor` from scikit-learn
- **Sweep Strategy:** Random sampling over the search space
- **Search Space:**
  - `n_estimators`: choice of `10`, `20`, `30`, `50`
  - `max_depth`: choice values (to be configured in the exercise)
- **Limits:** 20 total trials, 10 concurrent, 7200s timeout
- **Objective:** Minimize `MSE` (Mean Squared Error) on the test set
- **Tracking:** Each trial logs hyperparameters and MSE to MLflow

### Stage 3: Model Registration (`register.py`)

- **Input:** Best model from the sweep (MLflow sklearn format)
- **Process:** Loads model, logs to MLflow, registers in Azure ML model registry
- **Output:** `used_cars_price_prediction_model` with version number, plus `model_info.json`

## Azure ML Components Used

| Component | Purpose |
|-----------|---------|
| **Compute Cluster** | `Standard_DS11_v2` (0-1 nodes, dedicated tier) for training |
| **Data Asset** | `used-cars-data` -- registered URI file pointing to `used_cars.csv` |
| **Environment** | `used-cars-train-env` -- Conda environment with sklearn, mlflow, pandas |
| **Pipeline Job** | Orchestrates prep, sweep, and register steps sequentially |
| **Sweep Job** | Random hyperparameter search with 20 trials |
| **Model Registry** | Stores versioned MLflow models |

## GitHub Actions Workflow

The main workflow (`deploy-model-training-pipeline-classical.yml`) follows this dependency graph:

```
get-config (reads YAML)
    |
    +---> create-compute   (parallel)
    +---> register-dataset (parallel)
    +---> register-environment (parallel)
    |
    v
run-pipeline (depends on all three above)
```

Each step uses a **reusable workflow** from `.github/workflows/`:

| Workflow File | Purpose |
|---------------|---------|
| `custom-create-compute.yml` | Provisions Azure ML compute cluster |
| `custom-register-dataset.yml` | Registers dataset as Azure ML data asset |
| `custom-register-environment.yml` | Registers conda environment in Azure ML |
| `custom-run-pipeline.yml` | Submits the Azure ML pipeline job |

## Authentication

All workflows authenticate to Azure using the `AZURE_CREDENTIALS` GitHub secret, which contains a service principal JSON with contributor access to the Azure ML workspace.

## Exercise Structure

This project is designed as a learning exercise. Key files contain placeholders (`_______`, `WRITE YOUR CODE HERE`) that learners must complete:

| File | What to Implement |
|------|-------------------|
| `data-science/src/prep.py` | Argument parsing types, label encoding, train-test split, metric logging |
| `data-science/src/train.py` | Argument definitions, model training, MSE calculation, MLflow logging |
| `data-science/src/register.py` | Argument types, model loading, registration, JSON output |
| `mlops/azureml/train/newpipeline.yml` | Display names, output types, job names, sweep configuration |

## Key Configuration Files

| File | Purpose |
|------|---------|
| `config-infra-prod.yml` | Azure resource group and workspace names, VM image, location |
| `data-science/environment/train-conda.yml` | Python + sklearn + mlflow + pandas dependencies |
| `mlops/azureml/train/data.yml` | Azure ML data asset definition |
| `mlops/azureml/train/train-env.yml` | Azure ML environment definition |
| `mlops/azureml/train/train.yml` | Training command component (used by sweep) |
| `mlops/azureml/train/newpipeline.yml` | Full pipeline definition (exercise) |
