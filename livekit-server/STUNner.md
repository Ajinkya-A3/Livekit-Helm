# STUNner — Architecture & Traffic Flow

## What STUNner Is

STUNner is a Kubernetes-native TURN server. It is the **last resort** relay
for WebRTC clients who cannot reach LiveKit directly — clients behind strict
corporate firewalls, symmetric NATs, or networks that block all UDP.

Without STUNner those clients fail completely. With STUNner they relay
their WebRTC media through it and reach LiveKit as if directly connected.

---

## TLS Termination Model — STUNner, Not NLB

In this setup STUNner terminates TLS itself. The NLB is a **pure
passthrough** on both ports — it does not inspect, decrypt, or modify any
traffic. There is no ACM certificate, no NLB TLS listener configuration.

```
Before (NLB terminates):          After (STUNner terminates — this setup):

Client ──TLS──► NLB (decrypts)    Client ──TLS──► NLB (passthrough)
         ──TCP──► STUNner                   ──TLS──► STUNner (decrypts)

ACM cert lives in: AWS            Cert lives in: Kubernetes Secret
Managed by: ACM auto-renew        Managed by: cert-manager auto-renew
NLB config: TLS listener + ARN   NLB config: plain TCP listener only
```

**Why this is cleaner:**
- All certs in one place — cert-manager manages both LiveKit (HTTPRoute)
  and STUNner (TURN-TLS) certs. No AWS console involvement.
- NLB config is simpler — just TCP passthrough, no TLS annotations.
- cert-manager rotates certs automatically. STUNner operator watches the
  Secret and reloads without restarting the TURN listener.

---

## Component Map

```
┌──────────────────────────────────────────────────────────────────────┐
│  AWS EKS Cluster                                                     │
│                                                                      │
│  ┌─────────────┐   ┌────────────────────────┐    ┌────────────────┐  │
│  │  kgateway   │   │       STUNner          │    │  LiveKit       │  │
│  │ (HTTPRoute) │   │  (TURN relay)          │    │  Server        │  │
│  │             │   │                        │    │                │  │
│  │  :7880 wss  │   │  :3478 TURN-UDP        │    │  :7880         │  │
│  │             │   │  :5349 TURN-TLS ◄──────────►│  :7881 TCP     │  │
│  └─────────────┘   │    (cert-manager cert) │    │  :50000-60000  │  │
│        ▲           └────────────────────────┘    └────────────────┘  │
│        │                    ▲  UDPRoute livekit-udp                  │
│        │                    │  UDPRoute livekit-tls                  │
│        │                    │                                        │
│  ┌─────────────┐   ┌────────────────────────┐                        │
│  │  AWS NLB    │   │  AWS NLB               │                        │
│  │ (kgateway)  │   │  (STUNner)             │                        │
│  │  :443 HTTPS │   │  :3478 UDP passthrough │                        │
│  │             │   │  :5349 TCP passthrough │ ← no TLS here          │
│  └─────────────┘   └────────────────────────┘                        │
└──────────────────────────────────────────────────────────────────────┘
         ▲                    ▲
         │                    │
  livekit.dev.example.com   turn.example.com
  (WebSocket signaling)     (TURN relay)
```

---

## cert-manager Certificate Flow

```
ClusterIssuer (letsencrypt-prod)
  │
  │  ACME DNS-01 challenge
  │  cert-manager creates TXT record in Route53
  │  Let's Encrypt validates domain ownership
  │
  ▼
Certificate resource (stunner namespace)
  spec.secretName: stunner-tls-secret
  spec.dnsNames:  [turn.example.com]
  │
  │  cert-manager issues cert and writes:
  ▼
Secret: stunner-tls-secret (stunner namespace)
  type: kubernetes.io/tls
  data:
    tls.crt: <PEM cert chain — leaf + intermediates>
    tls.key: <PEM private key>
  │
  │  STUNner operator watches this Secret
  ▼
STUNner TURN-TLS listener
  loads tls.crt + tls.key at startup
  reloads automatically on Secret change (cert rotation)
  performs TLS handshake directly with each client
```

**Why DNS-01 and not HTTP-01:**
Let's Encrypt HTTP-01 requires port 80 to be reachable on the domain being
validated. Port 80 is not relevant to a TURN server. DNS-01 validates by
creating a TXT record in Cloudflare instead — no port dependency at all.

