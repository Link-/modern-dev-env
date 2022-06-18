# The Modern Development Environment

> TLDR; This repository is in association with the "Modern Development Environment" talk

- [Slide deck](./resources/deck.pdf)
- [Architecture drawing source](architecture.drawio)

## Architecture

![reference architecture diagram](./resources/architecture.png)

## Setup

### Create the app

_Don't forget replace all placeholders, like `:value`_

```bash
# Create repository in an organisation
gh repo create --internal :org/the-microservice

# Clone repo locally
gh repo clone :org/the-microservice

# Create a basic README file and initialise the repository
echo "# The Microservice" > README.md
git add .
git commit -S -m "Add README.md"
git push -u origin main

# Go to the repo
gh repo view --web

# BROWSER: Create a new codespace

# Connect to codespace from local VSCode
gh cs list
gh cs code -c :name

# Create an index.js file and write a simple web server
#!/usr/bin/env node
/**
 * Web server listening on environment variable PORT and defaults to 8080
 * the webserver should expose the following endpoints:
 * - GET /
 *   - Response: 200 with the following body: <h1>Hello World!</h1>
 * - GET /api/v1/data
 *   - Response: 200 with the following body: { data: [1,2,3,4] }
 * - default
 *  - Response: 404 with the following body: <h1>404</h1>
 */

const http = require('http');
const port = process.env.PORT || 8080;

const server = http.createServer((req, res) => {
  // Get the ip address of the requester
  const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
  // Log the request
  console.log(`${ip} ${req.method} ${req.url}`);
  // Respond to the request
  const url = req.url;
  const method = req.method;
  switch(url) {
    case '/':
      res.writeHead(200, {'Content-Type': 'text/html'});
      res.end('<h1>Hello World</h1>');
      break;
    case '/api/data':
      res.writeHead(200, {'Content-Type': 'application/json'});
      res.end(JSON.stringify({
        message: 'Hello World',
        ip: ip,
        method: method
      }));
      break;
    default:
      res.writeHead(404, {'Content-Type': 'text/html'});
      res.end('<h1>404 Not Found</h1>');
  }
});

// Start the server
server.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});

# Test
curl -G http://localhost:8080

# Push the changes
git add .
git commit -S -m "Add web server implementation"
git push -u origin main
```

### Create the app service

```bash
# Install azure-cli
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Authenticate azure-cli
az login

# Configure OIDC secrets on the repo
gh secret set AZURE_TENANT_ID \
  -R :org/the-microservice \
  --body $(az account list --query "[0].tenantId" --output tsv)

gh secret set AZURE_CLIENT_ID \
  -R :org/the-microservice \
  --body $(az ad app list --query "[?displayName=='github-actions'].appId | [0]" --output tsv)

gh secret set AZURE_SUBSCRIPTION_ID \
  -R :org/the-microservice \
  --body $(az account list --query "[0].id" --output tsv)

# Create a new resource group
az group create --name ModernDevDemoRG --location westeurope

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

# Get hostname
az webapp list --query "[0].defaultHostName" --output tsv

# Tests
curl -G https://the-microservice.azurewebsites.net
curl -G https://the-microservice.azurewebsites.net/api/v1/data
curl -G https://the-microservice.azurewebsites.net/something-else
```

### Create testing and deployment workflow

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
  test:
    name: Test the service
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
          output=$(curl -G http://localhost:8080/)
          if [ "$output" != "<h1>Hello World!</h1>" ]; then
            echo "Service response: $output"
            echo "Service is not working"
            exit 1
          else
            echo "Test 200: OK"
          fi

          # Test 404 response
          output=$(curl -G http://127.0.0.1:8080/wrong-path)
          if [ "$output" != "<h1>404 Not Found</h1>" ]; then
            echo "Service response: $output"
            echo "Service is not working"
            exit 1
          else
            echo "Test 404: OK"
          fi

      - name: Setup tmate session
        if: ${{ failure() }}
        uses: mxschmitt/action-tmate@v3

  deploy:
    name: Deploy to production
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        name: 'Checkout repository'

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

### Nuke the setup

```bash
# Destroy the setup
az group delete --name ModernDevDemoRG
```

## References

- <https://docs.microsoft.com/en-us/azure/app-service/deploy-github-actions?tabs=openid>
- <https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure>
- <https://docs.microsoft.com/en-us/azure/app-service/configure-language-nodejs?pivots=platform-linux>
