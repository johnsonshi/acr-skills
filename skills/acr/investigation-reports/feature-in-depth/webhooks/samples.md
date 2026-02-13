# ACR Webhook Samples and Use Cases

## Overview

This document provides practical examples and sample implementations for common ACR webhook use cases, including CI/CD integration, deployment automation, and notification systems.

---

## Sample 1: Automated Web App Deployment

### Scenario
Automatically deploy container images to Azure Web App for Containers when a new image is pushed.

### Setup

When you deploy a web app from a container image in your registry, Azure Container Registry automatically creates an image deployment webhook for you. This enables continuous deployment.

### Azure Portal Setup

1. Navigate to your container registry
2. Select **Repositories** > your repository
3. Right-click on the tag and select **Deploy to web app**
4. Configure the Web App settings
5. Azure automatically creates the webhook

### Verification

After pushing an image:
```bash
docker push myregistry.azurecr.io/myapp:latest
```

The webhook automatically triggers deployment. View webhook logs in the portal under **Webhooks**.

---

## Sample 2: Webhook Handler for Push Notifications

### Python Flask Webhook Handler

```python
from flask import Flask, request, jsonify
import logging
import json

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

@app.route('/webhook', methods=['POST'])
def handle_acr_webhook():
    """Handle ACR webhook notifications."""
    event = request.get_json()

    # Log the incoming event
    logging.info(f"Received webhook event: {json.dumps(event, indent=2)}")

    action = event.get('action')
    target = event.get('target', {})
    timestamp = event.get('timestamp')

    if action == 'push':
        repository = target.get('repository')
        tag = target.get('tag')
        digest = target.get('digest')
        registry = event.get('request', {}).get('host')

        logging.info(f"Image pushed: {registry}/{repository}:{tag}")
        logging.info(f"Digest: {digest}")

        # Trigger your deployment logic here
        trigger_deployment(registry, repository, tag, digest)

    elif action == 'delete':
        repository = target.get('repository')
        digest = target.get('digest')

        logging.info(f"Image deleted: {repository} ({digest})")

        # Handle cleanup logic
        handle_image_deletion(repository, digest)

    elif action == 'quarantine':
        repository = target.get('repository')
        tag = target.get('tag')

        logging.info(f"Image quarantined: {repository}:{tag}")

        # Trigger security scan
        trigger_security_scan(repository, tag)

    return jsonify({'status': 'ok'}), 200


def trigger_deployment(registry, repository, tag, digest):
    """Trigger deployment based on image push."""
    # Example: Call Kubernetes API to update deployment
    # Example: Trigger Azure DevOps pipeline
    # Example: Send notification to Slack
    pass


def handle_image_deletion(repository, digest):
    """Handle image deletion event."""
    # Example: Update inventory database
    # Example: Clean up related resources
    pass


def trigger_security_scan(repository, tag):
    """Trigger security scan for quarantined image."""
    # Example: Call vulnerability scanner API
    pass


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Create the Webhook

```bash
az acr webhook create \
  --registry myregistry \
  --name deployment-webhook \
  --actions push delete \
  --uri https://mywebhookhandler.example.com/webhook \
  --headers "X-Webhook-Secret=mysecrettoken"
```

---

## Sample 3: Node.js Webhook Handler with Kubernetes Deployment

### Express.js Handler with kubectl Integration

```javascript
const express = require('express');
const { exec } = require('child_process');
const app = express();

app.use(express.json());

// Webhook endpoint
app.post('/webhook', async (req, res) => {
    const event = req.body;
    console.log('Received webhook:', JSON.stringify(event, null, 2));

    if (event.action === 'push') {
        const { repository, tag, digest } = event.target;
        const registry = event.request.host;
        const image = `${registry}/${repository}:${tag}`;

        console.log(`New image pushed: ${image}`);

        try {
            // Update Kubernetes deployment
            await updateKubernetesDeployment(repository, image);
            res.status(200).json({ status: 'deployment triggered' });
        } catch (error) {
            console.error('Deployment failed:', error);
            res.status(500).json({ status: 'deployment failed', error: error.message });
        }
    } else {
        res.status(200).json({ status: 'event received' });
    }
});