---

## Ephemeral Auth Flow (HMAC-SHA1)

Before either traffic path can work, clients need time-limited credentials.

```
stunner-auth-secret (stunner namespace)
  type: ephemeral
  secret: abc123hmac...
       │
       │ authRef
       ▼
STUNner GatewayConfig         LiveKit (same HMAC secret via deploy.sh)
  operator reads secret key         │
       │                            │ When user joins room:
       │                            ▼
       │                   LiveKit computes per-session creds:
       │                     username   = "<unix_expiry>:<user_id>"
       │                     credential = base64(HMAC-SHA1(secret, username))
       │                   Sends to client in WebSocket ICE config
       │
       │ When client sends TURN Allocate:
       ▼
STUNner validates:
  1. Parse username → expiry timestamp + user_id
  2. Compute: expected = base64(HMAC-SHA1(secret, username))
  3. Compare expected == received credential
  4. Check: expiry timestamp > now
  5. ALLOW relay allocation   OR   REJECT with 401
```

Every client gets unique, time-limited credentials. The raw HMAC secret
never leaves your backend. Credentials expire automatically after TTL
(default 86400s / 24h) — no manual rotation needed.

---

## Traffic Path 1 — TURN-UDP (port 3478)

**Used by:** clients who can send UDP but whose NAT blocks direct paths
to LiveKit's media ports (50000-60000). Most home routers, mobile NATs.

```
WebRTC Client
│
│  Step 1 — ICE negotiation (WebSocket to LiveKit via kgateway)
│  LiveKit sends ICE server config:
│  {
│    urls:       ["turn:turn.example.com:3478?transport=udp"],
│    username:   "1714086400:user-xyz",     ← expiry:userid
│    credential: "EPddI2tMN9..."            ← HMAC-SHA1 derived
│  }
│
│  Step 2 — TURN Allocate Request (UDP)
│  Client → turn.example.com:3478
│
▼
AWS NLB — port 3478 UDP
│
│  Pure UDP passthrough. NLB does not inspect or modify UDP datagrams.
│  Forwards raw TURN-UDP packets directly to STUNner pods.
│  No TLS, no decryption, no modification.
│
▼
STUNner pod — TURN-UDP listener (port 3478)
│
│  1. Receives TURN Allocate request
│  2. Validates ephemeral credentials (HMAC-SHA1 check + expiry check)
│  3. On success: creates UDP relay allocation
│     Relay address: a port on STUNner's internal IP, e.g. 10.0.1.45:56789
│  4. Client sends media to relay address
│  5. STUNner forwards relayed packets to livekit-server Service
│
│  UDPRoute livekit-udp declares livekit-server as the ONLY allowed backend.
│  STUNner rejects relay to any Service not listed in a UDPRoute —
│  this is the security boundary.
│
▼
livekit-server Service → LiveKit pod
│
│  LiveKit receives UDP media from STUNner's relay address.
│  LiveKit is unaware the client is behind TURN — looks like any UDP stream.
│
▼
Return path: LiveKit → STUNner relay → NLB → Client (same UDP path)
```

**Protocol summary:**
```
Client          → NLB             :3478  UDP        (raw, no TLS)
NLB             → STUNner         :3478  UDP        (unchanged passthrough)
STUNner relay   → livekit-server  :svc   UDP        (relayed media)
```

---

## Traffic Path 2 — TURN-TLS (port 5349)

**Used by:** clients behind strict corporate firewalls that block all UDP
and non-standard TCP. These environments often allow only outbound port
443 (HTTPS) and sometimes port 5349 (IANA-registered for TURN over TLS).
This is the path that reaches every client — even the most restricted ones.

