# LiveKit Server — EKS Dev Deployment README

## Stack Overview

```
Client
  │
  ├─ WebSocket (wss://) ──► kgateway HTTPRoute ──► livekit-server:7880
  │
  ├─ UDP media (direct) ──► Node public IP:50000-60000   (hostNetwork)
  ├─ TCP media (direct) ──► Node public IP:7881          (hostNetwork)
  │
  └─ TURN relay (fallback) ─► STUNner NLB:3478 (UDP)
                             ► STUNner NLB:5349 (TLS)
                                   │
                                   └─► UDPRoute ──► livekit-server Service
```

Components:
- **kgateway** — handles HTTPS + WebSocket ingress via Gateway API (HTTPRoute)
- **STUNner** — TURN relay fallback for clients behind strict NAT/firewall
- **Redis** — room state storage, pinned to ondemand-arm64 (stable node)
- **Karpenter NodePools** — 4 pools; LiveKit pinned hard to ondemand-arm64

---

## §1 — What Changed from the Original Template and Why

### `loadBalancer.type: none`
**Original:** `type: alb` — created an AWS ALB Ingress resource.  
**Changed to:** `none` — you are using kgateway with Gateway API (HTTPRoute). Creating
an ALB Ingress on top of kgateway would duplicate routing and conflict with your
HTTPRoute. The chart must not create any Ingress resource.

### `keys: {}` + `keySecret: livekit-api-keys`
**Original:** `keys: devkey: <dev-api-secret>` — inline plaintext in the values file.  
**Changed to:** Kubernetes Secret reference. Inline keys appear in:
- `helm history` output
- GitOps repository diffs (if you commit values)
- `helm get values` which any cluster user can run

The Secret is only readable by pods that mount it and by users with RBAC access.

### `affinity` — `requiredDuring` (hard) instead of `preferredDuring` (soft)
**Original:** `preferredDuringSchedulingIgnoredDuringExecution` — Karpenter would
*try* to use the preferred pools but could fall back to any available node under
pressure (including Spot nodes).  
**Changed to:** `requiredDuringSchedulingIgnoredDuringExecution` pinned to
`ondemand-arm64` only. Reason: a Spot interruption during a live WebRTC session
drops every participant on that node instantly with no graceful recovery.
`terminationGracePeriodSeconds: 18000` only helps during controlled Kubernetes
drains — AWS Spot reclaim gives only a 2-minute warning, not 5 hours.

### `turn_servers` — `username`/`credential` instead of `secret`
**Original template:** used `secret:` field (HMAC-SHA1 longterm credential).  
**Changed to:** `username:` + `credential:` fields because your STUNner GatewayConfig
uses `authType: plaintext`. Longterm HMAC and plaintext auth are different protocols.
Using the wrong field means LiveKit sends clients credentials in the wrong format
and they cannot authenticate with STUNner.

### `turn_servers` — port 5349 instead of 443
**Original template:** TLS listener on port 443.  
**Changed to:** 5349 to match your STUNner Gateway manifest which declares
`port: 5349` for the `TURN-TCP` listener. Mismatching this means TLS TURN
connections fail silently — clients get a relay candidate but it never connects.

### Removed OpenStack CCM annotation from GatewayConfig
Your cluster is EKS — the OpenStack Cloud Controller Manager annotation
(`loadbalancer.openstack.org/enable-ingress-hostname`) does nothing on AWS and
should be removed. AWS CCM assigns the NLB hostname automatically.

---

## §2 — Secrets: What Each Is, How to Generate, Where It Goes

### Secret 1: LiveKit API Key + Signing Secret (`livekit-api-keys`)

**What it is:**
- API Key (hex-16): the **public identifier** for your LiveKit deployment.
  Safe to reference in client config and logs.
- API Secret (hex-32): the **JWT signing secret**. Your backend uses this to
  sign access tokens that clients present when joining a room. Never expose this
  client-side or commit it to git.

**How the chart uses it:**
The chart mounts the Secret as a file. LiveKit reads each entry as
`key_name=signing_secret`. The key name in the Secret (`${API_KEY}`) becomes
the public API key ID; its value (`${API_SECRET}`) becomes the signing secret.

**Generate and create:**
```bash
API_KEY=$(openssl rand -hex 16)
API_SECRET=$(openssl rand -hex 32)

echo "API Key    (public):  $API_KEY"
echo "API Secret (signing): $API_SECRET"

# Save both values somewhere safe (AWS Secrets Manager, Vault, .env file)
# before running this — kubectl create does not let you read them back easily.

kubectl create secret generic livekit-api-keys \
  --namespace livekit \
  --from-literal="${API_KEY}=${API_SECRET}"
```

**Where it goes in values.yaml:**
```yaml
livekit:
  keys: {}
  keySecret: livekit-api-keys   # ← name of the Secret you just created
```

