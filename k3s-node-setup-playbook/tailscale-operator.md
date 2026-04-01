# Tailscale Operator Setup

The Tailscale operator lets Kubernetes services get their own Tailscale IP by setting `loadBalancerClass: tailscale`. This is the primary way to expose services in this cluster since all nodes run over Tailscale.

## Prerequisites

### 1. Tailscale ACL tags

In the Tailscale admin console under **Access Controls**, add to `tagOwners`:

```json
"tagOwners": {
    "tag:k8s-operator": ["autogroup:admin"],
    "tag:k8s":          ["tag:k8s-operator"],
}
```

`tag:k8s-operator` must own `tag:k8s` so the operator can create auth keys for devices it manages.

### 2. OAuth client

In **Tailscale admin console → Settings → OAuth clients**, create a client with:

- Scopes: `Devices Core`, `Auth Keys`, `Services` — all **write**
- Tag: `tag:k8s-operator` on all scopes

Copy the client ID and secret — the secret is only shown once.

---

## Installation

```bash
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update
helm upgrade --install tailscale-operator tailscale/tailscale-operator --namespace=tailscale --create-namespace -f values.yaml --wait
```

Where `values.yaml` contains:

```yaml
oauth:
  clientId: "your-client-id"
  clientSecret: "your-client-secret"
```

### Known issue — secret not created automatically

The helm chart may fail to create the `operator-oauth` secret. If the operator pod is stuck in `ContainerCreating` with `secret "operator-oauth" not found`, create it manually:

```bash
kubectl create secret generic operator-oauth -n tailscale \
  --from-literal=client_id="<client-id>" \
  --from-literal=client_secret="<client-secret>"
```

---

## Using the operator

Add `loadBalancerClass: tailscale` to any `LoadBalancer` service:

```yaml
spec:
  type: LoadBalancer
  loadBalancerClass: tailscale
```

The operator will assign a `100.x.x.x` Tailscale IP. Check it with:

```bash
kubectl get svc <name> -n <namespace>
```

Then add a Pi-hole local DNS record pointing your hostname to that IP (**Local DNS → DNS Records**).

> **Note:** `loadBalancerClass` is immutable. If a service already has a different class set (e.g. MetalLB), delete and recreate it.

---

## Verify

```bash
kubectl get pods -n tailscale       # operator should be Running
kubectl get svc -n kube-system traefik  # should show a 100.x.x.x EXTERNAL-IP
```
