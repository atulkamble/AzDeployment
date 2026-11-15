# Azure Deployment Basics

A beginner-friendly guide to deploying an application to Microsoft Azure. This repository includes Azure CLI examples, a simple deployment workflow, validation steps, and rollback guidance.

## What you will learn

- Deploy an ASP.NET Core application to Azure App Service
- Create a Linux App Service plan and web app
- Deploy a ZIP package to a staging slot
- Check the application health endpoint
- Promote a tested release to production and roll back if needed
- Compare common deployment strategies

## Repository guides

- [Azure application deployment guide](azure-application-deployment-guide.md) — complete example including App Service, Azure SQL, configuration, health checks, VM deployment, validation, and rollback.
- [Deployment strategies guide](deployment-strategies-guide.md) — blue-green, canary, rolling, feature flags, traffic routing, zero-downtime practices, and App Service slot swap.

## Prerequisites

- An Azure subscription
- Azure CLI installed
- .NET 8 SDK installed
- Bash shell (macOS, Linux, or WSL)
- Permission to create resource groups and App Service resources

Sign in and choose the subscription:

~~~bash
az login
az account set --subscription "<SUBSCRIPTION_ID>"
~~~

## Basic App Service deployment

This example creates an App Service, publishes a sample .NET app, deploys it to a staging slot, checks it, and swaps it to production.

### 1. Set resource names

App Service names must be globally unique. Replace the sample values as needed.

~~~bash
export LOCATION="eastus"
export RG="rg-azdeployment-demo"
export PLAN="asp-azdeployment-demo"
export APP="azdeploy-<unique-suffix>"
export SLOT="staging"
~~~

### 2. Create Azure resources

The B1 plan supports deployment slots. Costs can apply while resources are running.

~~~bash
az group create --name "$RG" --location "$LOCATION"

az appservice plan create \
  --name "$PLAN" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --is-linux \
  --sku B1

az webapp create \
  --name "$APP" \
  --resource-group "$RG" \
  --plan "$PLAN" \
  --runtime "DOTNETCORE:8.0"

az webapp deployment slot create \
  --name "$APP" \
  --resource-group "$RG" \
  --slot "$SLOT"
~~~

### 3. Create and publish a sample app

If you already have an ASP.NET Core 8 app with a `/health` endpoint, use it instead.

~~~bash
dotnet new web -n DemoWebApp
cd DemoWebApp
~~~

Replace `Program.cs` with:

~~~csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHealthChecks();

var app = builder.Build();
app.MapGet("/", () => Results.Ok("Hello from Azure App Service"));
app.MapHealthChecks("/health");
app.Run();
~~~

Publish a ZIP package:

~~~bash
dotnet publish --configuration Release --output ./publish
(cd publish && zip -qr ../app.zip .)
~~~

### 4. Deploy to staging

~~~bash
az webapp deploy \
  --resource-group "$RG" \
  --name "$APP" \
  --slot "$SLOT" \
  --src-path ./app.zip \
  --type zip

STAGE_HOST=$(az webapp show \
  --resource-group "$RG" \
  --name "$APP" \
  --slot "$SLOT" \
  --query defaultHostName \
  --output tsv)

curl -fsS "https://${STAGE_HOST}/"
curl -fsS "https://${STAGE_HOST}/health"
~~~

Both requests should succeed before promotion. If a check fails, inspect the deployment logs and fix the issue before swapping.

### 5. Promote the release

~~~bash
az webapp deployment slot swap \
  --resource-group "$RG" \
  --name "$APP" \
  --slot "$SLOT" \
  --target-slot production
~~~

Verify production:

~~~bash
PROD_HOST=$(az webapp show \
  --resource-group "$RG" \
  --name "$APP" \
  --query defaultHostName \
  --output tsv)

curl -fsS "https://${PROD_HOST}/"
curl -fsS "https://${PROD_HOST}/health"
~~~

### 6. Roll back if required

If the new version has a problem immediately after the swap, swap the slots back:

~~~bash
az webapp deployment slot swap \
  --resource-group "$RG" \
  --name "$APP" \
  --slot "$SLOT" \
  --target-slot production
~~~

A code rollback does not undo database changes. Keep database migrations backward-compatible and plan data recovery separately.

### 7. Clean up the demo resources

Delete the resource group when you no longer need this lab. This permanently deletes the resources inside it.

~~~bash
az group delete --name "$RG" --yes --no-wait
~~~

## Deployment flow

1. Build and package the application.
2. Deploy to staging.
3. Run health checks and smoke tests.
4. Review logs and configuration.
5. Swap staging into production.
6. Monitor production and roll back if needed.

## Security notes

- Never commit passwords, connection strings, or other secrets.
- Prefer managed identity and Azure Key Vault for application secrets.
- Restrict database firewall access to only the required sources.
- Use separate test and production databases.
- Review plan pricing and regional availability before creating resources.

## License

See [LICENSE](LICENSE).
