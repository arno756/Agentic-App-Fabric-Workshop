# Deploying SQL Agentic App to Azure Web App

**Step-by-Step Deployment Guide**

---

## Overview

This guide walks you through deploying the SQL Agentic Banking App to Azure Web App with Managed Identity authentication to Microsoft Fabric SQL. The deployment combines both the banking service and analytics service into a single Azure Web App.

> **Architecture**
> - React Frontend → Azure Web App (Flask) → Microsoft Fabric SQL Database
> - Authentication: System Assigned Managed Identity (no passwords stored)
> - AI Integration: Azure OpenAI Service

---

## Prerequisites

Before starting, ensure you have the following:

- Azure subscription with permissions to create resources
- Azure CLI installed and authenticated (`az login`)
- GitHub account with the repository forked/cloned
- Microsoft Fabric workspace with SQL Database provisioned
- Azure OpenAI Service with deployed models

---

## Step 1: Create Azure Resources

### 1.1 Create Resource Group

Create a resource group to contain all deployment resources.

```bash
az group create --name rg-sql-agentic-fabric --location eastus
```

### 1.2 Create App Service Plan

Create a Linux App Service Plan. Use B1 tier or higher for production workloads.

```bash
az appservice plan create \
  --name asp-sql-agentic-fabric \
  --resource-group rg-sql-agentic-fabric \
  --is-linux \
  --sku B1
```

### 1.3 Create Web App

Create the Web App with Python 3.11 runtime.

```bash
az webapp create \
  --name web-sql-agentic-fabric \
  --resource-group rg-sql-agentic-fabric \
  --plan asp-sql-agentic-fabric \
  --runtime "PYTHON:3.11"
```

> 💡 **Tip:** The Web App name must be globally unique. If `web-sql-agentic-fabric` is taken, choose a unique suffix like `web-sql-agentic-fabric-yourname`.

---

## Step 2: Enable System Assigned Managed Identity

Managed Identity allows your Web App to authenticate to Fabric SQL without storing credentials. This is more secure than using connection string passwords.

### 2.1 Enable via Azure CLI

```bash
az webapp identity assign \
  --name web-sql-agentic-fabric \
  --resource-group rg-sql-agentic-fabric
```

This command returns a JSON object containing the `principalId`. Save this value — you'll need it for granting database access.

### 2.2 Enable via Azure Portal (Alternative)

1. Navigate to your Web App in Azure Portal
2. Select **Identity** from the left menu
3. Under **System assigned** tab, toggle Status to **On**
4. Click **Save** and confirm
5. Note the **Object (principal) ID** displayed after creation

> **Important:**
> - The Managed Identity name is typically the same as your Web App name (e.g., `web-sql-agentic-fabric`).
> - For Fabric SQL access, you'll use the Object ID from the identity, not the display name.

---

## Step 3: Configure Application Settings

Configure environment variables in App Service. These replace the `.env` file used in local development.

### 3.1 Required Environment Variables

| Variable Name | Description |
|---------------|-------------|
| `FABRIC_SQL_CONNECTION_URL_AGENTIC` | Fabric SQL ODBC connection string with ActiveDirectoryMsi authentication |
| `AZURE_OPENAI_KEY` | Azure OpenAI API key |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI endpoint URL |
| `AZURE_OPENAI_DEPLOYMENT` | Chat model deployment name (e.g., gpt-4o-mini) |
| `AZURE_OPENAI_EMBEDDING_DEPLOYMENT` | Embedding model deployment name (text-embedding-ada-002) |

### 3.2 Configure via Azure CLI

```bash
az webapp config appsettings set \
  --name web-sql-agentic-fabric \
  --resource-group rg-sql-agentic-fabric \
  --settings \
    FABRIC_SQL_CONNECTION_URL_AGENTIC="Driver={ODBC Driver 18 for SQL Server};Server=your-server.datawarehouse.fabric.microsoft.com,1433;Database=agentic_app_db;Encrypt=yes;TrustServerCertificate=no;Authentication=ActiveDirectoryMsi" \
    AZURE_OPENAI_KEY="your-openai-key" \
    AZURE_OPENAI_ENDPOINT="https://your-openai.openai.azure.com/" \
    AZURE_OPENAI_DEPLOYMENT="gpt-4o-mini" \
    AZURE_OPENAI_EMBEDDING_DEPLOYMENT="text-embedding-ada-002"
```

> ⚠️ **Security Note:** Never commit API keys to source control. Use Azure Key Vault references for production deployments.

---

## Step 4: Grant Managed Identity Access to Fabric SQL

The Web App's Managed Identity must be granted access to your Fabric SQL Database. This is done by creating a database user linked to the identity.

### 4.1 Connect to Fabric SQL Endpoint

Use SQL Server Management Studio (SSMS), Azure Data Studio, or the Fabric SQL Query Editor to connect to your database.

### 4.2 Create User and Grant Permissions

Execute the following T-SQL commands in your Fabric SQL database:

