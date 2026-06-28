# Grafana Homelab Observability Stack — 2026

Full monitoring, logging, and tracing for Windows, Linux, and macOS machines
across your Tailscale network.

---

## Architecture

```
  Your Tailnet
  ┌────────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  External Machines (Windows / Linux / macOS)                      │
  │  ┌────────────────────────────────────────────────────────────┐   │
  │  │  Grafana Alloy (agent)                                     │   │
  │  │  metrics → prometheus-homelab:9090/api/v1/write            │   │
  │  │  logs    → loki-homelab/loki/api/v1/push                   │   │
  │  │  traces  → tempo-homelab:4317 (OTLP gRPC)                  │   │
  │  └────────────────────────────────────────────────────────────┘   │
  │                       │              │              │              │
  │         ┌─────────────┘              │              └──────────┐   │
  │         ▼                           ▼                         ▼   │
  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
  │  │ prometheus-home │  │  loki-homelab   │  │  tempo-homelab  │  │
  │  │ lab (Tailscale) │  │  (Tailscale)    │  │  (Tailscale)    │  │
  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  │
  │           │ cluster-internal   │                     │            │
  │           └────────────────────▼─────────────────────┘            │
  │                         ┌───────────┐                             │
  │  grafana-homelab ──────▶│  Grafana  │                             │
  │  (Tailscale)            └───────────┘                             │
  │                                                                    │
  │                    Kubernetes Cluster                              │
  └────────────────────────────────────────────────────────────────────┘
```

**How it works:**
- The Tailscale Kubernetes Operator watches for Ingress resources with
  `ingressClassName: tailscale` and provisions a proxy pod for each,
  giving it a stable MagicDNS hostname with a Let's Encrypt certificate.
- All Helm-managed services are `ClusterIP` — no external IPs on services.
- `tailscale-ingress.yaml` provides the four Ingress resources applied
  once with kubectl, independent of Helm, so they survive chart upgrades.
- Internal Grafana → datasource traffic stays on plain HTTP within the
  cluster. HTTPS is only applied at the Tailscale network boundary.

---

## MagicDNS Hostnames (after setup)

| Service | Tailscale hostname | Protocol | Used by |
|---|---|---|---|
| Grafana | `grafana-homelab.<tailnet>.ts.net` | HTTPS | you (browser) |
| Prometheus | `prometheus-homelab.<tailnet>.ts.net` | HTTPS | Alloy agents (remote_write) |
| Loki Gateway | `loki-homelab.<tailnet>.ts.net` | HTTPS | Alloy agents (log push) |
| Tempo | `tempo-homelab.<tailnet>.ts.net` | HTTPS | Alloy agents (OTLP HTTP traces) |

---

## File Layout

```
grafana-homelab/
├── README.md
├── tailscale-ingress.yaml       ← Tailscale Ingress + HTTPS (kubectl apply)
├── values/
│   ├── prometheus-stack.yaml   ← Prometheus + Grafana + Alertmanager
│   ├── loki.yaml               ← Loki SingleBinary
│   ├── tempo.yaml              ← Tempo
│   └── alloy-cluster.yaml      ← Alloy DaemonSet (in-cluster k8s logs)
└── configs/
    ├── alloy-linux-mac.alloy   ← Alloy agent for Linux / macOS hosts
    └── alloy-windows.alloy     ← Alloy agent for Windows hosts
```

---

## Versions

| Component | Helm Chart | App Version |
|---|---|---|
| kube-prometheus-stack | 87.2.1 | v0.92.0 (Prometheus Operator) |
| Loki | 7.0.0 | 3.6.7 |
| Tempo | 1.24.4 | 2.9.0 |
| Grafana Alloy | 1.10.0 | v1.17.0 |
| Tailscale Operator | 1.98.4 | v1.98.4 |

---

## Prerequisites

- Kubernetes **1.28+**
- Helm **3.14+**
- `kubectl` configured for your cluster
- Default StorageClass available for PVCs
  (e.g. [rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner) if your cluster does not have one)