**Where your backend uses it:**
```js
// Node.js example (livekit-server-sdk)
const { AccessToken } = require('livekit-server-sdk')
const token = new AccessToken(API_KEY, API_SECRET, { identity: 'user-id' })
token.addGrant({ roomJoin: true, room: 'my-room' })
const jwt = token.toJwt()
```

---

### Secret 2: STUNner Credentials (`secret` / `ttl` in turn_servers)
 
## What it is
 
A single shared HMAC-SHA1 key stored in `stunner-auth-secret` in the
`stunner` namespace. LiveKit reads this key and derives unique, time-limited
`username` + `credential` pairs per session on the fly. Clients receive
those derived credentials — never the raw secret itself. STUNner
independently performs the same derivation to validate each TURN allocation.
 
**These are not static credentials you choose.**
There is no `GatewayConfig.spec.userName` or `spec.password` anymore.
The only thing that exists is one randomly generated HMAC key, shared
between STUNner and LiveKit.
 
---
 
## Generate and Create
 
```bash
kubectl create secret generic stunner-auth-secret \
  --namespace stunner \
  --from-literal=type=ephemeral \
  --from-literal=secret="$(openssl rand -hex 32)" \
  --dry-run=client -o yaml | kubectl apply -f -
```
 
The Secret has exactly two keys:
 
| Key      | Value                  | Purpose                                                              |
|----------|------------------------|----------------------------------------------------------------------|
| `type`   | `ephemeral`            | Literal string — tells STUNner operator which auth mode to activate  |
| `secret` | `openssl rand -hex 32` | Your HMAC-SHA1 signing key (64 hex characters)                       |
 
---
 
## How It Flows to Clients
 
LiveKit never sends the raw secret to clients. It derives per-session
credentials from it and sends those instead.
 
```
stunner-auth-secret.secret = "abc123hmac..."
         │
         │  LiveKit reads this at deploy time via deploy.sh --set
         ▼
When user joins a room, LiveKit computes:
 
  username   = "<unix_expiry_timestamp>:<user_id>"
             = "1714086400:user-xyz"
 
  credential = base64(HMAC-SHA1(secret, username))
             = "EPddI2tMN9vtfGMhup1RYE5nSkA="
 
Sent to WebRTC client in ICE config — derived values only, never raw secret:
  {
    urls:       "turns:turn.example.com:5349?transport=tcp",
    username:   "1714086400:user-xyz",
    credential: "EPddI2tMN9vtfGMhup1RYE5nSkA="
  }
 
STUNner receives TURN Allocate, independently computes:
  expected = base64(HMAC-SHA1(secret, received_username))
  if expected == received_credential AND timestamp > now → ALLOW
  else → 401 Unauthorized
```
 
---
 
## Where It Goes in values.yaml
 
```yaml
turn_servers:
  - host: turn.example.com      # your DNS name — must match the TLS cert SAN
    port: 3478
    protocol: udp
    secret: <your-hmac-secret>  # same value as stunner-auth-secret.secret
    ttl: 86400                  # credential lifetime in seconds — 24h
 
  - host: turn.example.com
    port: 5349
    protocol: tls
    secret: <your-hmac-secret>  # same secret, same derivation
    ttl: 86400
```
 
Keep `<your-hmac-secret>` as a placeholder in values.yaml — safe to commit
to git. The real value is injected at deploy time only (see below).
 
---
 
## How deploy.sh Reads It — No Hardcoding
 
```bash
# Read directly from the STUNner secret — one source of truth
STUNNER_HMAC=$(kubectl get secret stunner-auth-secret -n stunner \
  -o jsonpath='{.data.secret}' | base64 -d)
 
helm upgrade --install livekit livekit/livekit-server \
  --namespace livekit \
  --values values-dev.yaml \
  --set "livekit.rtc.turn_servers[0].host=turn.example.com" \
  --set "livekit.rtc.turn_servers[0].port=3478" \
  --set "livekit.rtc.turn_servers[0].protocol=udp" \
  --set "livekit.rtc.turn_servers[0].secret=${STUNNER_HMAC}" \
  --set "livekit.rtc.turn_servers[0].ttl=86400" \
  --set "livekit.rtc.turn_servers[1].host=turn.example.com" \
  --set "livekit.rtc.turn_servers[1].port=5349" \
  --set "livekit.rtc.turn_servers[1].protocol=tls" \
  --set "livekit.rtc.turn_servers[1].secret=${STUNNER_HMAC}" \
  --set "livekit.rtc.turn_servers[1].ttl=86400"
```
 
One source of truth: `stunner-auth-secret` in the `stunner` namespace.
No mirroring into the `livekit` namespace needed.
 
---
 
## Rotation
 
