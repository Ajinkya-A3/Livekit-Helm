# LiveKit Egress Helm chart (this repo)

This directory is the **egress** chart from [livekit/livekit-helm](https://github.com/livekit/livekit-helm), with a small fork focused on **secrets** and **AWS best practices**:

- **LiveKit API key / secret** are not stored in the ConfigMap when you set `livekitCredentials.secretName`. They are injected as **`LIVEKIT_API_KEY`** and **`LIVEKIT_API_SECRET`** via `secretKeyRef`, matching the [Egress config docs](https://github.com/livekit/egress) (*`api_key` / `api_secret` in YAML, or those env vars instead*).
- **S3 uploads from EKS** should use **IAM Roles for Service Accounts (IRSA)** so you do **not** put `access_key` / `secret` in YAML or in Kubernetes Secrets for AWS. Leave them unset in `egress.storage.s3`; the AWS SDK uses the web identity token on the pod.

---

## Helm chart changes (vs upstream `livekit-helm`)

Upstream ships a single config blob: everything under `egress` in `values.yaml` is rendered into a ConfigMap and passed to the container as **`EGRESS_CONFIG_BODY`** only. That means API keys and secrets would normally sit in the ConfigMap (and in Helm release metadata if inlined in values).

This fork adjusts **`templates/configmap.yaml`**, **`templates/deployment.yaml`**, and **`values.yaml`** so LiveKit credentials can live in a **Kubernetes Secret** and still satisfy the Egress binary’s documented env contract.

### `templates/configmap.yaml`

- The ConfigMap still contains one key, **`config.yaml`**, which is the egress YAML body.
- **If `livekitCredentials.secretName` is set**, `api_key` and `api_secret` are **stripped** from that YAML before it is written (`omit` in the template). They must then be provided via environment variables (see below).
- **If `livekitCredentials.secretName` is empty** (legacy), the full `egress` map is serialized as today, including inline `api_key` / `api_secret` if you define them.

### `templates/deployment.yaml` — container `env`

The egress image accepts configuration from **`EGRESS_CONFIG_BODY`** (YAML string) and, per [Egress docs](https://github.com/livekit/egress), can take **`api_key` / `api_secret` from `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET`** instead of embedding them in that YAML.

This chart’s **Deployment** wires that as follows.

**1. Optional LiveKit credentials (only when `livekitCredentials.secretName` is non-empty)**

| Env var | Source | Purpose |
|---------|--------|--------|
| `LIVEKIT_API_KEY` | `secretKeyRef` → Secret named `livekitCredentials.secretName`, data key `livekitCredentials.apiKey` (default `api`) | Replaces `api_key` in YAML. |
| `LIVEKIT_API_SECRET` | Same Secret, key `livekitCredentials.apiSecretKey` (default `api_secret`) | Replaces `api_secret` in YAML. |

The process reads these at startup the same way it would read them from the YAML file; they are **not** duplicated inside `EGRESS_CONFIG_BODY` when the Secret path is used.

**2. Main config (always)**

| Env var | Source | Purpose |
|---------|--------|--------|
| `EGRESS_CONFIG_BODY` | `configMapKeyRef` → ConfigMap named like the release (`egress.fullname`), key **`config.yaml`** | Full egress YAML for `ws_url`, `redis`, `storage`, ports, logging, etc. |

So the container always gets the non-sensitive (or legacy) config as one env var holding multiline YAML, plus **up to two** extra env vars for the signing identity when you use the Secret-based flow.

**Pod template annotations**

- **`checksum/config`** is emitted under `spec.template.metadata.annotations` and is updated when the ConfigMap content changes, so the Deployment rolls when you change `egress:` values.
- **`podAnnotations`** from values are merged into the same `annotations` map (fixes a layout issue where a raw checksum could sit outside `annotations` in older templates).

**Not templated here**

- No **`LIVEKIT_WS_URL`** injection in this chart (set `ws_url` in `egress` / ConfigMap if you need it).
- No **`AWS_ACCESS_KEY_ID`** / **`AWS_SECRET_ACCESS_KEY`** (or similar) env blocks—S3 is expected to use **IRSA** or instance metadata; omit static keys from `storage.s3` in values.

### `values.yaml`

- Adds **`livekitCredentials`** (`secretName`, `apiKey`, `apiSecretKey`) to point at the Secret that backs `LIVEKIT_*`.
- Default **`egress.storage.s3`** only sets **`region`** and **`bucket`** so IRSA is the natural path.

---

## What lives where

| What | Where | Notes |
|------|--------|--------|
| S3 access | **IAM role** assumed via IRSA | No long‑lived access keys in the cluster for S3. |
| LiveKit `api_key` / `api_secret` | **Kubernetes Secret** → pod env | Chart maps to `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET`. |
| Everything else (`ws_url`, `redis`, `storage.s3` bucket/region, ports, …) | **ConfigMap** → `EGRESS_CONFIG_BODY` | Non-sensitive config only. |
| Pod identity for AWS | **ServiceAccount** with `eks.amazonaws.com/role-arn` | Created by you or `eksctl`; referenced in Helm `serviceAccount`. |

---

## 1. LiveKit credentials (Kubernetes Secret)

Create a Secret whose data keys match `livekitCredentials` in `values.yaml` (defaults: `api`, `api_secret`):

```bash
kubectl create secret generic livekit-egress-credentials -n livekit \
  --from-literal=api='<your-api-key-id>' \
  --from-literal=api_secret='<your-signing-secret>'
```

With `livekitCredentials.secretName` set, **`api_key` and `api_secret` are omitted** from the rendered egress YAML in the ConfigMap so they never appear next to non-secret settings.

**Legacy:** set `livekitCredentials.secretName` to `""` if you must inline `api_key` / `api_secret` under `egress` (not recommended; they will be stored in the ConfigMap).

---

## 2. S3 via IAM Roles for Service Accounts (IRSA)

For S3 uploads from EKS, the recommended approach is **IRSA**: no access keys stored anywhere. The Egress docs state that `access_key` / `secret` can be **empty** when using an **IAM role** or **instance profile** — leaving them out of config is intentional and correct.

### Step 1 — IAM policy for the egress bucket

Adjust bucket name and account as needed:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-egress-bucket",
        "arn:aws:s3:::my-egress-bucket/*"
      ]
    }
  ]
}
```

```bash
aws iam create-policy \
  --policy-name livekit-egress-s3 \
  --policy-document file://policy.json
