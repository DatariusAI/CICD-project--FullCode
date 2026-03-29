# ML CI/CD Pipeline -- Used Car Price Prediction

<div align="center">

![Azure ML](https://img.shields.io/badge/Azure%20ML-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.x-3776AB?style=for-the-badge&logo=python&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?style=for-the-badge&logo=scikitlearn&logoColor=white)
![MLflow](https://img.shields.io/badge/MLflow-0194E2?style=for-the-badge&logo=mlflow&logoColor=white)

**Hands-on MLOps exercise: build an automated Azure ML pipeline with GitHub Actions CI/CD for used car price prediction using Random Forest regression.**

</div>

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Pipeline Stages](#pipeline-stages)
- [Repository Structure](#repository-structure)
- [Tech Stack](#tech-stack)
- [Setup & Configuration](#setup--configuration)
- [How It Works](#how-it-works)
- [Dataset](#dataset)
- [Exercise Instructions](#exercise-instructions)

---

## Overview

This repository is a **guided MLOps exercise** that teaches how to build a production CI/CD pipeline for machine learning on Azure. The project uses a used cars dataset and a Random Forest Regressor to predict vehicle prices. Code files contain scaffolded sections (marked with `WRITE YOUR CODE HERE`) where learners implement key components:

- Data preprocessing with label encoding
- Model training and evaluation
- Azure ML pipeline configuration
- Hyperparameter sweep setup

On every push or PR to `main`, GitHub Actions automates compute provisioning, data registration, environment setup, and model training on Azure ML.

---

## Architecture

```
GitHub (Push / PR to main)
        |
        v
+----------------------------+
|   GitHub Actions Workflow   |
|  deploy-model-training-     |
|  pipeline-classical.yml     |
+----------------------------+
        |
        v  (parallel jobs)
+-------------+  +------------------+  +---------------------+
| Create      |  | Register Dataset |  | Register Conda      |
| Compute     |  | (used_cars.csv)  |  | Environment         |
| Cluster     |  |                  |  |                     |
+-------------+  +------------------+  +---------------------+
        \               |                /
         \              |               /
          v             v              v
    +------------------------------------+
    |    Azure ML Training Pipeline      |
    |------------------------------------|
    |  1. prep.py  -- encode & split     |
    |  2. sweep    -- tune n_estimators  |
    |               & max_depth          |
    |  3. register -- save best model    |
    +------------------------------------+
              |
              v
    +--------------------+
    | Registered Model   |
    | (MLflow format)    |
    +--------------------+
```

---

## Pipeline Stages

| Stage | Script / Config | Description |
|-------|----------------|-------------|
| **Data Preparation** | `data-science/src/prep.py` | Reads raw CSV, applies label encoding to categorical features, performs train-test split, logs dataset sizes |
| **Hyperparameter Sweep** | `mlops/azureml/train/train.yml` | Sweeps over `n_estimators` (10, 20, 30, 50) and `max_depth` values using random sampling -- 20 total trials |
| **Model Training** | `data-science/src/train.py` | Trains a `RandomForestRegressor`, evaluates MSE on test set, logs metrics to MLflow |
| **Model Registration** | `data-science/src/register.py` | Registers the best sweep model in Azure ML as `used_cars_price_prediction_model` |

---

## Repository Structure

```
CICD-project--FullCode/
|
|-- config-infra-prod.yml                  # Azure infra config (resource group, workspace)
|
|-- data/
|   +-- used_cars.csv                      # Used cars dataset
|
|-- data-science/
|   |-- environment/
|   |   +-- train-conda.yml                # Conda environment specification
|   +-- src/
|       |-- prep.py                        # Data prep with label encoding (exercise)
|       |-- train.py                       # Random Forest training (exercise)
|       +-- register.py                    # Model registration to Azure ML
|
|-- mlops/azureml/train/
|   |-- data.yml                           # Azure ML data asset definition
|   |-- train-env.yml                      # Azure ML environment definition
|   |-- train.yml                          # Training command component (for sweep)
|   +-- newpipeline.yml                    # Full pipeline definition (exercise)
|
+-- .github/workflows/
    |-- deploy-model-training-pipeline-classical.yml   # Main CI/CD orchestration
    |-- custom-create-compute.yml                      # Reusable: provision compute
    |-- custom-register-dataset.yml                    # Reusable: register dataset
    |-- custom-register-environment.yml                # Reusable: register environment
    +-- custom-run-pipeline.yml                        # Reusable: submit pipeline job
```

---

## Tech Stack

| Category | Technology |
|----------|-----------|
| **Cloud Platform** | Microsoft Azure (Azure Machine Learning) |
| **CI/CD** | GitHub Actions |
| **ML Framework** | scikit-learn (RandomForestRegressor) |
| **Experiment Tracking** | MLflow |
| **Language** | Python 3.x |
| **Data Processing** | Pandas, LabelEncoder |
| **Compute** | Azure ML Compute Cluster (Standard_DS11_v2) |

---

## Setup & Configuration

### Prerequisites

- Azure subscription with an Azure ML workspace
- GitHub repository with Actions enabled
- Azure credentials stored as a GitHub secret (`AZURE_CREDENTIALS`)

### 1. Configure Azure Resources

Edit `config-infra-prod.yml` with your Azure resource group and workspace:

```yaml
resource_group: <your-resource-group-name>
aml_workspace: <your-aml-workspace-name>
```

### 2. Set GitHub Secrets

| Secret | Description |
|--------|-------------|
| `AZURE_CREDENTIALS` | Service principal credentials JSON for Azure authentication |

### 3. Trigger the Pipeline

```bash
git push origin main
```

The workflow triggers automatically on push or PR to `main`.

---

## How It Works

1. **GitHub Actions** detects a push/PR to `main` and reads the infrastructure config from `config-infra-prod.yml`.
2. **Parallel provisioning** -- compute cluster, dataset, and conda environment are set up simultaneously on Azure ML.
3. **Pipeline execution** on Azure ML:
   - `prep.py` applies **label encoding** to categorical features, then splits the data into train/test sets.
   - A **random hyperparameter sweep** trains up to 20 Random Forest models, minimizing **MSE**.
   - The best model is registered as `used_cars_price_prediction_model` in MLflow format.
4. All metrics and parameters are tracked in **MLflow** for experiment comparison.

---

## Dataset

**Used Cars Dataset**

| Property | Value |
|----------|-------|
| **Task** | Regression (Price Prediction) |
| **Target** | Vehicle price |
| **Preprocessing** | Label encoding for categorical features |
| **Primary Metric** | Mean Squared Error (minimized) |
| **Algorithm** | Random Forest Regressor |

---

## Exercise Instructions

This project is designed as a **learning exercise**. Files contain placeholders (`_______`, `WRITE YOUR CODE HERE`) that you need to complete:

| File | What to Implement |
|------|-------------------|
| `data-science/src/prep.py` | Argument parsing, label encoding, train-test split, metric logging |
| `data-science/src/train.py` | Argument parsing, model training, MSE calculation, MLflow logging |
| `mlops/azureml/train/newpipeline.yml` | Pipeline display names, output types, job names, sweep config |

### Steps

1. Fork this repository
2. Fill in the blanked-out sections in each file
3. Configure your Azure credentials as GitHub secrets
4. Push to `main` and watch the pipeline run in GitHub Actions

---

## Author

**DatariusAI** -- [github.com/DatariusAI](https://github.com/DatariusAI)