- Tailscale account with MagicDNS enabled
  (Settings → DNS → Enable MagicDNS)

---

## Step 1 — Add Helm Repos

```bash
helm repo add grafana             https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add tailscale           https://pkgs.tailscale.com/helmcharts
helm repo update
```

---

## Step 2 — Create Namespace

```bash
kubectl create namespace monitoring
```

---

## Step 3 — Install the Tailscale Kubernetes Operator

### 3a. Prepare your Tailscale ACL

Go to **Tailscale admin → Access Controls → JSON editor** and ensure
`tagOwners` contains `tag:k8s`, and that port `443` is allowed in your
`grants` block so HTTPS traffic reaches the Tailscale Ingress proxies:

```json
{
  "tagOwners": {
    "tag:k8s": []
  },
  "grants": [
    {
      "src": ["*"],
      "dst": ["*"],
      "ip": [
        "tcp:443"
      ]
    }
  ]
}
```

If you already have existing grant rules, add `"tcp:443"` to the `ip` list
of the relevant grant rather than replacing the whole block. Save.

### 3b. Create an OAuth client

1. Go to **https://login.tailscale.com/admin/settings/oauth**
2. Click **+ Credential → OAuth**
3. Set these scopes — everything else unchecked:

```
Devices
  ├── Core       Read ✓   Write ✓
  └── Tags       tag:k8s ✓

Keys
  ├── Auth Keys  Read ✓   Write ✓
  └── Tags       tag:k8s ✓
```

4. Click **Generate** — copy the **Client ID** and **Client Secret**
   (secret is shown only once)

### 3c. Install the operator

```bash
helm upgrade --install tailscale-operator tailscale/tailscale-operator \
  --version 1.98.4 \
  --namespace tailscale \
  --create-namespace \
  --set-string oauth.clientId="<CLIENT_ID>" \
  --set-string oauth.clientSecret="<CLIENT_SECRET>" \
  --wait --timeout 5m
```

### 3d. Verify

```bash
kubectl get pods -n tailscale
# operator-xxxx   1/1   Running
```

---

## Step 4 — Install kube-prometheus-stack

```bash
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --version 87.2.1 \
  --namespace monitoring \
  --values values/prometheus-stack.yaml \
  --wait --timeout 10m
```

---

## Step 5 — Install Loki

```bash
helm upgrade --install loki grafana/loki \
  --version 7.0.0 \
  --namespace monitoring \
  --values values/loki.yaml \
  --wait --timeout 10m
```

---

## Step 6 — Install Tempo

```bash
helm upgrade --install tempo grafana/tempo \
  --version 1.24.4 \
  --namespace monitoring \
  --values values/tempo.yaml \
  --wait --timeout 5m
```

---

## Step 7 — Install Alloy (in-cluster k8s log collection)

```bash
helm upgrade --install alloy grafana/alloy \
  --version 1.10.0 \
  --namespace monitoring \
  --values values/alloy-cluster.yaml \
  --wait --timeout 5m
```

---

## Step 8 — Enable Tailscale HTTPS Certificates

In the **Tailscale admin panel → DNS** scroll down to **HTTPS Certificates**
and click **Enable**. This allows the Tailscale Kubernetes Operator to
automatically provision Let's Encrypt certificates for each Ingress hostname
(`grafana-homelab`, `prometheus-homelab`, etc.). Certificates are trusted by
all standard CA bundles — no custom CA configuration needed on external machines.

---

## Step 9 — Apply Tailscale Ingress

All four services (Grafana, Prometheus, Loki, Tempo) are exposed via
Kubernetes Ingress resources with `ingressClassName: tailscale`. The
Tailscale operator provisions a proxy pod per Ingress and terminates TLS
using the Let's Encrypt certificates enabled in Step 8.

All Helm-managed services are `ClusterIP` — the Ingress resources in
`tailscale-ingress.yaml` are the single source of external access and
are not managed by Helm, so they survive chart upgrades.