```sql
-- Create a user for the Web App's Managed Identity
CREATE USER [web-sql-agentic-fabric] FROM EXTERNAL PROVIDER;

-- Grant read and write permissions
ALTER ROLE db_datareader ADD MEMBER [web-sql-agentic-fabric];
ALTER ROLE db_datawriter ADD MEMBER [web-sql-agentic-fabric];
```

> **Note:**
> - Replace `[web-sql-agentic-fabric]` with your actual Web App name.
> - The name must match exactly (case-sensitive in some configurations).
> - `FROM EXTERNAL PROVIDER` tells SQL to look up the identity in Entra ID (Azure AD).

---

## Step 5: Configure Startup Command

The repository includes a special Azure launcher (`launcher_azure.py`) that combines both the banking and analytics services into a single WSGI application.

### 5.1 Set the Startup Command

```bash
az webapp config set \
  --name web-sql-agentic-fabric \
  --resource-group rg-sql-agentic-fabric \
  --startup-file "gunicorn --bind=0.0.0.0:8000 --timeout 600 launcher_azure:run_combined_services"
```

### 5.2 What launcher_azure.py Does

- Initializes both `banking_app` and `agent_analytics` Flask applications
- Creates database tables and runs data ingestion
- Combines both apps using `DispatcherMiddleware`
- Routes: `/` → Banking App, `/analytics` → Analytics Service

---

## Step 6: Set Up GitHub Actions CI/CD

Configure GitHub Actions to automatically build and deploy your application when you push to the main branch.

### 6.1 Create Service Principal for GitHub

Create a service principal that GitHub Actions will use to deploy to Azure.

```bash
az ad sp create-for-rbac \
  --name "github-sql-agentic-deploy" \
  --role contributor \
  --scopes /subscriptions/{subscription-id}/resourceGroups/rg-sql-agentic-fabric \
  --sdk-auth
```

This outputs a JSON object. Copy the entire JSON — you'll add it as a GitHub secret.

### 6.2 Get Web App Publish Profile

Download the publish profile for deployment:

```bash
az webapp deployment list-publishing-profiles \
  --name web-sql-agentic-fabric \
  --resource-group rg-sql-agentic-fabric \
  --xml
```

Copy the entire XML output.

### 6.3 Add GitHub Repository Secrets

1. Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add `AZURE_CREDENTIALS` with the service principal JSON
4. Add `AZUREAPPSERVICE_PUBLISHPROFILE` with the publish profile XML
5. Add `AZURE_WEBAPP_NAME` with value: `web-sql-agentic-fabric`

### 6.4 GitHub Actions Workflow

The repository includes a workflow file at `.github/workflows/deploy-webapp.yml` that:

- Builds the React frontend with `npm run build`
- Copies the built frontend into `backend/static/`
- Packages the backend with all dependencies
- Deploys to Azure Web App using the publish profile

---

## Step 7: Deploy and Verify

### 7.1 Trigger Deployment

Push to your main branch to trigger the GitHub Actions workflow:

```bash
git add .
git commit -m "Configure Azure deployment"
git push origin main
```

### 7.2 Monitor Deployment

1. Go to GitHub → **Actions** tab to monitor the workflow
2. Check Azure Portal → Web App → **Deployment Center** for deployment status
3. View logs: Azure Portal → Web App → **Log stream**

### 7.3 Access Your Application

Once deployed, access your application at:

```
https://web-sql-agentic-fabric.azurewebsites.net
```

### 7.4 Verify All Services

- **Main app:** `https://your-app.azurewebsites.net/`
- **Analytics API:** `https://your-app.azurewebsites.net/analytics/api/chat/sessions`
- Test the chatbot to verify Azure OpenAI connection
- Create a demo account to verify Fabric SQL connection

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| 500 Internal Server Error | Check App Service logs. Often caused by missing environment variables or database connection issues. |
| Database connection failed | Verify Managed Identity is enabled and the user was created in Fabric SQL with correct permissions. |
| Module not found errors | Ensure requirements.txt includes all dependencies and deployment completed successfully. |
| OpenAI API errors | Verify AZURE_OPENAI_* environment variables are correct and the deployment exists. |
| Static files not loading | Confirm the GitHub Action copied frontend build to backend/static/ before deployment. |

### Viewing Logs

```bash
# Stream live logs
az webapp log tail \
  --name web-sql-agentic-fabric \
  --resource-group rg-sql-agentic-fabric

# Download logs
az webapp log download \
  --name web-sql-agentic-fabric \
  --resource-group rg-sql-agentic-fabric
```

---

## Summary

| Component | Purpose | Configuration |
|-----------|---------|---------------|
| **Service Principal** | GitHub Actions deployment | `az ad sp create-for-rbac` |
| **Managed Identity** | Runtime DB authentication | Web App → Identity → System assigned |
| **App Settings** | Environment variables | Web App → Configuration |
| **Startup Command** | Gunicorn with launcher_azure | Web App → Configuration → General |
| **Database User** | Fabric SQL access | `CREATE USER FROM EXTERNAL PROVIDER` |

> 💡 **Next Steps:** Consider enabling Azure Application Insights for monitoring, setting up custom domains with SSL certificates, and configuring auto-scaling for production workloads.
