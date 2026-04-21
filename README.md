Columbus State Bank — ADF Deployment Guide

This repository contains everything needed to deploy the Austin Sales Tax data pipeline in a new Azure environment.

---

Prerequisites

Make sure you have the following installed and set up before starting:

Azure CLI — to deploy the ARM template
Terraform — to provision Azure infrastructure
An active Azure subscription
A GitHub account with access to this repository

--------------------------------------------------------------------------------------------------------------------------------------------

## Step 1 — Login to Azure

az login


Set your subscription if you have multiple:

az account set --subscription "<your-subscription-id>"

--------------------------------------------------------------------------------------------------------------------------------------------


## Step 2 — Provision Infrastructure with Terraform

This creates the Resource Group, Storage Account, SQL Database, and ADF instance in your Azure environment.

terraform init
terraform apply -var="sql_admin_password=<your_secure_password>"


Note: If you get a naming conflict error, change the `project_name` variable in `main.tf` to something unique (e.g. `mycompanyadf2025`) and retry. Storage account names must be globally unique across all of Azure.

After `terraform apply` completes, save the outputs — you will need them in the next step:
sql_connection_string
storage_account_name
adf_name
resource_group_name

--------------------------------------------------------------------------------------------------------------------------------------------


## Step 3 — Deploy ADF Pipelines via ARM Template

Switch to the `adf_publish` branch and locate the ARM template files:
```bash
git checkout adf_publish
```

You will see:
```
ARMTemplateForFactory.json
ARMTemplateParametersForFactory.json
```

Open `ARMTemplateParametersForFactory.json` and fill in your values from the Terraform outputs:

```json
{
  "factoryName": { "value": "<your-adf-name>" },
  "AzureSqlDatabase_connectionString": { "value": "<your-sql-connection-string>" },
  "AzureBlobStorage_connectionString": { "value": "<your-storage-connection-string>" }
}
```

> ⚠️ Never commit this file back to the repo after filling in your credentials.

Then deploy:
```bash
az deployment group create \
  --resource-group <your-resource-group-name> \
  --template-file ARMTemplateForFactory.json \
  --parameters ARMTemplateParametersForFactory.json
```

--------------------------------------------------------------------------------------------------------------------------------------------


## Step 4 — Connect ADF to This Repo

1. Go to your ADF Studio → **Manage → Source Control → Git Configuration → Configure**
2. Fill in:
   - Repository type: `GitHub`
   - GitHub account: your username
   - Repository name: this repo
   - Collaboration branch: `main`
   - Publish branch: `adf_publish`
   - Root folder: `/`
3. Click **Apply**

--------------------------------------------------------------------------------------------------------------------------------------------


## Step 5 — Activate the Trigger

1. In ADF Studio → **Manage → Triggers**
2. Find `TriggerSalesTax_Monday`
3. Click the ▶️ toggle to **Start** it
4. Click **Publish All**

The pipeline will now run every **Monday at 10:00 AM Central Time**.

--------------------------------------------------------------------------------------------------------------------------------------------


## Step 6 — Verify

1. ADF Studio → **Monitor → Trigger Runs**
2. Confirm the trigger shows as **Started**
3. On the next Monday at 10 AM, check **Monitor → Pipeline Runs** to confirm successful execution

--------------------------------------------------------------------------------------------------------------------------------------------


## Repository Structure

```
Columbus-State-Bank_ADF/
├── main.tf                  ← Terraform — provisions Azure infrastructure
├── README.md                ← This file
├── pipeline/                ← ADF pipeline definitions
│   ├── Main_Pipeline.json
│   ├── p1_austin_salestax.json
│   └── p2_austin_tabc.json
├── trigger/                 ← ADF trigger definitions
│   └── TriggerSalesTax_Monday.json
├── linkedService/           ← ADF linked service connections
└── dataset/                 ← ADF dataset definitions
```

--------------------------------------------------------------------------------------------------------------------------------------------


## Pipeline Overview

| Pipeline | Description |
|---|---|
| `Main_Pipeline` | Parent pipeline that orchestrates both child pipelines |
| `p1_austin_salestax` | Fetches Texas Gov sales tax permit data, cleans it, loads to SQL |
| `p2_austin_tabc` | Fetches TABC data, cleans it, loads to SQL |

Data flows through a **staging Storage Account** before being loaded into the SQL Database.

--------------------------------------------------------------------------------------------------------------------------------------------


## Support

For any issues with deployment, contact the project team.