```bash
kubectl apply -f tailscale-ingress.yaml
```

Watch the operator provision proxy pods for all three:

```bash
kubectl get pods -n tailscale -w
# Expect: ts-prometheus-tailscale-*, ts-loki-tailscale-*, ts-tempo-tailscale-*
```

---

## HTTPS Certificate Renewal

Tailscale HTTPS certificates are issued by Let's Encrypt with a **90-day
validity period**. Renewal is fully automatic — each Tailscale proxy pod
monitors its own certificate and requests a renewal around day 60 while
the pods are running. No manual action is needed under normal operation.

**If a certificate expires** (e.g. proxy pods were down for an extended
period), you will see TLS errors in the browser or from Alloy agents.
Fix it by restarting the proxy pods so they reconnect to Tailscale and
get fresh certificates:

```bash
kubectl rollout restart statefulset -n tailscale
```

Or restart a specific proxy pod:

```bash
kubectl rollout restart statefulset/<pod-name> -n tailscale
```

A fresh certificate is provisioned automatically within a few seconds
of the pod coming back up.

---

## Step 10 — Verify All Four Tailscale Nodes Are Up

```bash
# 4 proxy pods + 1 operator = 5 pods total
kubectl get pods -n tailscale

# All four Ingress resources should have a Tailscale hostname in ADDRESS
kubectl get ingress -n monitoring
```

Expected output for the Ingress check:

```
NAME                 CLASS       HOSTS   ADDRESS                                PORTS     AGE
grafana-ingress      tailscale   *       grafana-homelab.<tailnet>.ts.net      80, 443
loki-ingress         tailscale   *       loki-homelab.<tailnet>.ts.net         80, 443
prometheus-ingress   tailscale   *       prometheus-homelab.<tailnet>.ts.net   80, 443
tempo-ingress        tailscale   *       tempo-homelab.<tailnet>.ts.net        80, 443
```

Also confirm in **Tailscale admin → Machines**:
- `grafana-homelab`
- `prometheus-homelab`
- `loki-homelab`
- `tempo-homelab`

---

## Step 11 — Retrieve Grafana Credentials

```bash
# Username: admin
kubectl get secret kube-prometheus-stack-grafana \
  -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 -d && echo
```

Open Grafana at **https://grafana-homelab.\<tailnet\>.ts.net** from any machine in your tailnet.

> Replace `<tailnet>` with your tailnet name (visible in the Ingress ADDRESS
> column from `kubectl get ingress -n monitoring`).

---

## Step 12 — Install Grafana Alloy on External Machines

All three env vars use Tailscale MagicDNS hostnames — they resolve
automatically on any machine in your tailnet.

| Variable | Value |
|---|---|
| `PROMETHEUS_URL` | `https://prometheus-homelab.<tailnet>.ts.net/api/v1/write` |
| `LOKI_URL` | `https://loki-homelab.<tailnet>.ts.net/loki/api/v1/push` |
| `TEMPO_URL` | `https://tempo-homelab.<tailnet>.ts.net` |

> `TEMPO_URL` is now an HTTPS base URL — Alloy's `otelcol.exporter.otlphttp`
> appends `/v1/traces` automatically. Traffic goes to Tempo's OTLP HTTP
> endpoint (port 4318) via the Tailscale Ingress on port 443.

---

### Linux (apt — Debian / Ubuntu)

```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] \
  https://apt.grafana.com stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update && sudo apt-get install -y alloy
sudo cp configs/alloy-linux-mac.alloy /etc/alloy/config.alloy

sudo systemctl edit alloy
```

Paste into the editor:

```ini
[Service]
Environment="PROMETHEUS_URL=https://prometheus-homelab.<tailnet>.ts.net/api/v1/write"
Environment="LOKI_URL=https://loki-homelab.<tailnet>.ts.net/loki/api/v1/push"
Environment="TEMPO_URL=https://tempo-homelab.<tailnet>.ts.net"
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now alloy
```

