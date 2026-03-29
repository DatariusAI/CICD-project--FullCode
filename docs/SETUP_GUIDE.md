# Setup Guide -- Used Car Price Prediction CI/CD Pipeline

Step-by-step instructions to configure and run the used car price prediction pipeline on Azure ML.

## Prerequisites

### Azure Requirements

1. **Azure Subscription** -- an active Azure account with billing enabled
2. **Azure Resource Group** -- a resource group to hold all ML resources
3. **Azure Machine Learning Workspace** -- an Azure ML workspace created within the resource group
4. **Service Principal** -- with Contributor role on the resource group

### Local Requirements

- Git
- Python 3.7+ (for local testing, optional)
- Azure CLI (optional, for manual resource management)

### GitHub Requirements

- A GitHub repository (fork of this project) with Actions enabled
- Permission to add repository secrets

## Step 1: Fork and Clone the Repository

```bash
# Fork on GitHub, then clone your fork
git clone https://github.com/<your-username>/CICD-project--FullCode.git
cd CICD-project--FullCode
```

## Step 2: Complete the Exercise Code

Before running the pipeline, fill in the scaffolded sections in these files:

### `data-science/src/prep.py`
- Set argument types to `str` and `float`
- Set default test-train ratio to `0.2`
- Implement label encoding for categorical features using `LabelEncoder`
- Implement `train_test_split`, save CSVs, and log metrics

### `data-science/src/train.py`
- Define arguments for `--train_data`, `--test_data`, `--model_output`, `--n_estimators`, `--max_depth`
- Read train/test CSVs, split into X/y, train `RandomForestRegressor`
- Log hyperparameters, compute MSE, save model with MLflow

### `data-science/src/register.py`
- Set argument types to `str`
- Load model, log with MLflow, register, and write `model_info.json`

### `mlops/azureml/train/newpipeline.yml`
- Fill in display names, experiment name, description
- Set output types to `uri_folder`
- Configure sweep job name, sampling algorithm (`random`), and max_depth values

## Step 3: Create Azure Resources

```bash
# Login to Azure
az login

# Create a resource group
az group create --name myResourceGroup --location eastus

# Create an Azure ML workspace
az ml workspace create \
  --name myMLWorkspace \
  --resource-group myResourceGroup \
  --location eastus
```

## Step 4: Create a Service Principal

```bash
az ad sp create-for-rbac \
  --name "used-cars-mlops-sp" \
  --role contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP> \
  --sdk-auth
```

Save the full JSON output from this command.

## Step 5: Configure GitHub Secrets

1. Go to your GitHub repository
2. Navigate to **Settings > Secrets and variables > Actions**
3. Click **New repository secret**
4. Add the following secret:

| Secret Name | Value |
|-------------|-------|
| `AZURE_CREDENTIALS` | The full JSON output from the service principal creation step |

## Step 6: Update Infrastructure Configuration

Edit `config-infra-prod.yml` with your Azure resource names:

```yaml
variables:
  resource_group: myResourceGroup        # <-- your resource group name
  aml_workspace: myMLWorkspace           # <-- your Azure ML workspace name
  location: eastus                       # <-- your preferred Azure region
```

## Step 7: Trigger the Pipeline

Push your completed code to trigger the CI/CD pipeline:

```bash
git add .
git commit -m "Complete exercise code and configure Azure resources"
git push origin main
```

The workflow runs automatically on push or PR to `main`.

## Step 8: Monitor Pipeline Execution

1. **GitHub Actions:** Watch the workflow in the **Actions** tab:
   - `get-config` reads Azure settings
   - `create-compute`, `register-dataset`, `register-environment` run in parallel
   - `run-pipeline` submits the Azure ML pipeline after all resources are ready

2. **Azure ML Studio** at [ml.azure.com](https://ml.azure.com):
   - View pipeline runs under **Jobs**
   - Inspect hyperparameter sweep trials and metrics
   - Check the registered `used_cars_price_prediction_model` under **Models**
   - Review MSE metrics in the experiment dashboard

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `AZURE_CREDENTIALS` authentication failure | Invalid or expired service principal | Regenerate the service principal and update the GitHub secret |
| `Resource group not found` | Incorrect name in `config-infra-prod.yml` | Verify the name matches your Azure subscription |
| `Workspace not found` | Workspace not created or name mismatch | Create the workspace or fix the name in config |
| Compute cluster creation timeout | Quota limits in the selected region | Try a different VM size or region; request quota increase |
| `Environment registration failed` | Conda dependency conflict | Check `data-science/environment/train-conda.yml` for version compatibility |
| Pipeline stuck in `Preparing` | Compute cluster scaling from 0 nodes | Wait 5-10 minutes for provisioning |
| `NameError` or `TypeError` in pipeline | Exercise code not completed correctly | Double-check all `_______` placeholders are filled in |
| `LabelEncoder` error | Categorical column has unseen values in test set | Ensure label encoding is fit on the full dataset before splitting |

### Checking Azure Resources

```bash
# List compute clusters
az ml compute list --resource-group myResourceGroup --workspace-name myMLWorkspace

# List registered datasets
az ml data list --resource-group myResourceGroup --workspace-name myMLWorkspace

# List registered models
az ml model list --resource-group myResourceGroup --workspace-name myMLWorkspace

# View pipeline job details
az ml job show --name <job-name> --resource-group myResourceGroup --workspace-name myMLWorkspace
```

### Re-running a Failed Workflow

1. Go to the **Actions** tab in GitHub
2. Click on the failed workflow run
3. Click **Re-run all jobs** or re-run only the failed job
4. Alternatively, push a new commit to trigger a fresh run
