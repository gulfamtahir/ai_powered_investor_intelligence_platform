# Mac (Apple Silicon) AKS Deployment Guide - AI Powered Investor Intelligence Platform

This is a corrected version of `deployment-Document.md`, based on what actually happened deploying
this app from an Apple Silicon (arm64) Mac to AKS. It covers everything up to exposing the app
publicly — it does not cover production hardening.

Always follow this guide for the mac deployment locally

Deployment Flow:

```text
Local Application (Mac, arm64)
    ↓
Docker Image (built for linux/amd64 — see Step 1)
    ↓
Azure Container Registry (ACR)
    ↓
Azure Kubernetes Service (AKS, amd64 nodes)
    ↓
Azure PostgreSQL
    ↓
Running Pod
```

---

# Step 1: Build the Docker Image for linux/amd64

AKS nodes run `linux/amd64`. An Apple Silicon Mac builds `linux/arm64` by default, and a plain
`docker build` will NOT fix this even if the Dockerfile has `FROM --platform=linux/amd64` pinned —
that pin alone does not guarantee the pushed manifest is labeled `amd64`. The `--platform` flag
must be passed explicitly to the build command itself.

Build and push in one step using `buildx`:

```bash
docker buildx build --platform linux/amd64 \
  -t invintelligence535.azurecr.io/investor-intelligence-platform:v1 \
  --push .
```

Verify the pushed manifest is actually amd64 before moving on:

```bash
docker buildx imagetools inspect invintelligence535.azurecr.io/investor-intelligence-platform:v1
```

Expected: `Platform: linux/amd64` in the output. If it says `linux/arm64`, the image will fail to
pull on AKS nodes with an error like:

```text
Failed to pull image ...: no match for platform in manifest ...: not found
```

---

# Step 2: Run Docker Locally (optional sanity check)

```bash
docker run -p 8000:8000 --env-file .env investor-intelligence-platform:latest
```

Open browser:

```text
http://localhost:8000
```

Verify:

* Application loads successfully
* Dashboard renders
* PDF upload works
* Chatbot works

Stop with `Ctrl + C`.

---

# Step 3: Login to Azure

```bash
az login
az account show
```

---

# Step 4: Login to Azure Container Registry (ACR)

```bash
az acr login --name invintelligence535
```

If you need the raw credentials (e.g. for the imagePullSecret in Step 6):

```bash
az acr credential show --name invintelligence535
```

---

# Step 5: Connect kubectl to AKS

```bash
az aks get-credentials --resource-group rg-inv-intelligence --name inv-intelligence-aks --overwrite-existing
```

Verify:

```bash
kubectl get nodes
```

Confirm the node architecture matches your image (should be `amd64`):

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  arch="}{.status.nodeInfo.architecture}{"\n"}{end}'
```

---

# Step 6: Create Image Pull Secret

This project uses the imagePullSecret method (not `az aks update --attach-acr`) so AKS can pull
from ACR.

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=invintelligence535.azurecr.io \
  --docker-username=invintelligence535 \
  --docker-password=<acr-password>
```

Wire the default service account to use it:

```bash
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "acr-secret"}]}'
```

Note: `az aks check-acr` will still report a 401 after this — that command only tests the
managed-identity (`--attach-acr`) path, not the imagePullSecret path used here. That failure does
not mean this method is broken.

---

# Step 7: Allow AKS to Reach Azure PostgreSQL (Firewall Rule)

