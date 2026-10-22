# Deploy ShoppingBench to Azure with azd

This guide deploys the Azure tool API and then a hosted agent in your own Microsoft Foundry project. The two parts are separate `azd` projects:

- Repository root: Azure AI Search, the tool API on Container Apps, and supporting resources.
- `agents/`: hosted agents connected to an existing Foundry project and model deployment.

For the simplest first deployment, start with the Product agent. Optimization and RFT are optional and are not required to deploy.

## Before you start

Have these ready before provisioning anything:

- An Azure subscription and region where you can create resources and accept the associated charges. Search and Container Registry continue to incur charges while provisioned. The `demo` profile is smaller than the default `benchmark` profile.
- Permission to create resources in the target resource group, and permission to create role assignments. The Bicep template assigns the Container App identity `AcrPull` on the registry and `Search Index Data Reader` on Search. In practice, this usually means `Contributor` plus `User Access Administrator` (or `Owner`) at the deployment scope.
- For uploading the catalog, your signed-in user also needs `Search Service Contributor` and `Search Index Data Contributor` on the created Search service.
- An existing Microsoft Foundry project that supports hosted agents, its project endpoint, and an available MAI-compatible model deployment with sufficient quota. The repository does not create the Foundry project or model deployment.
- Azure CLI, Azure Developer CLI (`azd`) version 1.27.1 or later, and Python 3.11 or later. To deploy hosted agents, the `azure.ai.agents` azd extension version 1.0.0-beta.17 or later must also be available.
- A generated tool API token of at least 32 characters. The commands below generate one without printing it.

Docker is not required for the API deployment; the root `azure.yaml` uses remote image builds.

### First-deployment note

For the first API deployment, **do not use a single `azd up`**. This repository deliberately separates provisioning from deployment so Azure has time to propagate the new registry `AcrPull` assignment before the app image is deployed. Use `azd provision`, verify the assignment, then `azd deploy`. Once the stack exists and that role assignment has propagated, `azd up` is suitable for later root-project updates.

## 1. Check tools and sign in

Run these from the repository root in a Bash terminal:

```bash
az version
azd version
python --version
azd ai agent --help
```

Confirm the versions listed above. The last command should show the hosted-agent commands; if it is unavailable, install or update the `azure.ai.agents` azd extension before continuing.

Sign in and select the intended subscription:

```bash
az login
azd auth login
export AZURE_SUBSCRIPTION_ID="<your-subscription-id>"
az account set --subscription "$AZURE_SUBSCRIPTION_ID"
az account show --query '{name:name,id:id,tenantId:tenantId}' --output table
```

Check that the displayed subscription is the one you intend to use.

## 2. Install the Python package

From the repository root:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -e ".[agents]"
```

The prepared catalog file is already included at `data/prepared/search-documents.jsonl`.

## 3. Configure and provision the API resources

Create a dedicated azd environment. `azd env new` may prompt you to confirm the subscription and location.

```bash
azd env new shoppingbench-demo
azd env set AZURE_SUBSCRIPTION_ID "$AZURE_SUBSCRIPTION_ID"
azd env set AZURE_LOCATION eastus2
azd env set AZURE_SEARCH_LOCATION eastus2
azd env set SHOPPINGBENCH_DEPLOYMENT_PROFILE demo
export SHOPPINGBENCH_API_TOKEN="$(openssl rand -hex 32)"
azd env set SHOPPINGBENCH_API_TOKEN "$SHOPPINGBENCH_API_TOKEN"
```

Change the region values if needed before the first provisioning attempt. `AZURE_LOCATION` controls the supporting resources, while `AZURE_SEARCH_LOCATION` independently controls Search. The `demo` profile uses Basic Search and one small Container App replica. The default `benchmark` profile uses Standard Search and more Container App capacity.

Now provision the resources:

```bash
azd provision
```

This creates the resource group resources, including Azure AI Search, Container Apps, Container Registry, Storage, managed identity, and monitoring. It does not deploy the API image yet.

If Search fails with `InsufficientResourcesAvailable` in `eastus2`, keep the
same azd environment and change only the Search region, then retry:

```bash
azd env set AZURE_SEARCH_LOCATION westus2
azd provision
```

`westus2` is an example; capacity is not guaranteed. Choose a region that
supports your Search tier and meets your data residency requirements. Do not
change `AZURE_LOCATION` to recover this Search-only failure: it also changes
the other resources' locations and all generated resource names. This applies
to a new Search service that failed to provision, not migrating an existing
service or its indexes.

Before proceeding, verify in Azure Portal that the new Container App managed identity has the `AcrPull` role on the new registry. Allow time for the role assignment to appear. The identity and assignment are created by the template.

Then deploy the API:

```bash
azd deploy
```

## 4. Upload the catalog and check the API

Your signed-in Azure identity needs the Search indexing roles listed under [Before you start](#before-you-start). Capture the generated endpoint values without printing the API token:

```bash
export AZURE_SEARCH_ENDPOINT="$(azd env get-value AZURE_SEARCH_ENDPOINT)"
export SHOPPINGBENCH_TOOL_URL="$(azd env get-value SHOPPINGBENCH_TOOL_URL)"
python -m shoppingbench_foundry.index_documents \
  --endpoint "$AZURE_SEARCH_ENDPOINT" \
  --documents data/prepared/search-documents.jsonl
