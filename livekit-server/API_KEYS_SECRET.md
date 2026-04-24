# LiveKit Server — API Keys Secret Setup

This document explains how to securely store LiveKit API keys using `storeKeysInSecret`
instead of inlining them in `values.yaml`.

---

## Why

By default the chart puts API keys directly into a ConfigMap:

```yaml
# ❌ insecure — visible in helm history, GitOps diffs, kubectl describe
livekit:
  keys:
    myapikey: "myapisecret"
```

With `storeKeysInSecret` enabled, the ConfigMap contains only a file path reference.
The actual credentials live in a Kubernetes Secret, mounted as a file into the pod.

---

## How it works

The chart mounts the secret as a file at `key_file: keys.yaml` inside the container.
LiveKit server reads that file on startup instead of reading keys from the config.

```
K8s Secret "livekit-api-keys"
  └── data key: "keys.yaml"            ← must match key_file value
      data value: "<apikey>: <secret>" ← livekit key file format
           │
           │ volumeMount subPath: keys.yaml
           ▼
Pod filesystem: keys.yaml (permissions: 0600)
           │
           ▼
livekit-server reads key_file: keys.yaml ✅
```

> **Note:** The secret data key name (`keys.yaml`) must exactly match
> `.Values.livekit.key_file`. The chart uses `subPath` to project that specific
> entry as a file — if the name does not match, the file will not appear in the
> container and the server will fail to start.

---

## Step 1 — Generate API key and secret

```bash
API_KEY=$(openssl rand -hex 16)
API_SECRET=$(openssl rand -hex 32)

echo "API Key    (public):  $API_KEY"
echo "API Secret (signing): $API_SECRET"

# Save both values somewhere safe (AWS Secrets Manager, Vault, .env file)
# before running the next command — kubectl create does not let you read
# them back easily.
```

---

## Step 2 — Create the Kubernetes Secret

```bash
kubectl create secret generic livekit-api-keys \
  --namespace livekit \
  --from-literal=keys.yaml="${API_KEY}: ${API_SECRET}"
```

Or apply the equivalent YAML (replace values first):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: livekit-api-keys        # must match existingSecret in values
  namespace: livekit
type: Opaque
stringData:
  keys.yaml: |                  # key name MUST match key_file value
    <your-api-key>: <your-api-secret>
```

To add multiple key/secret pairs (e.g. separate keys per environment or service):

```yaml
stringData:
  keys.yaml: |
    abc123hex16apikey: secrethex32value_secrethex32value_longvalue
    def456hex16apikey: anothersecret32value_anothersecret32value
```

---

## Step 3 — Configure values.yaml

```yaml
livekit:
  key_file: keys.yaml   # tells livekit-server to read a file
  keys: {}              # MUST be empty — nothing should go into ConfigMap

storeKeysInSecret:
  enabled: true
  existingSecret: "livekit-api-keys"   # must match the secret name above
  keys: {}                             # MUST be empty — secret is pre-existing
```

> **Important:** If `livekit.keys` is not empty, those values will leak into
> the ConfigMap alongside the secret mount. Always keep both `keys: {}` fields empty.

See `custom-values.yaml` in this folder for a full EKS-oriented example that includes `key_file` and `storeKeysInSecret`.

---

## What the chart renders

With the above values the Deployment template produces:

```yaml
# volumeMount — projects the secret entry as a file
volumeMounts:
  - name: keys-volume
    mountPath: keys.yaml    # file path inside container
    subPath: keys.yaml      # entry name from the secret

# volume — references the secret
volumes:
  - name: keys-volume
    secret:
      secretName: livekit-api-keys
      defaultMode: 0600     # livekit-server requires exactly 0600
```

And the ConfigMap contains only a safe file path reference:

```yaml
# configmap — no credentials
data:
  config.yaml: |
    key_file: keys.yaml
    ...
```

---

## Step 4 — Verify after deploy

```bash
# Confirm the secret has the correct structure
kubectl get secret livekit-api-keys -n livekit \
  -o jsonpath='{.data.keys\.yaml}' | base64 -d

# Expected output:
# abc123hex16apikey: secrethex32value_secrethex32value_longvalue

# Confirm the file is mounted inside the pod
kubectl exec -n livekit <pod-name> -- cat keys.yaml

# Confirm file permissions are exactly 0600 (required by livekit-server)
kubectl exec -n livekit <pod-name> -- ls -la keys.yaml
# Expected: -rw------- 1 root root ... keys.yaml
```

---

## Common mistakes

| Mistake | Result |
|---|---|
| Secret data key is not `keys.yaml` | `subPath` finds nothing, file missing, server fails to start |
| `livekit.keys` is not empty | credentials leak into ConfigMap AND are mounted from secret |
| `key_file` not set in values | livekit-server ignores the mounted file entirely |
| Wrong format inside secret (`key=secret` instead of `key: secret`) | livekit fails to parse keys, rejects all auth |
| `existingSecret` name does not match actual secret name | volume mount fails, pod stuck in `Pending` |
| `fsGroup` set in `podSecurityContext` | Kubernetes changes file permissions to 0640, livekit-server refuses to start — do not set `fsGroup` |

---

## Security comparison

| | `livekit.keys` inline | `storeKeysInSecret` |
|---|---|---|
| Stored in | ConfigMap (plaintext) | K8s Secret (base64, encryptable at rest) |
| Visible in `helm history` | ✅ Yes — exposed | ❌ No |
| Visible in GitOps diffs | ✅ Yes — exposed | ❌ No |
| Visible in `kubectl describe pod` | ✅ Yes — in env | ❌ No |
| File permissions enforced | N/A | 0600 — required by livekit |
| Works with external secret managers | ❌ No | ✅ Yes (Vault, ESO, AWS Secrets Manager) |