```bash
# 1. Rotate the HMAC secret in STUNner
kubectl create secret generic stunner-auth-secret \
  --namespace stunner \
  --from-literal=type=ephemeral \
  --from-literal=secret="$(openssl rand -hex 32)" \
  --dry-run=client -o yaml | kubectl apply -f -
 
# STUNner operator picks up the new Secret automatically — no restart.
# Existing sessions with old credentials remain valid until their TTL
# expires (up to 24h). STUNner accepts both old and new during the
# overlap window — zero dropped sessions during rotation.
 
# 2. Redeploy LiveKit so it derives new credentials with the rotated secret
./deploy.sh
```
 
---
 
## Comparison with Plaintext Auth
 
| | Plaintext (old) | Ephemeral HMAC (current) |
|---|---|---|
| Config fields | `username` + `credential` | `secret` + `ttl` |
| Sent to clients | Same static creds always | Unique per session |
| Credential expiry | Never — valid forever | Automatic at TTL (24h max) |
| If credential leaked | Valid until manual rotation | Expires within `ttl` seconds |
| Rotation impact | Redeploy everything, drop sessions | Zero dropped sessions (overlap window) |
| `values.yaml` fields | `username` / `credential` | `secret` / `ttl` |
| GatewayConfig | `userName` + `password` inline | `authRef` → Secret only |
| Secret in cluster | None (inline in GatewayConfig) | `stunner-auth-secret` (Opaque) |

---

### Secret 3: STUNner NLB hostname (`<stunner-nlb-hostname>`)

**What it is:**
The external hostname of the AWS Network Load Balancer that STUNner's operator
creates for the `stunner-gateway` Service. This is where clients send TURN
allocation requests.

**How to get it:**
```bash
# Deploy STUNner first, then wait ~60s for AWS to provision the NLB
kubectl get svc -n stunner

# Look for EXTERNAL-IP — it will be an AWS NLB hostname like:
# aabbcc1234def.elb.us-east-1.amazonaws.com
```

**Where it goes in values.yaml:**
```yaml
turn_servers:
  - host: aabbcc1234def.elb.us-east-1.amazonaws.com   # ← paste here (x2)
    ...
```

Also add it to stun_servers once confirmed reachable:
```yaml
stun_servers:
  - "stun:stun.l.google.com:19302"
  - "stun:stun1.l.google.com:19302"
  - "stun:stun.cloudflare.com:3478"
  - "stun:aabbcc1234def.elb.us-east-1.amazonaws.com:3478"   # STUNner STUN
```

---

### Secret 4: Redis address (`redis-master.redis.svc.cluster.local:6379`)

**What it is:**
The in-cluster DNS address of your Redis instance. LiveKit uses Redis to store
room state, participant metadata, and pub/sub for multi-node coordination.

**This is not secret** — it's a cluster-internal DNS name. If you add Redis auth,
create a separate secret for the password:
```bash
kubectl create secret generic livekit-redis \
  --namespace livekit \
  --from-literal=password=$(openssl rand -hex 16)
```
Then uncomment in values.yaml:
```yaml
redis:
  address: redis-master.redis.svc.cluster.local:6379
  passwordSecret: livekit-redis
```

**How to verify Redis is reachable from livekit namespace:**
```bash
kubectl run redis-test --rm -it --image=redis:alpine -n livekit \
  -- redis-cli -h redis-master.redis.svc.cluster.local ping
# Expected: PONG
```

---

## §3 — Node Affinity Explained

### Why ondemand-arm64 only (hard requirement)

LiveKit is pinned with `requiredDuringSchedulingIgnoredDuringExecution` to the
`ondemand-arm64` NodePool only. Here is why each pool was rejected:

**spot-arm64 / spot-amd64** — Rejected. AWS Spot instances can be reclaimed with
only a 2-minute warning. LiveKit has no Spot interruption handler. When the node
disappears, every WebRTC session on it drops instantly with no graceful recovery.
The `terminationGracePeriodSeconds: 18000` only works during controlled Kubernetes
drains (rolling updates, manual cordon+drain) — not Spot reclaim.

**ondemand-amd64** — Rejected for LiveKit specifically. Reserved for `workload-class:
critical` workloads. Also more expensive than ARM64 for the same workload profile.

**ondemand-arm64** — Correct choice. Your NodePool already labels these nodes
`workload-class: stable`, they are ARM-compatible (livekit/livekit-server
publishes multi-arch images including arm64), and Karpenter provisions them from
`m7g`/`r7g` Graviton instances which are ~20% cheaper than equivalent x86.

### Why podAntiAffinity is hard (required)

`podHostNetwork: true` means the pod binds ports directly on the node's network
interface. If two LiveKit pods land on the same node they both try to bind port
7880, 7881, and 50000-60000 — the second pod crashes. The hard anti-affinity rule
prevents the scheduler from ever placing two LiveKit pods on the same node.