```

### Step 2 — EKS OIDC provider (if not already associated)

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve
```

### Step 3 — IAM role + Kubernetes ServiceAccount

Using `eksctl` (creates the ServiceAccount and wires trust to the role):

```bash
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace livekit \
  --name livekit-egress \
  --attach-policy-arn arn:aws:iam::123456789012:policy/livekit-egress-s3 \
  --approve
```

The trust policy ties the role to that ServiceAccount, for example:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/OIDC_ID"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.REGION.amazonaws.com/id/OIDC_ID:sub": "system:serviceaccount:livekit:livekit-egress"
    }
  }
}
```

(Exact OIDC ARN and subject come from your cluster; `eksctl` generates the correct relationship.)

### Step 4 — Helm values (this chart)

Point the chart at the IRSA ServiceAccount. If **`eksctl`** created `livekit-egress` in namespace `livekit`, use:

```yaml
serviceAccount:
  create: false
  name: livekit-egress

livekitCredentials:
  secretName: livekit-egress-credentials
  apiKey: api
  apiSecretKey: api_secret

egress:
  ws_url: "ws://livekit-server.livekit.svc.cluster.local:7880"
  redis:
    address: redis-master.redis.svc.cluster.local:6379
  storage:
    s3:
      region: us-west-2
      bucket: my-egress-bucket
```

Do **not** set `access_key`, `secret`, or `session_token` under `storage.s3` when using IRSA. The default `values.yaml` in this chart already omits them; the AWS SDK picks up temporary credentials from the annotated ServiceAccount.

If you create the ServiceAccount yourself, set the role annotation on it:

```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/your-egress-role
```

You can instead set `serviceAccount.create: true` and pass that annotation under `serviceAccount.annotations` in Helm—use one approach, not both, for the same account name.

### Step 5 — Verify the ServiceAccount

```bash
kubectl get serviceaccount livekit-egress -n livekit -o yaml
```

You should see:

```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/...
```

---

## Summary

- **S3:** credentials come from **IRSA** (temporary STS credentials), not from static keys in ConfigMap or Secrets.
- **LiveKit API key / secret:** come from a **Kubernetes Secret** and **`LIVEKIT_*`** env vars, not from the egress ConfigMap (when `livekitCredentials.secretName` is set).
- **ConfigMap:** only **non-sensitive** egress YAML is mounted as **`EGRESS_CONFIG_BODY`**.

This keeps long‑lived AWS access keys out of the cluster while still protecting LiveKit signing secrets with normal Secret RBAC and rotation practices.

---

## References

- [livekit/egress](https://github.com/livekit/egress)
- [AWS: IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [livekit-helm egress chart (upstream)](https://github.com/livekit/livekit-helm/tree/master/egress)