curl --fail "$SHOPPINGBENCH_TOOL_URL/health"
```

The health check should return successfully before you configure an agent.

## 5. Deploy the Product hosted agent

Keep the same terminal open so the generated API token remains available in `SHOPPINGBENCH_API_TOKEN`. Set the values for your existing Foundry project and model deployment:

```bash
export FOUNDRY_PROJECT_ENDPOINT="https://<foundry-resource>.services.ai.azure.com/api/projects/<project>"
export PRODUCT_OPENAI_BASE_URL="<model-endpoint-ending-in-/openai/v1>"
export PRODUCT_MODEL_DEPLOYMENT_NAME="<your-model-deployment>"
```

The model endpoint and deployment name are from your Foundry/Azure model deployment; they are not necessarily the same as the project endpoint. Ensure the hosted agent can authenticate to the project and model resource, and that the model supports the Responses API used by this runtime.

Set up a separate azd environment for the hosted agents:

```bash
cd agents
azd env new shoppingbench-agents
azd env set AZURE_SUBSCRIPTION_ID "$AZURE_SUBSCRIPTION_ID"
azd env set FOUNDRY_PROJECT_ENDPOINT "$FOUNDRY_PROJECT_ENDPOINT"
azd env set PRODUCT_OPENAI_BASE_URL "$PRODUCT_OPENAI_BASE_URL"
azd env set PRODUCT_MODEL_DEPLOYMENT_NAME "$PRODUCT_MODEL_DEPLOYMENT_NAME"
azd env set SHOPPINGBENCH_TOOL_URL "$SHOPPINGBENCH_TOOL_URL"
azd env set SHOPPINGBENCH_API_TOKEN "$SHOPPINGBENCH_API_TOKEN"
azd deploy shoppingbench-product-agent
```

`azd deploy` targets only the Product service here; it does not deploy the other four agents. In Foundry, wait for the new agent version to become active, then run a small test request and confirm it calls the tool API and returns a product recommendation.

## 6. Deploy the other agents (optional)

Before deploying all hosted agents, set the corresponding endpoint and deployment-name pairs for `SHOP`, `VOUCHER`, `WEB`, and `WEB_LIVE` in the `agents` azd environment. The manifest uses these exact variable names:

- `SHOP_OPENAI_BASE_URL` and `SHOP_MODEL_DEPLOYMENT_NAME`
- `VOUCHER_OPENAI_BASE_URL` and `VOUCHER_MODEL_DEPLOYMENT_NAME`
- `WEB_OPENAI_BASE_URL` and `WEB_MODEL_DEPLOYMENT_NAME`
- `WEB_LIVE_OPENAI_BASE_URL` and `WEB_LIVE_MODEL_DEPLOYMENT_NAME`

If several agents use the same model endpoint and deployment, you may set the same values for each pair. Then deploy the remaining services from the `agents/` directory:

```bash
azd deploy
```

Web (live) also needs separate web-search model settings: `WEB_SEARCH_OPENAI_BASE_URL` and `WEB_SEARCH_MODEL_DEPLOYMENT_NAME`. Its API-side grader settings are `RFT_GRADER_MODEL_DEPLOYMENT_NAME` and the web-search endpoint/deployment values. Configure these in the root azd environment before provisioning if you plan to use Web (live), and in the `agents` environment for the hosted agent. The Container App identity must be granted `Cognitive Services OpenAI User` on the judge model resource. Web search, model inference, and RFT can incur additional charges; RFT is optional.

## After the first deployment

Once the API stack is provisioned and the registry role assignment has propagated, from the repository root you can use:

```bash
azd up
```

This runs the root project's provision-and-deploy workflow. Hosted agents remain a separate project under `agents/`; deploy them there with `azd deploy`.

Keep the local `.azure` environment files private and out of source control. They contain environment configuration, including the API token. Do not paste the token into logs or commit it.