async function updateKubernetesDeployment(deploymentName, image) {
    return new Promise((resolve, reject) => {
        const cmd = `kubectl set image deployment/${deploymentName} ${deploymentName}=${image}`;
        exec(cmd, (error, stdout, stderr) => {
            if (error) {
                reject(new Error(stderr));
            } else {
                console.log(`Deployment updated: ${stdout}`);
                resolve(stdout);
            }
        });
    });
}

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Webhook handler listening on port ${PORT}`);
});
```

---

## Sample 4: Slack Notification on Image Push

### Python Handler with Slack Integration

```python
from flask import Flask, request, jsonify
import requests
import os

app = Flask(__name__)

SLACK_WEBHOOK_URL = os.environ.get('SLACK_WEBHOOK_URL')

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    event = request.get_json()
    action = event.get('action')

    if action == 'push':
        target = event.get('target', {})
        registry = event.get('request', {}).get('host')

        message = {
            "blocks": [
                {
                    "type": "header",
                    "text": {
                        "type": "plain_text",
                        "text": "New Container Image Pushed",
                        "emoji": True
                    }
                },
                {
                    "type": "section",
                    "fields": [
                        {
                            "type": "mrkdwn",
                            "text": f"*Registry:*\n{registry}"
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Repository:*\n{target.get('repository')}"
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Tag:*\n{target.get('tag')}"
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Digest:*\n`{target.get('digest')[:20]}...`"
                        }
                    ]
                },
                {
                    "type": "context",
                    "elements": [
                        {
                            "type": "mrkdwn",
                            "text": f"Pushed at: {event.get('timestamp')}"
                        }
                    ]
                }
            ]
        }

        # Send to Slack
        if SLACK_WEBHOOK_URL:
            requests.post(SLACK_WEBHOOK_URL, json=message)

    elif action == 'delete':
        target = event.get('target', {})

        message = {
            "text": f"Image deleted from {target.get('repository')}"
        }

        if SLACK_WEBHOOK_URL:
            requests.post(SLACK_WEBHOOK_URL, json=message)

    return jsonify({'status': 'ok'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Sample 5: Azure Functions Webhook Handler

### Azure Function (Python)

```python
import logging
import azure.functions as func
import json

def main(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('ACR webhook triggered')

    try:
        event = req.get_json()
    except ValueError:
        return func.HttpResponse(
            "Invalid JSON",
            status_code=400
        )

    action = event.get('action')
    target = event.get('target', {})

    if action == 'push':
        repository = target.get('repository')
        tag = target.get('tag')
        digest = target.get('digest')

        logging.info(f'Image pushed: {repository}:{tag}')
        logging.info(f'Digest: {digest}')

        # Add your custom logic here
        # - Trigger Azure DevOps pipeline
        # - Update Azure Container Instances
        # - Send notification

    elif action == 'delete':
        repository = target.get('repository')
        digest = target.get('digest')

        logging.info(f'Image deleted: {repository} ({digest})')

    return func.HttpResponse(
        json.dumps({"status": "ok"}),
        status_code=200,
        mimetype="application/json"
    )
```

### function.json

```json
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["post"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ]
}
```

---

## Sample 6: Geo-Replication Tracking

### Tracking Image Replication Across Regions

Create webhooks for each region to track when images complete replication:

```bash
# Create webhook for West US replica
az acr webhook create \
  --registry myregistry \
  --name westus-tracker \
  --actions push \
  --uri https://mytracker.example.com/webhook?region=westus \
  --location westus