---

### Linux (rpm — RHEL / Fedora / Rocky)

```bash
cat << 'EOF' | sudo tee /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
EOF

sudo dnf install alloy -y
sudo cp configs/alloy-linux-mac.alloy /etc/alloy/config.alloy
sudo systemctl edit alloy    # same [Service] block as above
sudo systemctl enable --now alloy
```

---

### macOS (Homebrew)

```bash
brew install grafana/grafana/alloy

mkdir -p "$(brew --prefix)/etc/alloy"
cp configs/alloy-linux-mac.alloy "$(brew --prefix)/etc/alloy/config.alloy"

CFG="$(brew --prefix)/etc/alloy/config.alloy"
sed -i '' 's|env("PROMETHEUS_URL")|"https://prometheus-homelab.<tailnet>.ts.net/api/v1/write"|g' "$CFG"
sed -i '' 's|env("LOKI_URL")|"https://loki-homelab.<tailnet>.ts.net/loki/api/v1/push"|g'         "$CFG"
sed -i '' 's|env("TEMPO_URL")|"https://tempo-homelab.<tailnet>.ts.net"|g'                          "$CFG"

brew services start grafana/grafana/alloy
```

---

### Windows (PowerShell — run as Administrator)

```powershell
winget install GrafanaLabs.Alloy

Copy-Item configs\alloy-windows.alloy `
  "C:\Program Files\GrafanaLabs\Alloy\config.alloy" -Force

[System.Environment]::SetEnvironmentVariable(
  "PROMETHEUS_URL","https://prometheus-homelab.<tailnet>.ts.net/api/v1/write","Machine")
[System.Environment]::SetEnvironmentVariable(
  "LOKI_URL","https://loki-homelab.<tailnet>.ts.net/loki/api/v1/push","Machine")
[System.Environment]::SetEnvironmentVariable(
  "TEMPO_URL","https://tempo-homelab.<tailnet>.ts.net","Machine")

Restart-Service "Alloy"
```

---

## Step 13 — Import Grafana Dashboards

**Dashboards → Import → paste ID → Load:**

| Dashboard | ID |
|---|---|
| Node Exporter Full (Linux/Mac) | 1860 |
| Windows Exporter Full | 10467 |
| Kubernetes Cluster Overview | 7249 |
| Loki Overview | 13639 |
| Tempo Quickstart | 16611 |
| Alloy Runtime | 21226 |

---

## Verification

```bash
# All monitoring pods healthy — all should show Running
kubectl get pods -n monitoring

# Tailscale operator + 4 proxy pods = 5 pods total
kubectl get pods -n tailscale

# All four Ingresses should have a <tailnet>.ts.net address
kubectl get ingress -n monitoring
```

Test Grafana and Prometheus via their Tailscale HTTPS endpoints
(replace `<tailnet>` with your tailnet name):

```bash
# Grafana — should return HTTP 302 redirect to /login
curl -sI https://grafana-homelab.<tailnet>.ts.net

# Prometheus — should return "Prometheus Server is Ready."
curl -s https://prometheus-homelab.<tailnet>.ts.net/-/ready
```

Loki and Tempo health checks go directly to the pods — their Tailscale
Ingresses expose ingestion endpoints only (log push and OTLP traces),
not the internal HTTP API where `/ready` lives:

```bash
# Loki — should return "ready"
# Uses port-forward since the Loki container image does not include wget/curl
kubectl port-forward -n monitoring svc/loki 3100:3100 &
sleep 2   # wait for port-forward to be ready
curl -s http://localhost:3100/ready
kill %1

# Tempo — should return "ready"
kubectl exec -n monitoring tempo-0   -- wget -qO- http://localhost:3200/ready
```

```bash
# In-cluster Alloy — confirm k8s pod logs are being collected
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --tail=20
# Look for "opened log stream" lines — one per pod being collected
```

---