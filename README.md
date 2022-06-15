# Setup

## Create the app

```bash
# Create repository in Bayrouth organization
gh repo create --internal bayrouth/the-microservice

# Clone repo
gh repo clone bayrouth/the-microservice

# Create a basic README file
echo "# The Microservice" > README.md
git add .
git commit -S -m "Add README.md"
git push -u origin main

# Go to the repo
gh repo view --web

# BROWSER: Create a new codespace

# Connect to codespace from local VSCode
gh cs list
gh cs code -c <name>

# Create an index.js file and write a simple web server
#!/usr/bin/env node
/**
 * Webserver for the webapp listening to port
 * 3005
 */
const http = require('http');
const port = process.env.port || 3005;

// Create an http server instance
const server = http.createServer((req, res) => {
  // Get the request ip address
  const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
  // Get the request url
  const url = req.url;
  // Get the request method
  const method = req.method;
  switch(url) {
    case '/':
      res.writeHead(200, {'Content-Type': 'text/html'});
      res.end('<h1>Hello World</h1>');
      break;
    case '/api/data':
      res.writeHead(200, {'Content-Type': 'application/json'});
      res.end(JSON.stringify([1,2,3,4,5]));
      break;
    default:
      res.writeHead(404, {'Content-Type': 'text/html'});
      res.end('<h1>404 Not Found</h1>');
      break;
  }
});

// Start the server 
server.listen(port, () => {
  console.log(`Server is listening on port ${port}`);
});

# Push the changes
git add .
git commit -S -m "Add web server implementation"
git push -u origin main
```

## Create the VM and deploy

```bash
# Install azure-cli
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Authenticate azure-cli
az login

# Create a new resource group
az group create --name ModernDevDemoRG --location westeurope

# Create a new service principal for our app
az role assignment create \
  --role contributor \
  --subscription aa9ba7ff-8cbd-4fff-a0a9-bd9487062766 \
  --assignee-object-id 4bac9cd2-d03d-4b13-9728-e8e5bd0bba85 \
  --scope /subscriptions/aa9ba7ff-8cbd-4fff-a0a9-bd9487062766/resourceGroups/ModernDevDemoRG/providers/Microsoft.Web/sites/ \
  --assignee-principal-type ServicePrincipal

az role assignment create --role contributor --subscription $subscriptionId --assignee-object-id  $assigneeObjectId --scopes /subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.Web/sites/--assignee-principal-type ServicePrincipal

# Configure OIDC secrets on the repo
# 70c59516-9e96-43f5-81c3-ab9bd78176e2
gh secret set AZURE_TENANT_ID \
  -R bayrouth/the-microservice \
  --body (az account list --query "[0].tenantId" --output tsv)

# 4bac9cd2-d03d-4b13-9728-e8e5bd0bba85
gh secret set AZURE_CLIENT_ID \
  -R bayrouth/the-microservice \
  --body (az ad app list --query "[?displayName=='github-actions'].appId | [0]" --output tsv)

# aa9ba7ff-8cbd-4fff-a0a9-bd9487062766
gh secret set AZURE_SUBSCRIPTION_ID \
  -R bayrouth/the-microservice \
  --body (az account list --query "[0].id" --output tsv)

# Create the app service plan
az appservice plan create \
  --name the-microservice-plan-2022 \
  --resource-group ModernDevDemoRG \
  --location westeurope \
  --sku FREE \
  --is-linux

# List available runtimes
az webapp list-runtimes --os-type linux

# Create a web app
az webapp create \
  --name the-microservice \
  --resource-group ModernDevDemoRG \
  --plan the-microservice-plan-2022 \
  --runtime NODE:16-lts

# Assign contributor role to the app
# https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#assign-a-role-to-the-application
```

## Create deployment workflow

```yaml
name: Deploy service

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

env:
  AZURE_WEBAPP_NAME: the-microservice # set this to your application's name
  AZURE_WEBAPP_PACKAGE_PATH: '.'      # set this to your application's package path
  NODE_VERSION: '16.x'                # set this to the node version to use

jobs:
  deploy:
    name: Deploy to production
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v2
        name: 'Checkout repository'
      
      - name: Run service
        run: |
          RUNNER_TRACKING_ID="" && node index.js >out.log 2>err.log &

      - name: Test service
        run: |
          # Test 200 response
          output=$(curl -G http://localhost:3005/)
          if [ "$output" != "<h1>Hello World</h1>" ]; then
            echo "Service is not working"
            exit 1
          else
            echo "Test 200: OK"
          fi

          # Test 404 response
          output=$(curl -G http://127.0.0.1:3005/wrong-path)
          if [ "$output" != "<h1>404 Not Found</h1>" ]; then
            echo "Service is not working"
            exit 1
          else
            echo "Test 404: OK"
          fi

      - name: 'Az CLI login'
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}

      - name: Setup tmate session
        if: ${{ failure() }}
        uses: mxschmitt/action-tmate@v3
```

```bash
gh run watch
```

## Nuke the setup

```bash
# Destroy the setup
az group delete --name ModernDevDemoRG
```

## References

- <https://docs.microsoft.com/en-us/azure/app-service/deploy-github-actions?tabs=openid>
- <https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure>
- <https://docs.microsoft.com/en-us/azure/app-service/configure-language-nodejs?pivots=platform-linux>
