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
git push origin main

# Go to the repo
gh repo view --web

# BROWSER: Create a new codespace

# Connect to codespace from local VSCode
gh cs list
gh cs code -c <name>

# Create an index.js file and write a simple web server
#!/usr/env node
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
```

## Create the VM and deploy

```bash
# Install azure-cli
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Authenticate azure-cli
az login

# Create a new resource group
az group create --name ModernDevDemoRG --location westeurope

# Create the VM
az vm create \
  --resource-group ModernDevDemoRG \
  --name theMicroservice \
  --image Debian \
  --location westeurope \
  --public-ip-sku Standard \
  --admin-username sysadmin \
  --generate-ssh-keys

# Install Node
az vm run-command invoke \
   --resource-group ModernDevDemoRG \
   --name theMicroservice \
   --command-id RunShellScript \
   --scripts "sudo apt-get update && sudo apt-get install -y nodejs"

# Open the port 3005 to the world
az vm open-port --port 3005 --resource-group ModernDevDemoRG --name theMicroservice

# SSH into the VM
ssh -i ~/.ssh/id_rsa sysadmin@20.31.113.139

# Deploy & run the service
HOSTNAME='20.31.113.139' && \
USERNAME='sysadmin' && \
WORK_DIR='/home/sysadmin/service/' && \
ssh -i ~/.ssh/id_rsa $USERNAME@$HOSTNAME "mkdir -p $WORK_DIR" && \
scp -i $HOME/.ssh/id_rsa /workspaces/the-microservice/*.js $USERNAME@$HOSTNAME:$WORK_DIR && \
ssh -i ~/.ssh/id_rsa $USERNAME@$HOSTNAME "node $WORK_DIR/index.js >out.log 2>err.log &"

# Smoke tests
curl -G http://$HOSTNAME:3005
curl -G http://$HOSTNAME:3005/api/data
curl -G http://$HOSTNAME:3005/ap
```

## Create deployment workflow

```bash
# Add secrets to the repository
HOSTNAME='20.31.113.139'
USERNAME='sysadmin'
SSH_DEPLOYMENT_PRIVATE_KEY=$(cat ~/.ssh/id_rsa)
```

```yaml
name: Deploy service

on:
  push:
    branches:
      - main
  workflow_dispatch:

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

      - name: Install SSH key
        uses: shimataro/ssh-key-action@3c9b0fc6f2d223b8450b02a0445f526350fc73e0
        with:
          key: ${{ secrets.SSH_DEPLOYMENT_PRIVATE_KEY }}
          name: id_rsa
          known_hosts: unnecessary
          if_key_exists: ignore
          config: |
            Host *
                StrictHostKeyChecking no

      - name: Deploy to production
        run: |
          TARGET_WORK_DIR="/home/${{ secrets.USERNAME }}/service"
          scp -i $HOME/.ssh/id_rsa *.js ${{ secrets.USERNAME }}@${{ secrets.HOSTNAME }}:$TARGET_WORK_DIR
          ssh -i $HOME/.ssh/id_rsa ${{ secrets.USERNAME }}@${{ secrets.HOSTNAME }} "killall node"
          ssh -i $HOME/.ssh/id_rsa ${{ secrets.USERNAME }}@${{ secrets.HOSTNAME }} "node $TARGET_WORK_DIR/index.js >out.log 2>err.log &"

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
