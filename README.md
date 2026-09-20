# DigitalOcean Demo App on DigitalOcean Kubernetes (DOKS)

A single-page Flask app styled with DigitalOcean's brand blue, deployed to DigitalOcean Kubernetes with a DigitalOcean Load Balancer and Horizontal Pod Autoscaling (HPA). Built to demo autoscaling behavior live to a customer.

## Repository layout

This project is split across two folders:

| Folder | Contents |
| --- | --- |
| `digitalocean-demo/` | Application code: `app.py`, `requirements.txt`, `Dockerfile`, `.dockerignore`, `templates/index.html` |
| `digitalocean-demo-yaml/` | Kubernetes manifests: `deployment.yaml`, `service.yaml`, `hpa.yaml`, `load-generator.yaml` |

### Application files

| File | Purpose |
| --- | --- |
| `app.py` | Flask app: `/` renders the demo page (shows the serving pod's hostname), `/healthz` for liveness/readiness probes |
| `requirements.txt` | Python dependencies (Flask, gunicorn) |
| `Dockerfile` | Container image definition (Python 3.12-slim, non-root user, gunicorn) |
| `templates/index.html` | Single-page UI — DigitalOcean blue gradient background, "Welcome to DigitalOcean" heading |

### Kubernetes manifests

| File | Purpose |
| --- | --- |
| `deployment.yaml` | Deployment (2 replicas, resource requests/limits, liveness/readiness probes) |
| `service.yaml` | Service, `type: LoadBalancer` |
| `hpa.yaml` | HorizontalPodAutoscaler (2-6 replicas, 55% CPU target) |
| `load-generator.yaml` | 8-replica BusyBox Deployment used only to generate load for the autoscaling demo (not part of the app) |

## Architecture

External traffic → DigitalOcean Load Balancer → Kubernetes Service → Pods (Deployment). Images are pulled from a private DigitalOcean Container Registry. The Horizontal Pod Autoscaler scales pod count based on CPU usage (via metrics-server); the DOKS Cluster Autoscaler independently scales node count when pods can no longer be scheduled on existing nodes.

## Prerequisites

- A DigitalOcean account with a payment method on file
- [`doctl`](https://docs.digitalocean.com/reference/doctl/how-to/install/) installed and authenticated (`doctl auth init`)
- `kubectl` installed
- Docker installed and running

## Deployment instructions

### 1. Build and push the image

Build explicitly for `amd64` (DOKS worker nodes are amd64, regardless of the architecture of the machine you build on):

```bash
cd digitalocean-demo
docker build --platform linux/amd64 -t digitalocean-demo .
```

Create a registry (one-time) and log in:

```bash
doctl registry create sunil-container-registry --region=nyc3
doctl registry login
```

Tag and push:

```bash
docker tag digitalocean-demo registry.digitalocean.com/sunil-container-registry/digitalocean-demo:v2
docker push registry.digitalocean.com/sunil-container-registry/digitalocean-demo:v2
```

### 2. Create the DOKS cluster

```bash
doctl kubernetes cluster create digitalocean-demo-cluster \
  --region nyc3 \
  --node-pool "name=demo-pool;size=s-2vcpu-4gb;auto-scale=true;min-nodes=1;max-nodes=2"
```

`doctl` merges the kubeconfig automatically once the cluster is ready (takes 3-5 minutes). Verify:

```bash
kubectl get nodes
```

Grant the cluster pull access to the private registry:

```bash
doctl kubernetes cluster registry add digitalocean-demo-cluster
```

### 3. Deploy the application

```bash
cd ../digitalocean-demo-yaml
kubectl apply -f deployment.yaml
kubectl get pods
```

Wait until both pods show `Running`.

### 4. Expose via Load Balancer

```bash
kubectl apply -f service.yaml
kubectl get service digitalocean-demo-lb --watch
```

Once `EXTERNAL-IP` is populated (a couple of minutes), test it:

```bash
curl http://<EXTERNAL-IP>/
curl http://<EXTERNAL-IP>/healthz
```

### 5. Configure autoscaling

Install metrics-server (required for HPA to read CPU usage):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

If `kubectl top nodes` returns an error instead of numbers (a known DOKS quirk with self-signed kubelet certs), patch it:

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

Apply the HPA:

```bash
kubectl apply -f hpa.yaml
kubectl get hpa digitalocean-demo-hpa --watch
```

### 6. (Optional) Load-test the autoscaling

```bash
kubectl apply -f load-generator.yaml
kubectl get hpa digitalocean-demo-hpa --watch
kubectl get nodes --watch
kubectl delete -f load-generator.yaml
```

Replica count should climb under load and scale back down a few minutes after the load generator is removed (HPA's default 5-minute scale-down stabilization window). Node count may also increase if the extra pods can't be scheduled on the existing node(s); the new node typically appears within 3-5 minutes and is removed roughly 10-15 minutes after load stops, once the cluster autoscaler confirms it's no longer needed.

## Cleanup

```bash
cd digitalocean-demo-yaml
kubectl delete -f hpa.yaml -f service.yaml -f deployment.yaml
doctl kubernetes cluster delete digitalocean-demo-cluster
doctl registry delete sunil-container-registry
```