```
WebRTC Client
│
│  Step 1 — ICE negotiation (WebSocket to LiveKit via kgateway)
│  LiveKit sends ICE server config:
│  {
│    urls:       ["turns:turn.example.com:5349?transport=tcp"],
│                  ↑ "turns" = TURN over TLS (not "turn")
│    username:   "1714086400:user-xyz",
│    credential: "EPddI2tMN9..."
│  }
│
│  Step 2 — TCP connection to turn.example.com:5349
│  Client initiates TLS handshake
│
▼
AWS NLB — port 5349 TCP
│
│  Plain TCP passthrough. NLB does NOT terminate TLS.
│  No ACM cert. No TLS listener. No ssl-cert annotation.
│  The NLB simply forwards raw TCP bytes to STUNner on port 5349.
│  The TLS ClientHello travels through the NLB untouched.
│
▼
STUNner pod — TURN-TLS listener (port 5349)
│
│  STUNner performs the TLS handshake directly with the client:
│    - Client verifies cert: turn.example.com signed by Let's Encrypt
│    - TLS session established (TLS 1.2 or 1.3)
│    - The cert comes from stunner-tls-secret (tls.crt + tls.key)
│      which cert-manager issued and keeps current
│
│  After TLS handshake, STUNner receives the TURN Allocate request
│  inside the decrypted TCP stream:
│    1. Validates ephemeral credentials (identical to UDP path)
│    2. Creates TCP relay allocation
│    3. Relays media datagrams inside the TCP tunnel to livekit-server
│
│  UDPRoute livekit-tls declares livekit-server as the allowed backend.
│  Note: route type is UDPRoute even though transport is TLS/TCP.
│  "UDP" in UDPRoute = inner WebRTC media datagrams, not outer TURN transport.
│
▼
livekit-server Service → LiveKit pod
│
│  Identical to UDP path from LiveKit's perspective.
│  LiveKit sees a normal media stream regardless of TURN transport used.
│
▼
Return path: LiveKit → STUNner relay → NLB → Client (same TLS/TCP path)
```

**Protocol summary:**
```
Client          → NLB             :5349  TLS/TCP    (TLS ClientHello passes through)
NLB             → STUNner         :5349  TLS/TCP    (raw bytes, unchanged)
STUNner         → Client          handshake         (STUNner holds the cert)
STUNner relay   → livekit-server  :svc   UDP        (relayed media, now decrypted)
```

---

## Why UDPRoute for a TLS Listener

This is confusing but correct. The route type describes what is being
relayed, not the transport carrying it:

```
TURN protocol has two layers:

  Outer layer (TURN transport):
    TURN-UDP  → UDP datagrams between client and STUNner
    TURN-TLS  → TLS/TCP stream between client and STUNner

  Inner layer (relayed media):
    Always UDP datagrams — WebRTC media is always UDP regardless
    of how TURN tunnels it. RTP/RTCP are UDP protocols.

UDPRoute describes the INNER layer (what gets relayed).
Listener protocol describes the OUTER layer (how client connects).

So both routes relay the same thing — UDP media datagrams to livekit-server.
The only difference is how the client connects to STUNner to request the relay.
```

---

## ICE Candidate Priority

LiveKit pushes all of these to the client. The client tries them in order:

```
Priority 1 — HOST
  LiveKit's public IP:port advertised because use_external_ip: true
  makes LiveKit call STUN at startup to discover its real public IP.
  → Fast, no relay. Works for most clients.

Priority 2 — SRFLX (Server-Reflexive)
  Client's own public IP discovered via stun_servers config.
  Helps with symmetric NAT traversal.
  → Works when both sides can punch through NAT.

Priority 3 — RELAY via STUNner TURN-UDP :3478
  Client relays through STUNner over plain UDP.
  → Works when direct + SRFLX both fail.

Priority 4 — RELAY via STUNner TURN-TLS :5349
  Client relays through STUNner over TLS/TCP.
  → Last resort. Works in the strictest environments.
  → Higher latency than UDP relay (TCP head-of-line blocking).
  → But it works when everything else fails.

Approximate distribution:
  ~75-80%  connect via HOST (direct)
  ~10-15%  connect via SRFLX (NAT traversal)
  ~5-10%   use TURN-UDP relay
  ~1-5%    use TURN-TLS relay (corporate firewall)
  ~0%      no connection
```

---

## DNS Setup

Two records in Route53 (or your DNS provider):

### Record 1 — ACME DNS-01 validation (cert-manager handles automatically)

If cert-manager has Route53 IAM permissions, it creates and removes
the validation TXT record automatically. You do not touch this manually.

If IAM is not set up, cert-manager will tell you what TXT record to add:
```bash
kubectl describe certificate stunner-tls -n stunner
kubectl describe certificaterequest -n stunner
# Check Events for DNS challenge details
```

### Record 2 — Point turn.example.com at the STUNner NLB

