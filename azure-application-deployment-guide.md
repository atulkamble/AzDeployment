# Azure Application Deployment Guide

## Scope and assumptions

This guide deploys a sample ASP.NET Core 8 web app to Azure App Service, provisions Azure SQL, configures settings and health checks, validates the release, and rolls back to the previous deployment. It also includes a VM deployment option. Replace names and credentials for your environment. The commands use Bash and Azure CLI.

### Deployment map

| Area | Implementation |
|---|---|
| Application deployment | Build and publish ASP.NET Core app |
| App Service deployment | Linux App Service, ZIP package deployment, deployment slots |
| VM deployment | Ubuntu VM with Nginx reverse proxy and systemd service |
| Azure SQL deployment | Logical server, database, firewall, app identity access |
| Configuration management | App settings, managed identity, Key Vault option |
| Connection strings | App Service settings; never commit secrets |
| Health checks | `/health` endpoint and App Service health check |
| Validation | Smoke tests, logs, database connectivity, readiness |
| Rollback | Swap slots or redeploy prior package; VM release symlink rollback |

## 1. Set variables and sign in

```bash
az login
az account set --subscription "<SUBSCRIPTION_ID>"

export LOCATION="eastus"
export RG="rg-webapp-prod"
export APP="webapp-<unique-suffix>"
export PLAN="asp-webapp-prod"
export SQL_SERVER="sql<unique-suffix>"       # lowercase letters/numbers only
export SQL_DB="appdb"
export SLOT="staging"
```

Choose globally unique names for the App Service and SQL server. Check current available SKUs and regional availability before creating resources.

## 2. Prepare the application

Example project layout:

```text
src/WebApp/WebApp.csproj
src/WebApp/Program.cs
```

Add a health endpoint in `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHealthChecks();

var app = builder.Build();
app.MapGet("/", () => Results.Ok("Application is running"));
app.MapHealthChecks("/health");
app.Run();
```

Use configuration binding for settings rather than hard-coding values. For a database connection, read `ConnectionStrings:DefaultConnection` using the .NET configuration provider:

```csharp
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
if (string.IsNullOrWhiteSpace(connectionString))
    throw new InvalidOperationException("DefaultConnection is not configured.");
```

Build and publish locally:

```bash
dotnet restore src/WebApp/WebApp.csproj
dotnet publish src/WebApp/WebApp.csproj \
  --configuration Release \
  --output ./publish

(cd publish && zip -qr ../webapp.zip .)
```

## 3. Provision App Service

Create the resource group, Linux plan, and web app:

```bash
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
```

Create a staging slot (slot support requires an appropriate App Service plan tier):

```bash
az webapp deployment slot create \
  --name "$APP" --resource-group "$RG" --slot "$SLOT"
```

Configure the health check path and required runtime settings:

```bash
az webapp config set \
  --name "$APP" --resource-group "$RG" \
  --linux-fx-version "DOTNETCORE|8.0"

az webapp config set \
  --name "$APP" --resource-group "$RG" \
  --health-check-path "/health"
```

## 4. Provision Azure SQL and authorize the app

Create a SQL logical server. For a teaching lab, SQL authentication is shown; for production, prefer Microsoft Entra authentication and managed identity.

```bash
export SQL_ADMIN="sqladmin<suffix>"
read -s -p "SQL admin password: " SQL_PASSWORD; echo

az sql server create \
  --name "$SQL_SERVER" --resource-group "$RG" --location "$LOCATION" \
  --admin-user "$SQL_ADMIN" --admin-password "$SQL_PASSWORD"

az sql db create \
  --resource-group "$RG" --server "$SQL_SERVER" \
  --name "$SQL_DB" --service-objective S0
```

For a local development connection, add your current public IP temporarily (do not open the server to all addresses):

```bash
MY_IP=$(curl -s https://api.ipify.org)
az sql server firewall-rule create \
  --resource-group "$RG" --server "$SQL_SERVER" \
  --name "AllowMyClient" --start-ip-address "$MY_IP" --end-ip-address "$MY_IP"
```

Enable a system-assigned identity and set it as the SQL server's Microsoft Entra administrator. Then create a database user for the app identity in the database. These database grants require connecting as a SQL/Entra administrator:

```bash
az webapp identity assign --name "$APP" --resource-group "$RG"
az webapp identity assign --name "$APP" --resource-group "$RG" --slot "$SLOT"
```

After setting an Entra admin on the SQL server in Azure portal or with `az sql server ad-admin create`, connect to `$SQL_DB` and execute (substitute the identity name):

```sql
CREATE USER [webapp-<unique-suffix>] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [webapp-<unique-suffix>];
ALTER ROLE db_datawriter ADD MEMBER [webapp-<unique-suffix>];
```

Grant only the permissions the application needs. Managed identity connection strings should use Microsoft Entra authentication, for example:

```text
Server=tcp:<sql-server>.database.windows.net,1433;Database=appdb;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;Authentication=Active Directory Default;
```

If using SQL authentication for a short lab, store the complete connection string in App Service settings, not source control. Keep the staging slot's database setting separate and mark it as a deployment slot setting.

## 5. Configure application settings and connection strings

Example non-secret settings:

```bash
az webapp config appsettings set \
  --name "$APP" --resource-group "$RG" \
  --settings ASPNETCORE_ENVIRONMENT=Production WEBSITE_RUN_FROM_PACKAGE=1
```

Set a connection string without putting it in shell history. Use managed identity where possible. If a password-based lab is necessary, read the password securely and set it from a protected shell/session; rotate it after the lab.

