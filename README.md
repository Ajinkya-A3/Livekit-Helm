# LiveKit Helm Charts

Collection of Helm charts for deploying the full LiveKit stack on Kubernetes (tested on AWS EKS with ARM64 node pools). This repository contains:

| Chart | Source | Purpose |
|---|---|---|
| `livekit-server` | Upstream (livekit/livekit-helm) | Core WebRTC media server |
| `egress` | Forked (secrets + AWS improvements) | Room recording and streaming |
| `sip` | **Custom (new chart)** | SIP/PSTN bridge for phone calls |
| `upstream-charts/ingress` | Upstream reference | RTMP/WHIP ingress (unmodified) |

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Repository Structure](#2-repository-structure)
3. [Prerequisites](#3-prerequisites)
4. [Kubernetes Secrets — Create Before Installing](#4-kubernetes-secrets--create-before-installing)
5. [livekit-server Chart](#5-livekit-server-chart)
6. [Egress Chart](#6-egress-chart)
7. [SIP Chart](#7-sip-chart)
8. [STUNner TURN Relay Setup](#8-stunner-turn-relay-setup)
9. [ICE Candidate Priorities](#9-ice-candidate-priorities)
10. [Deploy Order](#10-deploy-order)
11. [Upgrade and Rollback](#11-upgrade-and-rollback)
12. [Troubleshooting](#12-troubleshooting)
13. [Security Notes](#13-security-notes)
14. [Reference Links](#14-reference-links)

---

## 1. Architecture Overview

```
Internet
   │
   ├── WSS :443  ──► kgateway (Gateway API)  ──► livekit-server :7880
   │                 (TLS termination)             (WebSocket signaling)
   │
   ├── UDP :50000-60000 ──► node hostNetwork ──► livekit-server
   │   TCP :7881         (direct media)
   │
   ├── TURN-UDP :3478 ──► AWS NLB (passthrough) ──► STUNner pods ──► livekit-server
   ├── TURN-TLS :5349 ──► AWS NLB (passthrough) ──► STUNner pods ──► livekit-server
   │                      (cert-manager issues TLS cert, STUNner terminates it)
   │
   └── SIP :5060 / RTP :10000-20000 ──► SIP pod (hostNetwork) ──► livekit-server

All services talk to livekit-server via in-cluster WebSocket:
  ws://livekit-server.livekit.svc.cluster.local:7880

Shared Redis (same instance for all services):
  redis-master.redis.svc.cluster.local:6379
```

**Why `hostNetwork: true` on livekit-server and SIP?**
Both services bind large UDP port ranges that are impractical to map through kube-proxy NAT. With `hostNetwork`, pods bind ports directly on the node's network interface, matching the upstream Docker `network_mode: host` pattern. Side effect: only one pod per node (enforced via `podAntiAffinity`).

---

## 2. Repository Structure

```
Livekit-Helm/
├── README.md                          ← this file
├── Secrets.md                         ← full secrets creation and validation guide
│
├── livekit-server/
│   ├── Chart.yaml                     ← appVersion: v1.9.0
│   ├── values.yaml                    ← default values
│   ├── custom-values.yaml             ← EKS production example (use this)
│   ├── API_KEYS_SECRET.md             ← deep-dive on storeKeysInSecret
│   ├── STUNner.md                     ← full STUNner architecture and traffic flows
│   ├── stunner.yaml                   ← STUNner manifests (ClusterIssuer, Gateway, UDPRoutes…)
│   └── templates/
│       ├── deployment.yaml            ← hostNetwork, secret volume mount, TURN certs
│       ├── secret.yaml                ← creates secret only when existingSecret is not set
│       ├── configmap.yaml             ← LIVEKIT_CONFIG (no credentials)
│       ├── service.yaml               ← ClusterIP for in-cluster access
│       ├── ingress.yaml               ← optional Ingress (disabled when type: none)
│       ├── hpa.yaml                   ← HPA (disabled by default)
│       ├── servicemonitor.yaml        ← Prometheus ServiceMonitor
│       ├── backendconfig.yaml         ← GKE BackendConfig (36000s WS timeout)
│       ├── turnloadbalancer.yaml      ← optional TURN LoadBalancer
│       └── tests/test-connection.yaml
│
├── egress/
│   ├── Chart.yaml                     ← appVersion: v1.9.0
│   ├── values.yaml                    ← includes livekitCredentials block
│   ├── egress-sample-value.yaml       ← upstream reference (inline keys — not recommended)
│   ├── README.md                      ← egress fork changes explained
│   └── templates/
│       ├── deployment.yaml            ← secretKeyRef for LIVEKIT_API_KEY/SECRET
│       ├── configmap.yaml             ← EGRESS_CONFIG_BODY (no credentials)
│       ├── hpa.yaml
│       └── serviceaccount.yaml
│
├── sip/
│   ├── Chart.yaml                     ← appVersion: latest (custom chart)
│   ├── values.yaml                    ← hostNetwork, livekitCredentials, affinity
│   ├── reference-docker-compose.yaml  ← upstream compose this chart is based on
│   ├── README.md                      ← SIP chart design rationale
│   └── templates/
│       ├── deployment.yaml            ← hostNetwork, secretKeyRef, SIP_CONFIG_BODY
│       ├── configmap.yaml             ← SIP_CONFIG_BODY (no credentials)
│       ├── service.yaml               ← ClusterIP for health/metrics only
│       └── serviceaccount.yaml
│
└── upstream-charts/
    └── ingress/                       ← unmodified upstream chart (RTMP/WHIP)
```

---

## 3. Prerequisites

### Cluster requirements

- Kubernetes 1.25+
- **Node pool labeled `node-pool=ondemand-arm64`** — livekit-server and SIP pin to this pool (on-demand only; spot interruption drops all active calls)
- Security group rules on livekit-server / SIP nodes:
  - Inbound UDP `50000–60000` from `0.0.0.0/0` (WebRTC media)
  - Inbound TCP `7881` from `0.0.0.0/0` (WebRTC TCP fallback)
  - Inbound UDP/TCP `5060` and UDP `10000–20000` from `0.0.0.0/0` (SIP + RTP)

### Required cluster add-ons

| Add-on | Purpose |
|---|---|
| Redis (in-cluster or ElastiCache) | Shared pubsub for all LiveKit services |
| kgateway (Gateway API) + AWS NLB | TLS termination for WebSocket signaling (`wss://`) |
| cert-manager | Manages TLS certs for kgateway and STUNner |
| STUNner operator | Kubernetes-native TURN server for relay fallback |
| Helm 3.x | Chart installation |

### Install Helm repos

```bash
helm repo add livekit https://helm.livekit.io
helm repo add stunner https://l7mp.io/stunner
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

---

## 4. Kubernetes Secrets — Create Before Installing

All three charts externalize credentials into Kubernetes Secrets. **Create these before running any `helm install`.**

> See `Secrets.md` for complete creation, validation, and troubleshooting steps.

### Why two different secret shapes?

```
livekit-api-keys                       livekit-agent-credentials
────────────────────────────           ─────────────────────────
keys.yaml: |                           api: <api-key>
  <api-key>: <api-secret>              api_secret: <api-secret>
       │                                     │
       │ volumeMount subPath → file          │ secretKeyRef → env var
       ▼                                     ▼
livekit-server reads key_file         LIVEKIT_API_KEY
ValidateKeys() checks 0600 perm       LIVEKIT_API_SECRET
```

`livekit-server` reads a YAML key-map file (supports multiple key pairs, enforces `0600` permissions). Egress and SIP are SDK clients and consume individual env vars.

### Step 1 — Generate credentials

```bash
API_KEY=$(openssl rand -hex 16)
API_SECRET=$(openssl rand -hex 32)

echo "API Key    : $API_KEY"
echo "API Secret : $API_SECRET"
# Save these before running the next commands — they cannot be read back from kubectl easily
```

### Step 2 — Create namespace

```bash
kubectl create namespace livekit
```

### Step 3 — Secret for livekit-server

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: livekit-api-keys
  namespace: livekit
type: Opaque
stringData:
  keys.yaml: "${API_KEY}: ${API_SECRET}"
EOF
```

> **Critical:** The data key must be exactly `keys.yaml` — it must match `livekit.key_file` in values and the `subPath` in the volume mount. A mismatch means the file never appears in the container.
> **Critical:** Do not set `fsGroup` in `podSecurityContext` — it changes the mounted file permissions from `0600` to `0640`, causing `livekit-server` to refuse to start with `ErrKeyFileIncorrectPermission`.

### Step 4 — Shared secret for egress and SIP

Both services use the same secret shape. You can use one shared secret or create one per service.

```bash
# For egress
kubectl create secret generic livekit-egress-credentials -n livekit \
  --from-literal=api="${API_KEY}" \
  --from-literal=api_secret="${API_SECRET}"

# For SIP
kubectl create secret generic livekit-sip-credentials -n livekit \
  --from-literal=api="${API_KEY}" \
  --from-literal=api_secret="${API_SECRET}"
```

### Step 5 — Verify secrets

```bash
# livekit-server: check content and key name
kubectl get secret livekit-api-keys -n livekit \
  -o jsonpath='{.data.keys\.yaml}' | base64 -d
# Expected: <api-key>: <api-secret>

# egress / SIP: decode key and secret
kubectl get secret livekit-egress-credentials -n livekit \
  -o jsonpath='{.data.api}' | base64 -d
kubectl get secret livekit-egress-credentials -n livekit \
  -o jsonpath='{.data.api_secret}' | base64 -d
```

### Common mistakes

| Mistake | Symptom |
|---|---|
| Secret data key is not `keys.yaml` | Pod starts, fails immediately — `ErrKeysNotSet` |
| `livekit.keys` not empty | Credentials leak into ConfigMap **and** are mounted from secret |
| `fsGroup` set in `podSecurityContext` | Server refuses to start — `ErrKeyFileIncorrectPermission` |
| `existingSecret` name doesn't match actual secret name | Pod stuck in `Pending` — volume mount fails |
| Wrong format inside secret (`key=secret` vs `key: secret`) | Server rejects all auth — YAML parse error |
| API secret shorter than 32 chars | Server starts but logs a warning |

---

## 5. livekit-server Chart

### Key design decisions

**STUN for node public IP discovery (`use_external_ip: true`)**

On EKS (and any cloud), nodes have private IPs on their primary interface (e.g., `10.0.x.x`). LiveKit uses STUN from the **node itself** at startup to discover the node's real public IP, then advertises that IP as an ICE HOST candidate.

```yaml
# livekit-server/pkg/config/config.go (upstream)
# RTCConfig.UseExternalIP triggers an outbound STUN binding request
# from the server process. The mapped address in the response becomes
# the public IP announced to WebRTC clients in ICE candidates.
livekit:
  rtc:
    use_external_ip: true   # required on EKS — without this, clients
                            # receive a 10.x.x.x host candidate and fail
```

Without `use_external_ip`, clients outside the VPC receive a private IP as the HOST candidate, fail ICE, and fall back to STUNner relay — adding 5–15 seconds of connection delay to every session.

**STUN servers sent to clients (`stun_servers`)**

These are separate from the server's own `use_external_ip` STUN lookup. They are pushed to **WebRTC clients** so clients can gather server-reflexive (SRFLX) candidates — i.e. discover their own public IP behind NAT.

```yaml
livekit:
  rtc:
    stun_servers:
      - "stun:stun.l.google.com:19302"
      - "stun:stun1.l.google.com:19302"
      - "stun:stun.cloudflare.com:3478"
```

**API keys stored in a Kubernetes Secret (not ConfigMap)**

```yaml
livekit:
  key_file: keys.yaml   # server reads this file at startup
  keys: {}              # MUST stay empty — prevents leaking into ConfigMap

storeKeysInSecret:
  enabled: true
  existingSecret: "livekit-api-keys"   # pre-created secret
  keys: {}                             # MUST stay empty
```

The Deployment mounts the secret at `keys.yaml` with `defaultMode: 0600`.

**hostNetwork and anti-affinity**

```yaml
podHostNetwork: true   # binds UDP/TCP ports directly on the node

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-pool
              operator: In
              values: ["ondemand-arm64"]    # hard pin to on-demand — spot interruption kills all calls

  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: kubernetes.io/hostname   # one pod per node — port conflict protection
```

**LoadBalancer and TURN**

```yaml
loadBalancer:
  type: none             # kgateway handles signaling via HTTPRoute (Gateway API)

turnLoadbalancer:
  enable: false          # STUNner operator creates its own LoadBalancer Service
```

### Install

```bash
kubectl create namespace livekit  # if not already created

helm install livekit livekit/livekit-server \
  --namespace livekit \
  --values livekit-server/custom-values.yaml
```

### Key values reference

| Path | Default | Description |
|---|---|---|
| `livekit.port` | `7880` | WebSocket signaling port |
| `livekit.rtc.tcp_port` | `7881` | WebRTC TCP fallback |
| `livekit.rtc.port_range_start/end` | `50000–60000` | UDP media port range |
| `livekit.rtc.use_external_ip` | `true` | STUN lookup for node public IP |
| `livekit.rtc.stun_servers` | (empty) | STUN servers pushed to clients |
| `livekit.rtc.turn_servers` | (empty) | TURN relay pushed to clients (STUNner) |
| `livekit.redis.address` | (empty) | Redis address — required |
| `livekit.key_file` | (empty) | Set to `keys.yaml` to use secret mount |
| `storeKeysInSecret.enabled` | `false` | Enable secret-based key loading |
| `storeKeysInSecret.existingSecret` | `""` | Pre-created secret name |
| `podHostNetwork` | `true` | Bind ports on node network interface |
| `terminationGracePeriodSeconds` | `18000` | 5h — drains in-flight WebRTC sessions |

### After install — create the HTTPRoute (kgateway)

The chart does not create a Gateway API HTTPRoute. Create it separately:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: livekit
  namespace: livekit
spec:
  parentRefs:
    - name: <your-gateway>
      namespace: <gateway-namespace>
  hostnames:
    - livekit.example.com
  rules:
    - backendRefs:
        - name: livekit-server
          port: 7880
```

---

## 6. Egress Chart

Egress records and streams LiveKit rooms. This chart is forked from upstream `livekit/livekit-helm` with two changes:

1. **Secure secret handling** — `api_key` and `api_secret` are removed from the ConfigMap when `livekitCredentials.secretName` is set, and injected as `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` via `secretKeyRef` instead.
2. **AWS S3 via IRSA** — S3 storage config only sets `region` and `bucket` by default; `access_key` and `secret` are intentionally absent. The AWS SDK reads credentials from the pod's IRSA web identity token automatically.

### How credentials flow

```
Kubernetes Secret (livekit-egress-credentials)
  data.api        →  LIVEKIT_API_KEY     (secretKeyRef)
  data.api_secret →  LIVEKIT_API_SECRET  (secretKeyRef)

ConfigMap (config.yaml / EGRESS_CONFIG_BODY)
  ws_url, redis, storage.s3 (region + bucket only), ports, logging
  — api_key and api_secret are stripped from this YAML when secretName is set
```

### Install

```bash
helm upgrade --install lk-egress ./egress \
  --namespace livekit --create-namespace \
  --values egress/values.yaml
```

### Key values reference

| Path | Default | Description |
|---|---|---|
| `livekitCredentials.secretName` | `livekit-egress-credentials` | Secret holding API credentials |
| `livekitCredentials.apiKey` | `api` | Data key for API key ID |
| `livekitCredentials.apiSecretKey` | `api_secret` | Data key for API secret |
| `egress.ws_url` | in-cluster address | WebSocket URL to livekit-server |
| `egress.redis.address` | in-cluster Redis | Same Redis as livekit-server |
| `egress.storage.s3.region` | `us-east-1` | S3 region |
| `egress.storage.s3.bucket` | `your-bucket` | S3 bucket name |
| `egress.enable_chrome_sandbox` | `true` | Required for room composite recording |
| `terminationGracePeriodSeconds` | `3600` | 1h — allows recording jobs to finish |
| `autoscaling.enabled` | `false` | HPA based on CPU or `livekit_egress_available` metric |

### S3 with IRSA (recommended)

Do **not** set `access_key` / `secret` in `egress.storage.s3`. Instead:

1. Create an IAM role with `s3:PutObject` on your bucket
2. Annotate the pod's ServiceAccount with the IAM role ARN:
   ```yaml
   serviceAccount:
     create: true
     annotations:
       eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/egress-s3-role
   ```
3. The AWS SDK on the pod automatically uses the web identity token — no static credentials needed.

---

## 7. SIP Chart

This is a **custom chart** — upstream `livekit/livekit-helm` does not publish a SIP chart. It was created to match the `livekit/sip` Docker Compose pattern on Kubernetes.

### Design

| Docker Compose | Kubernetes equivalent |
|---|---|
| `network_mode: host` | `hostNetwork: true` + `dnsPolicy: ClusterFirstWithHostNet` |
| `SIP_CONFIG_BODY` env var | ConfigMap key `config.yaml` → `configMapKeyRef` |
| `api_key` / `api_secret` in YAML | Kubernetes Secret → `secretKeyRef` → `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` |
| Single container | Deployment with `replicaCount: 1` + hard pod anti-affinity |

**Why `hostNetwork`?** SIP binds on port `5060` (UDP+TCP) and RTP on `10000–20000`. These are large ephemeral port ranges that are impractical to map through kube-proxy. `hostNetwork` binds them directly on the node, identical to Docker's `network_mode: host`.

**Why `dnsPolicy: ClusterFirstWithHostNet`?** Without this, pods on the host network use the node's DNS resolver, which cannot resolve in-cluster names like `livekit-server.livekit.svc.cluster.local`.

### Credentials

```yaml
livekitCredentials:
  secretName: livekit-sip-credentials
  apiKey: api           # data key in secret
  apiSecretKey: api_secret

# SIP config body — no api_key/api_secret here
sip:
  ws_url: "ws://livekit-server.livekit.svc.cluster.local:7880"
  redis:
    address: redis-master.redis.svc.cluster.local:6379
  sip_port: 5060
  rtp_port: 10000-20000
  use_external_ip: true
```

### Node affinity and anti-affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-pool
              operator: In
              values: ["ondemand-arm64"]   # SIP mid-call drop = call lost permanently

  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: kubernetes.io/hostname  # one SIP pod per node — port conflict
```

### Install

```bash
# Create secret first
kubectl create secret generic livekit-sip-credentials -n livekit \
  --from-literal=api="${API_KEY}" \
  --from-literal=api_secret="${API_SECRET}"

# Install chart
helm upgrade --install lk-sip ./sip \
  --namespace livekit --create-namespace \
  --values sip/values.yaml
```

### After install

SIP trunks and dispatch rules are **not** created by this chart. Create them with the LiveKit CLI:

```bash
# Install livekit-cli
# https://github.com/livekit/livekit-cli

# Create an inbound trunk
lk sip inbound create trunk.json

# Create a dispatch rule
lk sip dispatch create rule.json
```

### Security groups

Open inbound on the SIP node (or NLB targeting the node):

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| `5060` | UDP + TCP | `0.0.0.0/0` | SIP signaling |
| `10000–20000` | UDP | `0.0.0.0/0` | RTP media |

---

## 8. STUNner TURN Relay Setup

STUNner is a Kubernetes-native TURN server. It handles WebRTC relay for clients behind strict corporate NATs that block direct UDP paths to LiveKit's media ports.

> See `livekit-server/STUNner.md` for the full architecture, traffic flows, cert-manager setup, and HMAC auth derivation.

### TLS model

STUNner terminates TLS itself. The AWS NLB is a **pure TCP passthrough** on both ports — no ACM cert, no TLS listener config needed.

```
Client → NLB :5349 TCP passthrough → STUNner :5349 (TLS termination)
                                      cert from cert-manager → stunner-tls-secret
```

### Apply STUNner manifests

Edit `livekit-server/stunner.yaml` — replace placeholders:

- `turn.example.com` → your TURN domain
- `your-email@example.com` → cert-manager Let's Encrypt email

```bash
kubectl apply -f livekit-server/stunner.yaml
```

This creates:
- `ClusterIssuer` (Let's Encrypt, DNS-01 via Cloudflare)
- `Certificate` → `stunner-tls-secret`
- `GatewayClass` (STUNner)
- `Gateway` with UDP `:3478` and TLS `:5349` listeners
- `UDPRoute` → livekit-server Service (the security boundary — STUNner only relays to declared backends)
- `GatewayConfig` (HMAC ephemeral auth)

### HMAC auth secret

```bash
HMAC_SECRET=$(openssl rand -hex 32)

kubectl create secret generic stunner-auth-secret \
  --namespace stunner \
  --from-literal=type=ephemeral \
  --from-literal=secret="${HMAC_SECRET}"

# Use the same HMAC_SECRET in custom-values.yaml:
# livekit.rtc.turn_servers[].secret: <HMAC_SECRET>
```

### Wire STUNner into livekit-server values

```yaml
livekit:
  rtc:
    turn_servers:
      - host: <stunner-nlb-hostname>        # kubectl get svc -n stunner → EXTERNAL-IP
        port: 3478
        protocol: udp
        secret: <HMAC_SECRET>
        ttl: 86400

      - host: <stunner-nlb-hostname>
        port: 5349
        protocol: tls
        secret: <HMAC_SECRET>
        ttl: 86400
```

### DNS

Point your TURN domain at the STUNner NLB:

```bash
kubectl get svc -n stunner   # find EXTERNAL-IP
# Create Route53 A Alias: turn.example.com → <nlb-hostname>
```

---

## 9. ICE Candidate Priorities

When a client joins a room, LiveKit sends it a prioritized list of ICE candidates:

```
Priority 1 — HOST
  LiveKit's real public IP (discovered via use_external_ip STUN at startup)
  Fast, no relay. Works for ~75-80% of clients.

Priority 2 — SRFLX (Server-Reflexive)
  Client's own public IP, discovered via stun_servers
  Works when both sides can punch through NAT. ~10-15% of clients.

Priority 3 — RELAY via STUNner TURN-UDP :3478
  Client relays media through STUNner over plain UDP.
  Works when direct + SRFLX fail. ~5-10% of clients.

Priority 4 — RELAY via STUNner TURN-TLS :5349
  Client relays over TLS/TCP — works through strict corporate firewalls.
  Last resort. ~1-5% of clients. Higher latency (TCP head-of-line blocking).
```

`use_external_ip: true` is what makes Priority 1 work. Without it, the HOST candidate is a private `10.x.x.x` address and most external clients fall through to TURN relay.

---

## 10. Deploy Order

```bash
# 1. Create namespace
kubectl create namespace livekit

# 2. Deploy Redis (if not already running)
helm upgrade --install redis bitnami/redis \
  --namespace redis --create-namespace \
  --set auth.enabled=false

# 3. Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# 4. Install STUNner operator
helm install stunner-gateway-operator stunner/stunner-gateway-operator \
  --namespace stunner-system --create-namespace

# 5. Create all secrets (see §4 above)
# ...

# 6. Apply STUNner manifests
kubectl apply -f livekit-server/stunner.yaml

# 7. Wait for certificate to be issued (~60-120s)
kubectl get certificate -n stunner --watch

# 8. Wait for STUNner NLB to provision, then add DNS record
kubectl get svc -n stunner --watch

# 9. Install livekit-server
helm upgrade --install livekit livekit/livekit-server \
  --namespace livekit \
  --values livekit-server/custom-values.yaml

# 10. Install egress
helm upgrade --install lk-egress ./egress \
  --namespace livekit \
  --values egress/values.yaml

# 11. Install SIP
helm upgrade --install lk-sip ./sip \
  --namespace livekit \
  --values sip/values.yaml

# 12. Create HTTPRoute for livekit-server (kgateway)
kubectl apply -f <your-httproute.yaml>

# 13. Configure SIP trunks and dispatch rules
lk sip inbound create trunk.json
```

---

## 11. Upgrade and Rollback

```bash
# Upgrade livekit-server
helm upgrade livekit livekit/livekit-server \
  --namespace livekit \
  --values livekit-server/custom-values.yaml

# Upgrade egress
helm upgrade lk-egress ./egress \
  --namespace livekit --values egress/values.yaml

# Upgrade SIP
helm upgrade lk-sip ./sip \
  --namespace livekit --values sip/values.yaml

# Rollback a chart
helm rollback livekit 1 --namespace livekit
```

**Note:** `terminationGracePeriodSeconds: 18000` on livekit-server (5 hours) and `3600` on egress/SIP (1 hour) mean rolling updates wait for in-flight sessions to drain. Schedule upgrades during low-traffic windows.

---

## 12. Troubleshooting

### livekit-server won't start

```bash
kubectl logs -n livekit deployment/livekit-server | grep -E "ERR|WARN|keys"

# Check secret mount
kubectl exec -n livekit <pod> -- ls -la keys.yaml
# Must show: -rw------- (0600)

kubectl exec -n livekit <pod> -- cat keys.yaml
# Must show: <api-key>: <api-secret>
```

### Clients can't connect (ICE failure)

```bash
# Verify server discovered its public IP
kubectl logs -n livekit <pod> | grep "external IP"

# Check STUN servers are reachable from the node
# (run on the node, not the pod)
dig stun.l.google.com
```

### TURN relay not working

```bash
# Check STUNner gateway is ready
kubectl get gateway -n stunner
kubectl get udproute -n stunner

# Check certificate
kubectl get certificate -n stunner
kubectl describe certificate stunner-tls -n stunner

# Test TURN-TLS reachability
openssl s_client -connect turn.example.com:5349 -servername turn.example.com
# Should show: certificate issued by Let's Encrypt

# Verify HMAC secrets match
kubectl get secret stunner-auth-secret -n stunner \
  -o jsonpath='{.data.secret}' | base64 -d
# Compare against livekit turn_servers[].secret in helm values

# Check STUNner logs
kubectl logs -n stunner -l app=stunner --tail=50
```

### 401 Unauthorized from STUNner

The HMAC secret in `stunner-auth-secret` and in `livekit.rtc.turn_servers[].secret` must be identical. LiveKit derives per-session TURN credentials from this secret; STUNner validates them using the same secret.

### Egress pod won't start

```bash
kubectl describe pod -n livekit -l app.kubernetes.io/name=egress
# Look for: secretKeyRef not found → secret name or key name mismatch

kubectl get secret livekit-egress-credentials -n livekit \
  -o jsonpath='{.data}' | python3 -m json.tool
# Must have keys: "api" and "api_secret"
```

### SIP pod won't start

```bash
kubectl describe pod -n livekit -l app.kubernetes.io/name=sip
# Look for: port conflict (another pod on same node with hostNetwork)
# or: secretKeyRef not found

# Verify anti-affinity is working (should show one pod per node)
kubectl get pods -n livekit -o wide | grep sip
```

### ConfigMap checksum annotation rolling unnecessarily

Both egress and SIP deployments have a `checksum/config` annotation that rolls the Deployment when the ConfigMap changes. If you update credentials only (no ConfigMap change), the rollout will not happen automatically — use `kubectl rollout restart` if needed.

---

## 13. Security Notes

- **Never set `livekit.keys` to a non-empty value.** Values flow into Helm release history, GitOps diffs, and ConfigMaps — all visible to anyone with `kubectl get cm` access.
- **Never set `fsGroup` in `podSecurityContext`** for livekit-server — it changes the key file permissions from `0600` to `0640` and the server refuses to start.
- **Egress and SIP ConfigMaps contain no credentials** when `livekitCredentials.secretName` is set. Verify with `kubectl get cm <name> -n livekit -o yaml` after install.
- **STUNner UDPRoutes are the security boundary** — STUNner will only relay to Services listed in a UDPRoute. Relay to arbitrary cluster services is not possible.
- **HMAC ephemeral credentials expire** after `ttl` seconds (default 86400 / 24h). Clients cannot reuse old credentials.
- **S3 access for egress should use IRSA** — no static `access_key` / `secret` in values or ConfigMaps.
- Consider enabling **Kubernetes secrets encryption at rest** (`--encryption-provider-config`) in your cluster to protect secrets stored in etcd.

---

## 14. Reference Links

| Resource | URL |
|---|---|
| LiveKit Server config reference | https://github.com/livekit/livekit/blob/master/pkg/config/config.go |
| LiveKit Server config sample | https://github.com/livekit/livekit/blob/master/config-sample.yaml |
| LiveKit SIP repository | https://github.com/livekit/sip |
| LiveKit Egress repository | https://github.com/livekit/egress |
| LiveKit CLI | https://github.com/livekit/livekit-cli |
| LiveKit SIP docs | https://docs.livekit.io/sip/ |
| STUNner operator | https://github.com/l7mp/stunner-helm |
| STUNner architecture | `livekit-server/STUNner.md` (this repo) |
| Secrets guide | `Secrets.md` (this repo) |
| API keys secret deep-dive | `livekit-server/API_KEYS_SECRET.md` (this repo) |
| Upstream livekit-helm | https://github.com/livekit/livekit-helm |
| cert-manager | https://cert-manager.io/docs/ |
| kgateway (Gateway API) | https://kgateway.dev |