# Create webhook for East US replica
az acr webhook create \
  --registry myregistry \
  --name eastus-tracker \
  --actions push \
  --uri https://mytracker.example.com/webhook?region=eastus \
  --location eastus

# Create webhook for West Europe replica
az acr webhook create \
  --registry myregistry \
  --name westeurope-tracker \
  --actions push \
  --uri https://mytracker.example.com/webhook?region=westeurope \
  --location westeurope
```

### Handler for Replication Tracking

```python
from flask import Flask, request, jsonify
from datetime import datetime
import json

app = Flask(__name__)

# Track replication status
replication_status = {}

@app.route('/webhook', methods=['POST'])
def track_replication():
    region = request.args.get('region')
    event = request.get_json()

    if event.get('action') == 'push':
        target = event.get('target', {})
        image_key = f"{target.get('repository')}:{target.get('tag')}"

        if image_key not in replication_status:
            replication_status[image_key] = {}

        replication_status[image_key][region] = {
            'timestamp': datetime.utcnow().isoformat(),
            'digest': target.get('digest')
        }

        print(f"Image {image_key} replicated to {region}")
        print(f"Replication status: {json.dumps(replication_status[image_key], indent=2)}")

    return jsonify({'status': 'ok'}), 200

@app.route('/status/<path:image>', methods=['GET'])
def get_status(image):
    """Get replication status for an image."""
    return jsonify(replication_status.get(image, {}))

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Sample 7: Security Scanning with Quarantine

### Quarantine Webhook Handler for Security Scanner

```python
from flask import Flask, request, jsonify
import requests
import os

app = Flask(__name__)

ACR_REGISTRY = os.environ.get('ACR_REGISTRY')
SCANNER_API = os.environ.get('SCANNER_API')

@app.route('/quarantine-webhook', methods=['POST'])
def handle_quarantine():
    event = request.get_json()

    if event.get('action') == 'quarantine':
        target = event.get('target', {})
        repository = target.get('repository')
        tag = target.get('tag')
        digest = target.get('digest')

        print(f"Image quarantined: {repository}:{tag}")

        # Trigger security scan
        scan_result = trigger_scan(repository, digest)

        if scan_result['passed']:
            # Mark image as passed - release from quarantine
            release_from_quarantine(repository, digest, scan_result)
        else:
            # Mark image as failed - keep in quarantine
            mark_as_failed(repository, digest, scan_result)

    return jsonify({'status': 'ok'}), 200


def trigger_scan(repository, digest):
    """Trigger vulnerability scan."""
    # Call your scanner API
    # response = requests.post(f"{SCANNER_API}/scan", json={...})
    # return response.json()

    # Placeholder
    return {'passed': True, 'vulnerabilities': []}


def release_from_quarantine(repository, digest, scan_result):
    """Release image from quarantine by marking as Passed."""
    # Use ACR REST API to update quarantine state
    # PATCH /acr/v1/{repository}/_manifests/{digest}
    # Body: {"quarantineState": "Passed", "quarantineDetails": "..."}
    pass


def mark_as_failed(repository, digest, scan_result):
    """Keep image in quarantine by marking as Failed."""
    # PATCH /acr/v1/{repository}/_manifests/{digest}
    # Body: {"quarantineState": "Failed", "quarantineDetails": "..."}
    pass


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Sample 8: Event Grid Integration Alternative

### Creating Event Grid Subscription for ACR

```bash
# Get registry resource ID
ACR_REGISTRY_ID=$(az acr show --name myregistry --query id --output tsv)

# Create Event Grid event subscription
az eventgrid event-subscription create \
  --name acr-events \
  --source-resource-id $ACR_REGISTRY_ID \
  --endpoint https://myhandler.example.com/eventgrid