Azure PostgreSQL Flexible Server blocks all traffic by default except IPs explicitly allow-listed
in its firewall rules. AKS pods connect out through the cluster's **outbound IP**, which almost
certainly is NOT already allow-listed (only your local dev machine's IP typically is).

Find the AKS outbound IP:

```bash
az aks show --resource-group rg-inv-intelligence --name inv-intelligence-aks \
  --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs" -o tsv

az network public-ip show --ids <resource-id-from-above> --query "ipAddress" -o tsv
```

Check what the Postgres server currently allows:

```bash
az postgres flexible-server firewall-rule list \
  --resource-group rg-inv-intelligence --server-name inv-intelligence-535 -o table
```

If the AKS outbound IP isn't in that list, add it:

```bash
az postgres flexible-server firewall-rule create \
  --resource-group rg-inv-intelligence --server-name inv-intelligence-535 \
  --name AllowAKSOutbound \
  --start-ip-address <aks-outbound-ip> --end-ip-address <aks-outbound-ip>
```

Skipping this step causes the pod to crash-loop on startup with:

```text
psycopg2.OperationalError: connection to server at "...postgres.database.azure.com" (...) failed: Connection timed out
```

---

# Step 8: Deploy the Application

Use the **full image name as pushed to ACR** (`investor-intelligence-platform`), not a shortened
name like `invint`. The deployment itself can be named `invint`, but the image reference must be
exact:

```bash
kubectl create deployment invint --image=invintelligence535.azurecr.io/investor-intelligence-platform:v1
```

Verify:

```bash
kubectl get deployments
```

---

# Step 9: Load Environment Variables into the Pod

`kubectl create deployment` does not read your local `.env` file — the container starts with zero
environment variables, which will crash the app on startup (e.g. Postgres connection failures).

Create a Kubernetes Secret from `.env` and attach it to the deployment:

```bash
kubectl create secret generic app-env --from-env-file=.env
kubectl set env deployment/invint --from=secret/app-env
```

`kubectl set env` patches the deployment and automatically triggers a new rollout.

---

# Step 10: Monitor Pod Startup

```bash
kubectl get pods
kubectl get pods -w
```

Expected:

```text
READY   STATUS    RESTARTS
1/1     Running   0
```

If you see `ErrImagePull` / `ImagePullBackOff`, re-check Step 1 (platform) and Step 8 (image
name). If you see the pod crash-looping with `Connection timed out` to Postgres, check Step 7
(firewall rule). If it crash-loops for another reason, check Step 9 (env vars).

---

# Step 11: View Logs

```bash
kubectl logs <pod-name>
kubectl logs -f <pod-name>
kubectl describe pod <pod-name>
```

Find the current image and container name of a running pod:

```bash
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
kubectl get deployment invint -o jsonpath='{.spec.template.spec.containers[*].name}'
```

---

# Step 12: Expose Application

Create a public Load Balancer:

```bash
kubectl expose deployment invint --type=LoadBalancer --port=80 --target-port=8000
```

Verify the service:

```bash
kubectl get svc
```

Expected output:

```text
NAME     TYPE           EXTERNAL-IP
invint   LoadBalancer   20.xxx.xxx.xxx
```

`EXTERNAL-IP` may show `<pending>` for a minute or two while Azure provisions the load balancer —
re-run `kubectl get svc` until an IP appears. The app is then reachable at `http://<external-ip>`.

---

# Rolling Out a New Image Version

Because Kubernetes does not repull an already-used tag, always build with a **new tag** (`v2`,
`v3`, ...) when pushing an update:

```bash
docker buildx build --platform linux/amd64 \
  -t invintelligence535.azurecr.io/investor-intelligence-platform:v2 \
  --push .

kubectl set image deployment/invint investor-intelligence-platform=invintelligence535.azurecr.io/investor-intelligence-platform:v2
kubectl rollout status deployment invint
```

Clean up old, unused tags from ACR once a new one is confirmed working:

```bash
az acr repository delete --name invintelligence535 --image investor-intelligence-platform:v1 --yes
```

---

# Issues Hit During This Deployment (Root Causes)

1. **Wrong image name in deployment** — deployed as `invint`, but the image was pushed to ACR as
   `investor-intelligence-platform`. `kubectl create deployment <name> --image=...` names the
   *deployment* whatever you tell it, but auto-names the *container* after the image repo name —
   these are three different names (Deployment → Pod → Container) and only the first is chosen by
   you.
2. **Platform mismatch (arm64 vs amd64)** — built on an Apple Silicon Mac (arm64), but AKS nodes
   run amd64. A plain `docker build`, even with `FROM --platform=linux/amd64` in the Dockerfile,
   does not reliably produce a correctly-labeled amd64 manifest — the `--platform` flag must be
   passed explicitly to `docker buildx build`. Always verify with `docker buildx imagetools
   inspect` before deploying.
3. **Missing environment variables** — `kubectl create deployment` starts the container with no
   env vars at all, so the app couldn't reach Postgres/Azure services. Fixed by creating the
   `app-env` Secret from `.env` and attaching it with `kubectl set env`.
4. **Postgres firewall blocked the AKS outbound IP** — even with correct env vars, the pod
   crash-looped (100+ restarts) with `Connection timed out` reaching
   `inv-intelligence-535.postgres.database.azure.com`. The Postgres server's firewall only
   allow-listed the developer's local machine IP, not AKS's outbound IP. Fixed by adding a
   firewall rule for the AKS outbound IP (Step 7). Diagnosed by comparing
   `az aks show ... effectiveOutboundIPs` against
   `az postgres flexible-server firewall-rule list`.

---

# Terminology Quick Reference

* **Node** = the VM (the machine that runs your workloads).
* **Pod** = a wrapper around one or more containers, plus shared networking/storage — not itself a
  container.
* **Container** = the actual running process, built from your Docker image.
* **Deployment** = a controller object that manages and recreates Pods for you; it does not run
  anything itself.

Chain: **Deployment** (manager) → **Pod** (wrapper) → **Container** (your app).