```bash
read -s -p "SQL password: " SQL_PASSWORD; echo
CONN="Server=tcp:${SQL_SERVER}.database.windows.net,1433;Initial Catalog=${SQL_DB};Persist Security Info=False;User ID=${SQL_ADMIN};Password=${SQL_PASSWORD};MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
az webapp config connection-string set \
  --name "$APP" --resource-group "$RG" \
  --connection-string-type SQLAzure \
  --settings DefaultConnection="$CONN"
unset SQL_PASSWORD CONN
```

Repeat configuration for the staging slot with its own database or test data. To use Key Vault for secrets, grant the web app identity access to the secret and reference it from App Service configuration; avoid printing secret values in logs.

## 6. Deploy to staging and validate

Deploy the ZIP package to the staging slot:

```bash
az webapp deploy \
  --resource-group "$RG" --name "$APP" --slot "$SLOT" \
  --src-path ./webapp.zip --type zip
```

Retrieve the staging hostname and check the homepage and health endpoint:

```bash
STAGE_HOST=$(az webapp show -g "$RG" -n "$APP" --slot "$SLOT" --query defaultHostName -o tsv)
curl -fsS "https://${STAGE_HOST}/"
curl -fsS "https://${STAGE_HOST}/health"
```

Inspect logs if a check fails:

```bash
az webapp log config -g "$RG" -n "$APP" --application-logging filesystem --level information
az webapp log tail -g "$RG" -n "$APP" --slot "$SLOT"
```

Validation checklist:

1. Deployment command completes successfully.
2. `/health` returns HTTP 200 and reports healthy dependencies as appropriate.
3. Main page/API responds with expected status and content.
4. Database query or application workflow succeeds from staging.
5. App Service logs show no startup, configuration, authentication, or migration errors.
6. Confirm the staging slot points to the intended test database before swap.

## 7. Promote to production

After staging checks pass, swap staging into production:

```bash
az webapp deployment slot swap \
  --resource-group "$RG" --name "$APP" \
  --slot "$SLOT" --target-slot production
```

Run the same smoke tests against production:

```bash
PROD_HOST=$(az webapp show -g "$RG" -n "$APP" --query defaultHostName -o tsv)
curl -fsS "https://${PROD_HOST}/"
curl -fsS "https://${PROD_HOST}/health"
```

## 8. Rollback for App Service

If production validation fails immediately after a slot swap, swap back:

```bash
az webapp deployment slot swap \
  --resource-group "$RG" --name "$APP" \
  --slot "$SLOT" --target-slot production
```

If no slot is available, redeploy the last known-good ZIP package:

```bash
az webapp deploy \
  --resource-group "$RG" --name "$APP" \
  --src-path ./releases/last-known-good.zip --type zip
```

Keep release packages versioned and retain at least one verified previous package. Database schema changes need a separate backward-compatible migration plan; swapping application code does not reverse SQL data or schema changes.

## 9. VM deployment option

Use this path when the application needs OS-level control. Example assumes an existing Ubuntu VM named `vm-web` and SSH access. Build a self-contained Linux publish package:

```bash
dotnet publish src/WebApp/WebApp.csproj -c Release -r linux-x64 \
  --self-contained true -o ./publish-vm
 tar -C ./publish-vm -czf webapp-vm.tgz .
scp webapp-vm.tgz azureuser@<VM_PUBLIC_IP>:/tmp/
```

On the VM, install prerequisites and create a service account:

```bash
sudo apt-get update
sudo apt-get install -y nginx
sudo useradd --system --create-home --shell /usr/sbin/nologin webapp || true
sudo mkdir -p /opt/webapp/releases
sudo tar -xzf /tmp/webapp-vm.tgz -C /opt/webapp/releases
sudo chown -R webapp:webapp /opt/webapp/releases
```

Create `/etc/systemd/system/webapp.service`:

```ini
[Unit]
Description=ASP.NET Core Web Application
After=network.target

[Service]
WorkingDirectory=/opt/webapp/current
ExecStart=/opt/webapp/current/WebApp
Restart=always
RestartSec=5
User=webapp
Environment=ASPNETCORE_URLS=http://127.0.0.1:5000
Environment=ASPNETCORE_ENVIRONMENT=Production
EnvironmentFile=-/etc/webapp/webapp.env

[Install]
WantedBy=multi-user.target
```

Create `/etc/nginx/sites-available/webapp`:

```nginx
server {
    listen 80;
    server_name _;
    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Activate the release and service:

```bash
sudo ln -sfn /opt/webapp/releases/<release-id> /opt/webapp/current
sudo ln -sfn /etc/nginx/sites-available/webapp /etc/nginx/sites-enabled/webapp
sudo nginx -t
sudo systemctl daemon-reload
sudo systemctl enable --now webapp nginx
curl -fsS http://127.0.0.1:5000/health
```

Put secrets in `/etc/webapp/webapp.env`, owned by root and readable only by the service account as needed. Do not store credentials in the unit file or deployment archive. Limit inbound network access with NSGs; expose HTTPS through a properly configured TLS endpoint.

### VM rollback

Keep each release in its own directory. Point `current` back to the prior release and restart:

```bash
sudo ln -sfn /opt/webapp/releases/<previous-release-id> /opt/webapp/current
sudo systemctl restart webapp
curl -fsS http://127.0.0.1:5000/health
```

## 10. Release and security practices

- Use staging slots and separate production/test databases.
- Prefer managed identity and Microsoft Entra authentication over embedded passwords.
- Store secrets in Key Vault or protected App Service settings; never commit them.
- Restrict SQL firewall rules and VM inbound ports to required sources.
- Keep deployments immutable and versioned; record the commit and package used.
- Apply database migrations as a controlled, backward-compatible release step.
- Configure alerts for health check failures, HTTP 5xx rates, and availability.