```

### Event Grid Handler

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/eventgrid', methods=['POST'])
def handle_eventgrid():
    # Handle Event Grid validation
    if request.headers.get('aeg-event-type') == 'SubscriptionValidation':
        event = request.get_json()[0]
        validation_code = event['data']['validationCode']
        return jsonify({'validationResponse': validation_code})

    # Handle events
    events = request.get_json()
    for event in events:
        event_type = event.get('eventType')

        if event_type == 'Microsoft.ContainerRegistry.ImagePushed':
            data = event.get('data', {})
            print(f"Image pushed: {data.get('target', {}).get('repository')}")

        elif event_type == 'Microsoft.ContainerRegistry.ImageDeleted':
            data = event.get('data', {})
            print(f"Image deleted: {data.get('target', {}).get('repository')}")

    return jsonify({'status': 'ok'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Sample 9: Webhook Authentication Best Practices

### HMAC Signature Verification

```python
from flask import Flask, request, jsonify, abort
import hmac
import hashlib
import os

app = Flask(__name__)

WEBHOOK_SECRET = os.environ.get('WEBHOOK_SECRET', 'your-secret-key')

@app.route('/webhook', methods=['POST'])
def secure_webhook():
    # Verify signature from custom header
    signature = request.headers.get('X-ACR-Signature')
    if not signature:
        abort(401, 'Missing signature')

    # Calculate expected signature
    payload = request.get_data()
    expected_signature = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature, expected_signature):
        abort(401, 'Invalid signature')

    # Process the webhook
    event = request.get_json()
    # ... handle event ...

    return jsonify({'status': 'ok'}), 200
```

### Create Webhook with Authentication Header

```bash
az acr webhook create \
  --registry myregistry \
  --name secure-webhook \
  --actions push \
  --uri https://myhandler.example.com/webhook \
  --headers "X-ACR-Signature=$(echo -n 'webhook-secret' | sha256sum | cut -d' ' -f1)"
```

---

## Sample 10: Complete CI/CD Pipeline Integration

### End-to-End Workflow

```bash
#!/bin/bash

# Variables
REGISTRY="myregistry"
WEBHOOK_NAME="cicd-webhook"
PIPELINE_ENDPOINT="https://dev.azure.com/myorg/myproject/_apis/webhooks/receivers/genericweb"

# 1. Create webhook for CI/CD
az acr webhook create \
  --registry $REGISTRY \
  --name $WEBHOOK_NAME \
  --actions push \
  --uri $PIPELINE_ENDPOINT \
  --headers "Authorization=Bearer $AZURE_DEVOPS_TOKEN" \
  --scope production/*

# 2. Test the webhook
az acr webhook ping --registry $REGISTRY --name $WEBHOOK_NAME

# 3. Verify configuration
az acr webhook show --registry $REGISTRY --name $WEBHOOK_NAME

# 4. Build and push image (this triggers the webhook)
az acr build \
  --registry $REGISTRY \
  --image production/myapp:$(git rev-parse --short HEAD) \
  .

# 5. Check webhook events
az acr webhook list-events --registry $REGISTRY --name $WEBHOOK_NAME
```

---

## Deployment Options

### Docker Compose for Webhook Handler

```yaml
version: '3.8'

services:
  webhook-handler:
    build: .
    ports:
      - "5000:5000"
    environment:
      - SLACK_WEBHOOK_URL=${SLACK_WEBHOOK_URL}
      - WEBHOOK_SECRET=${WEBHOOK_SECRET}
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: acr-webhook-handler
spec:
  replicas: 2
  selector:
    matchLabels:
      app: acr-webhook-handler
  template:
    metadata:
      labels:
        app: acr-webhook-handler
    spec:
      containers:
      - name: handler
        image: myregistry.azurecr.io/webhook-handler:latest
        ports:
        - containerPort: 5000
        env:
        - name: WEBHOOK_SECRET
          valueFrom:
            secretKeyRef:
              name: webhook-secrets
              key: secret
---
apiVersion: v1
kind: Service
metadata:
  name: acr-webhook-handler
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: acr-webhook-handler
```