```
Type:  A (Alias) — preferred in Route53
Name:  turn.example.com
Value: <stunner-nlb>.elb.us-east-1.amazonaws.com
TTL:   60
```

Get the NLB hostname after deploying:
```bash
kubectl get svc -n stunner
# EXTERNAL-IP column → NLB hostname
```

---

## cert-manager IAM Setup for DNS-01 (Route53)

cert-manager needs permission to create TXT records in your hosted zone.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/YOUR_HOSTED_ZONE_ID"
    }
  ]
}
```

Attach this policy to the cert-manager service account via IRSA:
```bash
eksctl create iamserviceaccount \
  --name cert-manager \
  --namespace cert-manager \
  --cluster YOUR_CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::ACCOUNT:policy/cert-manager-route53 \
  --approve
```

---

## Deploy Order

```bash
# 1. Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# 2. Set up IAM for Route53 DNS-01 (see above)

# 3. Create stunner namespace
kubectl create namespace stunner

# 4. Install STUNner operator
helm repo add stunner https://l7mp.io/stunner
helm repo update
helm install stunner-gateway-operator stunner/stunner-gateway-operator \
  --namespace stunner-system --create-namespace

# 5. Create HMAC auth secret
kubectl create secret generic stunner-auth-secret \
  --namespace stunner \
  --from-literal=type=ephemeral \
  --from-literal=secret="$(openssl rand -hex 32)" \
  --dry-run=client -o yaml | kubectl apply -f -

# 6. Apply stunner.yaml (ClusterIssuer + Certificate + everything else)
kubectl apply -f stunner.yaml

# 7. Wait for cert-manager to issue the certificate (~60-120s)
kubectl get certificate -n stunner --watch
# STATUS column should show: True

# 8. Verify Secret was created with correct keys
kubectl get secret stunner-tls-secret -n stunner \
  -o jsonpath='{.data}' | python3 -m json.tool
# Should show: tls.crt and tls.key keys

# 9. Wait for NLB to provision (~60-90s)
kubectl get svc -n stunner --watch
# EXTERNAL-IP column populates when ready

# 10. Add turn.example.com DNS record pointing to NLB hostname

# 11. Verify gateway is ready
kubectl get gateway -n stunner
kubectl get udproute -n stunner

# 12. Check STUNner logs
kubectl logs -n stunner -l app=stunner --tail=50

# 13. Test TURN connectivity
# Use https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/
# Add relay: turn:turn.example.com:3478 → should show Done + relay candidates
# Add relay: turns:turn.example.com:5349 → should show Done + relay candidates
```

---

## Troubleshooting

**Certificate not issuing:**
```bash
kubectl describe certificate stunner-tls -n stunner
kubectl describe certificaterequest -n stunner
kubectl describe challenge -n stunner
# Check Events for DNS-01 challenge failures
# Most common: IAM permissions, wrong hostedZoneID, wrong region
```

**TURN-TLS handshake failing (wrong cert / cert not found):**
```bash
# Verify Secret has the right keys
kubectl get secret stunner-tls-secret -n stunner -o yaml
# Must have: tls.crt and tls.key under data

# Check STUNner loaded the cert
kubectl logs -n stunner -l app=stunner | grep -i tls
kubectl logs -n stunner -l app=stunner | grep -i cert
```

**TURN-UDP works but TURN-TLS fails:**
```bash
# Verify listener is ready
kubectl get gateway stunner-gateway -n stunner -o yaml
# Check status.listeners — tls-listener should show Programmed: True

# Test raw TCP reaches STUNner (bypass TLS to check NLB passthrough)
openssl s_client -connect turn.example.com:5349 -servername turn.example.com
# Should show: Certificate for turn.example.com, issued by Let's Encrypt
```

**401 Unauthorized from STUNner:**
```bash
# Verify HMAC secret matches what LiveKit uses
kubectl get secret stunner-auth-secret -n stunner \
  -o jsonpath='{.data.secret}' | base64 -d

# Compare against LiveKit helm values
helm get values livekit -n livekit | grep secret
# Both must be identical
```

**Relay reaches STUNner but not LiveKit:**
```bash
kubectl describe udproute livekit-udp -n stunner
kubectl describe udproute livekit-tls -n stunner
# backendRefs.name must exactly match:
kubectl get svc -n livekit
```