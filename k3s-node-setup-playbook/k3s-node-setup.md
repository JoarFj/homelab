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

**Quick access — port-forward:**
```bash
kubectl port-forward svc/<name> 9000:9000 -n <namespace>
```

**Permanent — externalIPs** (brittle if pod reschedules):
```yaml
spec:
  type: LoadBalancer
  externalIPs:
  - 100.x.x.x   # Tailscale IP of the node the pod runs on
```

**Best — Tailscale operator** (each service gets its own Tailscale IP):
```bash
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm install tailscale-operator tailscale/tailscale-operator \
  -n tailscale-operator --create-namespace \
  --set oauth.clientId=<id> --set oauth.clientSecret=<secret>
```
```yaml
spec:
  type: LoadBalancer
  loadBalancerClass: tailscale
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Worker shows LAN IP | Add config.yaml before installing, restart `k3s-agent` |
| Worker not joining | `ping 100.x.x.x` must work both ways first |
| Node stuck `NotReady` | `sudo journalctl -u k3s-agent -f` — usually token mismatch |
| Service stuck `<pending>` | Check Klipper pod logs, confirm port isn't already in use |
