# K3s Node Setup Runbook

## Prerequisites (every node)

```bash
# install Tailscale and authenticate
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --authkey=<reusable-auth-key>
tailscale ip -4  # note this — you'll need it below

# confirm nodes can reach each other
ping 100.x.x.x   # ping the other node's Tailscale IP
```

Get a reusable auth key at [login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys).

---

## Control Plane

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
node-ip: 100.x.x.x
node-external-ip: 100.x.x.x
flannel-iface: tailscale0
write-kubeconfig-mode: "0644"
tls-san:
  - 100.x.x.x
EOF

curl -sfL https://get.k3s.io | sh -

# save this token — workers need it
sudo cat /var/lib/rancher/k3s/server/node-token
```

---

## Worker Node

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
server: https://100.x.x.x:6443   # control plane Tailscale IP
token: <node-token>
node-ip: 100.x.x.x               # this worker's Tailscale IP
node-external-ip: 100.x.x.x
flannel-iface: tailscale0
EOF

curl -sfL https://get.k3s.io | K3S_URL=https://100.x.x.x:6443 K3S_TOKEN=<token> sh -s - agent
```

Verify from control plane — `INTERNAL-IP` should show `100.x.x.x`:
```bash
kubectl get nodes -o wide
```

---

## Remote kubeconfig

```bash
# copy /etc/rancher/k3s/k3s.yaml from control plane to ~/.kube/config
# then point it at the Tailscale IP instead of localhost
sed -i 's/127.0.0.1/100.x.x.x/g' ~/.kube/config   # Linux
sed -i '' 's/127.0.0.1/100.x.x.x/g' ~/.kube/config # macOS
```

---

## Exposing Services

The cluster runs entirely over Tailscale (nodes use `100.x.x.x` as their node IPs, flannel runs over `tailscale0`). MetalLB L2 mode does not work in this setup — it relies on ARP which doesn't propagate correctly over Tailscale/WiFi.

**Quick access — port-forward:**
```bash
kubectl port-forward svc/<name> 9000:9000 -n <namespace>
```

**Permanent — Tailscale operator** (each service gets its own Tailscale IP):

The Tailscale operator is installed in the `tailscale` namespace. To expose a service, set `loadBalancerClass: tailscale`:

```yaml
spec:
  type: LoadBalancer
  loadBalancerClass: tailscale
```

The operator assigns a `100.x.x.x` Tailscale IP to the service. Add a Pi-hole local DNS record pointing your chosen hostname to that IP.

> **Note:** Services are only reachable from devices connected to Tailscale.

---

## Tailscale Operator Setup

See `tailscale-operator.md` for full installation notes.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Worker shows LAN IP | Add config.yaml before installing, restart `k3s-agent` |
| Worker not joining | `ping 100.x.x.x` must work both ways first |
| Node stuck `NotReady` | `sudo journalctl -u k3s-agent -f` — usually token mismatch |
| Service stuck `<pending>` | Check Tailscale operator pod logs in `tailscale` namespace |
| MetalLB IP assigned instead of Tailscale | `loadBalancerClass: tailscale` is immutable — delete and recreate the service |