---

## §4 — Why `use_external_ip: true` Is Required Even With STUNner

STUNner and `use_external_ip` solve completely different problems in the WebRTC
ICE negotiation process. You need both.

### How ICE candidate priority works

When a client connects to LiveKit, it gathers a list of every possible network
path to the server — called ICE candidates — and tries them in priority order:

```
Priority 1 (HOST)    → Direct IP:port of the server
Priority 2 (SRFLX)   → Server-reflexive: what STUN reveals about the client's public IP
Priority 3 (RELAY)   → TURN relay: what STUNner provides
```

The client always tries cheaper paths first and only falls back to relay if
direct connections fail.

### What `use_external_ip: false` (the broken case) looks like

```
LiveKit pod runs on EKS node with:
  Private IP:  10.0.1.45   (internal VPC)
  Public IP:   54.23.x.x   (what the internet sees)

Without use_external_ip:
  LiveKit advertises HOST candidate: 10.0.1.45:7881

Client outside VPC:
  Tries 10.0.1.45:7881 → UNREACHABLE (private IP, not routable)
  Tries SRFLX candidates → may work, may not depending on NAT
  Falls back to STUNner relay → eventually connects after 5-15s
```

Every single session goes through STUNner relay even when a direct path exists.
TURN relay adds latency, burns STUNner bandwidth, and can cap at lower bitrates.

### What `use_external_ip: true` (the correct case) looks like

```
At startup, LiveKit calls a STUN server FROM THE EKS NODE:
  STUN response → "your public IP is 54.23.x.x"

LiveKit advertises HOST candidate: 54.23.x.x:7881

Client outside VPC:
  Tries 54.23.x.x:7881 → CONNECTS DIRECTLY (public IP, port open in SG)
  STUNner is never involved for this client
```

### What each component actually does

```
┌─────────────────────────────────────────────────────────┐
│  use_external_ip: true                                  │
│  → LiveKit discovers its own public IP via STUN         │
│  → Advertises it as HOST candidate                      │
│  → Most clients connect directly, no relay              │
├─────────────────────────────────────────────────────────┤
│  stun_servers (in config)                               │
│  → Sent to CLIENTS so they discover their public IP     │
│  → Enables SRFLX candidates on the client side          │
│  → Unrelated to the server's own IP discovery           │
├─────────────────────────────────────────────────────────┤
│  STUNner (TURN relay)                                   │
│  → Last resort for clients behind strict firewall/NAT   │
│  → Used when both direct UDP and TCP paths fail         │
│  → ~5-20% of sessions in typical enterprise deployments │
└─────────────────────────────────────────────────────────┘
```

STUNner is essential but it is not a replacement for correct host candidates.
Without `use_external_ip: true`, you force 100% of your traffic through TURN
relay unnecessarily, which is exactly the outcome TURN is supposed to avoid.

---

## §5 — HTTPRoute for kgateway (create separately)

The chart does not create this — you must apply it manually after helm install:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: livekit-server
  namespace: livekit
spec:
  parentRefs:
    - name: <your-kgateway-gateway-name>   # kubectl get gateway -A
      namespace: <kgateway-namespace>
  hostnames:
    - livekit.dev.example.com              # must match your TLS cert SAN
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: livekit-server
          port: 7880
```

WebSocket upgrades (`Upgrade: websocket`) are forwarded automatically by
kgateway — no extra annotation is needed.

Apply with:
```bash
kubectl apply -f httproute-livekit.yaml
```

---

## §6 — Deploy Order

```bash
# 1. Create namespace
kubectl create namespace livekit

# 2. Generate and create API key secret
API_KEY=$(openssl rand -hex 16)
API_SECRET=$(openssl rand -hex 32)
echo "Save these: KEY=$API_KEY  SECRET=$API_SECRET"
kubectl create secret generic livekit-api-keys \
  --namespace livekit \
  --from-literal="${API_KEY}=${API_SECRET}"

# 3. Deploy Redis (in redis namespace, ondemand-arm64 node)
helm install redis bitnami/redis \
  --namespace redis --create-namespace \
  --values redis-values.yaml

# 4. Deploy STUNner, then get NLB hostname
kubectl get svc -n stunner
# Fill in <stunner-nlb-hostname> in values-dev.yaml

# 5. Deploy LiveKit
helm install livekit livekit/livekit-server \
  --namespace livekit \
  --values values-dev.yaml

# 6. Apply HTTPRoute for kgateway
kubectl apply -f httproute-livekit.yaml

# 7. Verify
kubectl get pods -n livekit
kubectl logs -n livekit -l app.kubernetes.io/name=livekit-server --tail=50
```