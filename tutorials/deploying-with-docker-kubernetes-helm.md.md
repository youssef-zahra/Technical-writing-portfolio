
# Deploying a containerized service with Docker, Kubernetes, and Helm

This tutorial walks through taking a service from source code to a running deployment, using Docker to containerize it, Kubernetes to run it, and Helm to manage the configuration. It uses a small fictional service called Taskflow as the example, but the steps apply to containerizing and deploying any service with a similar structure.

By the end, you will have a container image built, a Kubernetes deployment running it, and a Helm chart that makes the whole thing repeatable across environments.

## What you will need

- Docker installed locally
- A running Kubernetes cluster (a local one like minikube or kind works fine for this walkthrough)
- Helm installed and configured to talk to your cluster
- A basic service to deploy (this guide assumes a simple HTTP service with a Dockerfile already in place)

## Step 1: Containerize the service

Start with a Dockerfile that builds a minimal image for the service:

```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

Build and tag the image:

```bash
docker build -t taskflow:1.0.0 .
```

Test it locally before moving to Kubernetes, so you know any problems you hit later are about the deployment, not the container itself:

```bash
docker run -p 3000:3000 taskflow:1.0.0
```

If the service responds correctly at `localhost:3000`, the image is ready to deploy.

## Step 2: Write a Kubernetes deployment manifest

A basic deployment manifest tells Kubernetes how many copies of the service to run and how to run them:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskflow
spec:
  replicas: 3
  selector:
    matchLabels:
      app: taskflow
  template:
    metadata:
      labels:
        app: taskflow
    spec:
      containers:
        - name: taskflow
          image: taskflow:1.0.0
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

The `replicas: 3` setting is worth calling out specifically. It tells Kubernetes to keep three copies of the service running at all times, so if one pod crashes or gets rescheduled, the other two keep serving traffic while Kubernetes replaces it. That is the core value Kubernetes adds over just running a container directly, it actively keeps your desired state running, rather than running something once and leaving it there.

Apply the manifest:

```bash
kubectl apply -f deployment.yaml
```

Alongside the deployment, you also need a service to expose it:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: taskflow
spec:
  selector:
    app: taskflow
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

## Step 3: Package it with Helm

Writing raw YAML manifests works fine for one environment, but it gets repetitive fast once you need slightly different configuration for local, staging, and production. Helm solves this by templating the manifests and letting you swap in different values per environment.

Scaffold a chart:

```bash
helm create taskflow-chart
```

This generates a chart structure with a `values.yaml` file for configuration and a `templates/` folder for your templated manifests. Replace the generated deployment template with a templated version of what you wrote by hand in Step 2:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

And the matching values in `values.yaml`:

```yaml
replicaCount: 3
image:
  repository: taskflow
  tag: "1.0.0"
service:
  port: 3000
```

Now the replica count, image version, and port are all configuration, not hardcoded values buried in a manifest. Deploying to a different environment is a matter of overriding values, not editing YAML by hand:

```bash
helm install taskflow ./taskflow-chart --set replicaCount=5 --set image.tag=1.1.0
```

## Step 4: Verify the deployment

Confirm the pods are running:

```bash
kubectl get pods -l app=taskflow
```

And check the logs of a specific pod if something looks wrong:

```bash
kubectl logs <pod-name>
```

If all replicas show as `Running` and the service responds when you port-forward or hit it through your cluster's ingress, the deployment is working end to end, from container image to a running, self-healing service.

## Why Helm matters here, not just Kubernetes

Kubernetes on its own already gives you self-healing and scaling. Helm's value is specifically about configuration management across environments. Without it, keeping local, staging, and production manifests in sync means manually tracking which YAML file has which values, and that gets error-prone fast, especially across a team. With Helm, the difference between environments lives in a values file, not in hand-edited copies of the same manifest, which makes the deployment process both faster to repeat and easier to review.
