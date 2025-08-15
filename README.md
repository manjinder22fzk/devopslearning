az deployment group create \
  --resource-group rg-adf-learning-dev \
  --template-file adf_publish/ARMTemplateForFactory.json \
  --parameters @adf_publish/ARMTemplateParametersForFactory.json



az deployment group create \
  --resource-group rg-adf-learning-uat \
  --template-file adf_publish/ARMTemplateForFactory.json \
  --parameters @arm-parameters/adf-uat.parameters.json


az deployment group create \
  --resource-group rg-adf-learning-prod \
  --template-file adf_publish/ARMTemplateForFactory.json \
  --parameters @arm-parameters/adf-prod.parameters.json



Don’t forget: grant each environment’s ADF system-assigned managed identity the Storage Blob Data Contributor role on the matching storage account.




Sweet — here’s a complete, click-by-click guide to push your ADF from Dev → UAT → Prod using Azure DevOps Classic Release pipelines (Classic editor). I’ll assume you’re using the exact ARM template + parameter files we just finished.

0) One-time prerequisites (do these first)

Create empty ADFs (one per env) – only needed once

adf-learning-dev (you already have it)

adf-learning-uat

adf-learning-prod

Reason: your template deploys child resources (pipelines/datasets/linked services). The parent factory must exist.

Create storage accounts per env (you already have Dev):

staccountlearningdev, staccountlearninguat, staccountlearningprod

Grant ADF managed identity RBAC on each env’s storage

In each storage account → IAM (Access control) → Add role assignment

Role: Storage Blob Data Contributor

Assign to the matching ADF’s System-assigned managed identity (e.g., adf-learning-uat)

Repeat for prod.

This is required because your ls_adls uses Managed Identity.

Azure DevOps service connection (one-time)

Azure DevOps → Project settings → Service connections → New service connection → Azure Resource Manager

Scope: Subscription

Name it e.g. sc-azure-subscription

Grant access permission to all pipelines (checkbox).

Repo has the files

adf_publish/ARMTemplateForFactory.json (from my last message)

adf_publish/ARMTemplateParametersForFactory.json (DEV values)

arm-parameters/adf-uat.parameters.json

arm-parameters/adf-prod.parameters.json

1) Create a Classic Release pipeline

Azure DevOps → Pipelines → Releases → New pipeline (Classic).

Artifact (bottom left) → Add

Source type: Azure Repos Git

Project/Repo: your repo that contains adf_publish

Default branch: adf_publish

Artifact alias: set to arm (easy to reference)

Add

Enable Continuous deployment trigger (lightning icon on the artifact) if you want a release when adf_publish updates (i.e., after each ADF Publish).

2) Add three stages (Dev, UAT, Prod)

Click + Add a new stage three times (or clone the first one after you configure it):

Stage 1: Deploy to Dev

Stage 2: Deploy to UAT

Stage 3: Deploy to Prod

(Optional but recommended) Set Pre-deployment approvals on UAT/Prod stages.

3) Configure the task in each stage

Each stage will have one agent job with one task: Azure Resource Group Deployment (ARM).

Add the task

Inside the stage → Tasks → Agent job → + → search Resource Group

Add Azure Resource Group Deployment (a.k.a. ARM template deployment).

Configure common fields

Azure subscription: sc-azure-subscription (the service connection you created)

Action: Create or update resource group

Resource group:

Dev: rg-adf-learning-dev

UAT: rg-adf-learning-uat

Prod: rg-adf-learning-prod

Location: your region (same as ADF)

Template location: Linked artifact

Template:

$(System.DefaultWorkingDirectory)/arm/adf_publish/ARMTemplateForFactory.json


Template parameters: (pick the right file per stage, see below)

Why that path? In Classic releases, the artifact is downloaded to
$(System.DefaultWorkingDirectory)/<artifact-alias>/....
We set alias = arm, so your files are under /arm/....

Dev stage – Parameters file
$(System.DefaultWorkingDirectory)/arm/adf_publish/ARMTemplateParametersForFactory.json

UAT stage – Parameters file
$(System.DefaultWorkingDirectory)/arm/arm-parameters/adf-uat.parameters.json

Prod stage – Parameters file
$(System.DefaultWorkingDirectory)/arm/arm-parameters/adf-prod.parameters.json


Leave “Override template parameters” empty (we already supply everything via files).

Hit Save on the release definition.

4) Wire up stage order + approvals

On the pipeline designer, drag connectors so it’s Dev → UAT → Prod.

Click the thunder/approval icon on UAT and Prod to enable Pre-deployment approvals (e.g., you approve in UAT, a lead approves in Prod).

5) Run it

Two options:

Manual: Click Create release, select the pipeline, pick the artifact version (the latest adf_publish), and deploy.

Automatic: If you enabled the continuous deployment trigger, publishing from ADF (which commits to adf_publish) will automatically queue a new release.

Watch logs for each stage. The ARM task should report creating/updating:

linkedServices/ls_adls, linkedServices/ls_github

datasets/ds_adls_sink, datasets/ds_github_source

pipelines/pl_copy_github_to_adls

6) Post-deploy checks (per environment)

ADF Studio → open the UAT (then Prod) factory.

Confirm the pipeline & datasets exist.

Open the pipeline parameters panel: values should reflect your UAT/Prod parameter files (baseUrl, relativeUrl, fileName, etc.).

Test run in each env:

Trigger the pipeline with default params (they’re coming from ARM).

Confirm the file lands in the right storage (staccountlearninguat / staccountlearningprod, landing container).

7) Typical “gotchas” (and quick fixes)

Parent factory missing error:
Create the empty ADF adf-learning-uat / adf-learning-prod first (Step 0), same region.

403 writing to ADLS:
The ADF MI doesn’t have RBAC on the storage. Re-check Storage Blob Data Contributor on the right storage account for the right ADF identity.

Template path not found:
Double-check artifact alias (arm) and branch (adf_publish).
Use the Logs → Initialize job to see the downloaded folder structure.

Wrong factoryName:
factoryName must be the ADF name (e.g., adf-learning-uat), not the resource group.

Strict environments:
Add stage-scoped variables (e.g., RG_NAME, PARAM_FILE) and reference them in the task to avoid path typos and simplify maintenance.

8) (Optional) Auto-create the UAT/Prod factory once

If you want the release to also create the empty factory the first time:

Add a small Stage 0 that deploys a tiny ARM template with just the factory resource:

{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": { "factoryName": { "type": "string" }, "location": { "type": "string" } },
  "resources": [
    {
      "type": "Microsoft.DataFactory/factories",
      "apiVersion": "2018-06-01",
      "name": "[parameters('factoryName')]",
      "location": "[parameters('location')]",
      "identity": { "type": "SystemAssigned" },
      "properties": { }
    }
  ]
}


Run this once per env, then run your main pipeline.
(Most teams just create the factory from Portal once and move on.)

You’re done 🎉

From here on:

Devs make changes in Dev ADF, Publish → updates adf_publish

Classic Release picks up the new ARM, deploys Dev → UAT → Prod with the env-specific parameter files

No JSON edits per env, just the parameters file values

If you want, I can also give you a one-stage export + three-stage deploy YAML equivalent later; but for Classic, the above is the full, production-style flow.
