# LiveKit Secrets — Creation & Validation Guide

This document covers how to create and validate the two Kubernetes Secrets
required for a LiveKit deployment: one for livekit-server and one shared
by egress and SIP.

---

## Overview

LiveKit uses two different secret shapes because each service consumes
credentials differently:

```
Same API key + secret — different structures

livekit-api-keys                     livekit-agent-credentials
────────────────────────────         ─────────────────────────
keys.yaml: |                         api: <api-key>
  <api-key>: <api-secret>            api_secret: <api-secret>
       │                                  │
       │ volume mount → file              │ secretKeyRef → env var
       ▼                                  ▼
key_file: keys.yaml              LIVEKIT_API_KEY
ValidateKeys() reads it          LIVEKIT_API_SECRET
```

---

## Step 1 — Generate API key and secret

```bash
API_KEY=$(openssl rand -hex 16)
API_SECRET=$(openssl rand -hex 32)

echo "API Key    (public):  $API_KEY"
echo "API Secret (signing): $API_SECRET"

# Save both values somewhere safe (AWS Secrets Manager, Vault, .env file)
# before running the commands below — kubectl create does not let you
# read them back easily.
```

---

## Step 2 — Create the namespace

```bash
kubectl create namespace livekit
```

---

## Secret 1 — livekit-server

Used by: `livekit-server` only.
Consumed as: a file mounted at `key_file: keys.yaml` inside the container.
Required because: livekit-server reads a key map (`keyid: secret`), not individual env vars.

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

### What this renders to inside the container

```
Pod filesystem
└── keys.yaml (permissions: 0600)
    content: "<api-key>: <api-secret>"
          │
          ▼
livekit-server ValidateKeys():
  1. os.Stat("keys.yaml")         → file exists ✅
  2. st.Mode().Perm() == 0600     → permission check passes ✅
  3. yaml.Decode(f) → conf.Keys   → {"<api-key>": "<api-secret>"} ✅
  4. len(conf.Keys) > 0           → server starts ✅
```

### Helm values reference

```yaml
livekit:
  key_file: keys.yaml   # must match secret data key name
  keys: {}              # MUST be empty

storeKeysInSecret:
  enabled: true
  existingSecret: "livekit-api-keys"   # must match secret name above
  keys: {}                             # MUST be empty
```

---

## Secret 2 — egress and SIP (shared)

Used by: `livekit-egress`, `livekit-sip`, `livekit-ingress`.
Consumed as: individual env vars `LIVEKIT_API_KEY` and `LIVEKIT_API_SECRET`.
Required because: these services are clients of livekit-server and use the
standard LiveKit SDK credential pattern.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: livekit-agent-credentials
  namespace: livekit
type: Opaque
stringData:
  api: "${API_KEY}"
  api_secret: "${API_SECRET}"
EOF
```

### What this renders to inside the container

```
secretKeyRef → env var injection

  secret key "api"        → LIVEKIT_API_KEY=<api-key>
  secret key "api_secret" → LIVEKIT_API_SECRET=<api-secret>
```

### Helm values reference (egress and SIP)

```yaml
livekitCredentials:
  secretName: livekit-agent-credentials   # must match secret name above
  apiKey: api                             # must match secret data key
  apiSecretKey: api_secret                # must match secret data key
```

---

## Step 3 — Verify both secrets

### Verify livekit-server secret

```bash
# Check key name is exactly "keys.yaml"
kubectl get secret livekit-api-keys -n livekit \
  -o jsonpath='{.data}' | python3 -m json.tool

# Decode and read the file content
kubectl get secret livekit-api-keys -n livekit \
  -o jsonpath='{.data.keys\.yaml}' | base64 -d
```

Expected output:
```
<api-key>: <api-secret>
```

### Verify egress/SIP secret

```bash
# Decode api key
kubectl get secret livekit-agent-credentials -n livekit \
  -o jsonpath='{.data.api}' | base64 -d

# Decode api secret
kubectl get secret livekit-agent-credentials -n livekit \
  -o jsonpath='{.data.api_secret}' | base64 -d
```

---

## Step 4 — Validate the file mount (optional, Killercoda / any cluster)

Create a test pod that mimics the exact volume mount the livekit-server
Helm chart produces:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: livekit-secret-test
  namespace: livekit
spec:
  containers:
    - name: test
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: keys-volume
          mountPath: keys.yaml    # same as key_file
          subPath: keys.yaml      # same as secret data key name
  volumes:
    - name: keys-volume
      secret:
        secretName: livekit-api-keys
        defaultMode: 0600         # livekit-server requires exactly 0600
EOF

# Wait for pod to be ready
kubectl wait pod livekit-secret-test -n livekit \
  --for=condition=Ready --timeout=30s

# Check file exists at correct path
kubectl exec -n livekit livekit-secret-test -- ls -la keys.yaml

# Check permissions are exactly 0600
kubectl exec -n livekit livekit-secret-test -- stat keys.yaml

# Read file content
kubectl exec -n livekit livekit-secret-test -- cat keys.yaml
```

Expected outputs:

```bash
# ls -la
-rw------- 1 root root 52 ... keys.yaml    # 0600 confirmed

# cat keys.yaml
<api-key>: <api-secret>
```

### Cleanup test pod

```bash
kubectl delete pod livekit-secret-test -n livekit
```

---

## Common mistakes

| Mistake | Result |
|---|---|
| Secret data key is not `keys.yaml` | `subPath` finds nothing, file never appears, server fails with `ErrKeysNotSet` |
| `livekit.keys` is not empty | credentials leak into ConfigMap AND are mounted from secret |
| `key_file` not set in values | livekit-server ignores the mounted file, fails with `ErrKeysNotSet` |
| Wrong format inside secret (`key=secret` instead of `key: secret`) | `yaml.Decode` fails to parse, server rejects all auth |
| `existingSecret` name does not match actual secret name | volume mount fails, pod stuck in `Pending` |
| `fsGroup` set in `podSecurityContext` | Kubernetes changes file permissions to `0640`, `ValidateKeys()` returns `ErrKeyFileIncorrectPermission` |
| Egress/SIP `apiKey` value doesn't match secret data key | `secretKeyRef` fails, pod fails to start with `secret not found` |
| API secret shorter than 32 chars | Server starts but logs a warning — not enforced in prod mode but should be fixed |

---

## Secret shape comparison

| Property | `livekit-api-keys` | `livekit-agent-credentials` |
|---|---|---|
| Used by | livekit-server | egress, SIP, ingress |
| Data keys | `keys.yaml` | `api`, `api_secret` |
| Consumed as | Volume mount → file | `secretKeyRef` → env vars |
| File format | `keyid: secret` (YAML map) | flat key/value |
| Permission enforced | `0600` required | N/A |
| Supports multiple key pairs | ✅ Yes (one per line) | ❌ No (single pair) |