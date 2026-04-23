# sip — LiveKit SIP Helm chart

This chart runs the official [`livekit/sip`](https://github.com/livekit/sip) image on Kubernetes. It lives in this repo as the **`sip`** chart (upstream LiveKit does not publish a SIP chart in [livekit-helm](https://github.com/livekit/livekit-helm) today).

## How this chart was created

1. **Source behavior** — The upstream project documents running SIP with **`SIP_CONFIG_BODY`**: a YAML string that matches the service’s config file format (same idea as `docker-compose.yaml` in the SIP repo, which sets `SIP_CONFIG_BODY` under `environment`).
2. **Kubernetes mapping** — Non-secret settings under `sip:` in `values.yaml` are rendered into a **ConfigMap** key `config.yaml`. The Deployment sets **`SIP_CONFIG_BODY`** from that key via **`configMapKeyRef`**, so the pod sees the same config shape as Docker Compose.
3. **Networking** — Compose uses `network_mode: host` because SIP (**5060**) and RTP (**10000–20000**) are painful to NAT through Docker. On Kubernetes this chart uses **`hostNetwork: true`** and **`dnsPolicy: ClusterFirstWithHostNet`** so the process binds ports on the node and can still resolve in-cluster DNS (for `ws_url` and Redis).
4. **Extras** — Optional **ClusterIP Service** for `health_port` / `prometheus_port`, **affinity** for Karpenter-style node labels, and **pod anti-affinity** so only one SIP pod runs per node (port conflicts).

See `reference-docker-compose.yaml` in this directory for the upstream compose file this pattern is based on.

---

## How configuration and environment variables are passed

The SIP process is configured in two ways that match the **official GitHub documentation** (file or env substitutes for YAML fields):

| Mechanism | What it supplies |
|-----------|------------------|
| **`SIP_CONFIG_BODY`** | YAML body (from ConfigMap): `ws_url`, `redis`, ports, logging, etc. Credentials **must not** appear here in this chart’s default design. |
| **Environment variables** | `LIVEKIT_API_KEY` and `LIVEKIT_API_SECRET` (from a Kubernetes **Secret** via `secretKeyRef`) replace `api_key` and `api_secret` in YAML. |

Optional upstream env vars you can add yourself (not templated by default) include **`LIVEKIT_WS_URL`**, which can replace `ws_url` in YAML the same way—see the reference block below.

In the Deployment, order is: inject **`LIVEKIT_*`** from the Secret, then inject **`SIP_CONFIG_BODY`** from the ConfigMap. The application merges these according to [livekit/sip](https://github.com/livekit/sip) behavior.

---

## Official config reference (from LiveKit SIP)

The SIP service accepts a YAML config (file, or body via `SIP_CONFIG_BODY`). From the upstream project:

```yaml
# required fields
api_key: livekit server api key. LIVEKIT_API_KEY env can be used instead
api_secret: livekit server api secret. LIVEKIT_API_SECRET env can be used instead
ws_url: livekit server websocket url. LIVEKIT_WS_URL env can be used instead
redis:
  address: must be the same redis address used by your livekit server
  username: redis username
  password: redis password
  db: redis db

# optional fields
health_port: if used, will open an http port for health checks
prometheus_port: port used to collect prometheus metrics. Used for autoscaling
log_level: debug, info, warn, or error (default info)
sip_port: port to listen and send SIP traffic (default 5060)
rtp_port: port to listen and send RTP traffic (default 10000-20000)
```

This chart’s `values.yaml` under **`sip:`** mirrors the pieces you want in that YAML **except** `api_key` and `api_secret`, which are supplied via env instead.

---

## Why `api_key` and `api_secret` are not in the ConfigMap

**They are still credentials; they are just not stored in the ConfigMap.**

- **ConfigMaps are usually not treated as secrets.** They are often logged, cached in GitOps repos, visible to anyone who can read ConfigMaps, and show up in plain form in many debugging paths. Putting API keys and signing secrets there duplicates the risk of committing them in `values.yaml`.
- **Kubernetes Secrets** are the right object for `secretKeyRef`: RBAC can be tighter, encryption at rest can be enabled, and operators like External Secrets / Sealed Secrets integrate cleanly.
- **Upstream SIP explicitly allows env substitution:** `LIVEKIT_API_KEY` and `LIVEKIT_API_SECRET` are documented replacements for `api_key` / `api_secret`. This chart sets those env vars from your Secret keys (configurable names under `livekitCredentials`), so the running process receives the same logical config without ever serializing those values into the ConfigMap.

So: **credentials go Secret → pod env (`LIVEKIT_*`); everything else goes values → ConfigMap → `SIP_CONFIG_BODY`.**

---

## Prerequisites

- Same **Redis** as your LiveKit server (`redis.address` in `sip:`).
- **WebSocket URL** to the server (`ws_url`), typically in-cluster, e.g. `ws://livekit-server.<namespace>.svc.cluster.local:7880`.
- A **Kubernetes Secret** with your LiveKit API key id and signing secret (same pair you use for the server).

Create the Secret (example key names match default `livekitCredentials`):

```bash
kubectl create secret generic livekit-sip-credentials -n livekit \
  --from-literal=api='<your-api-key-id>' \
  --from-literal=api_secret='<your-signing-secret>'
```

Adjust `livekitCredentials.apiKeySecretKey` / `apiSecretSecretKey` in `values.yaml` if your Secret uses different data keys.

---

## Install

```bash
helm upgrade --install lk-sip ./sip -n livekit --create-namespace -f values.yaml
```

After install, configure **SIP trunks and dispatch rules** with the LiveKit API or [`livekit-cli`](https://github.com/livekit/livekit-cli)—this chart only deploys the SIP worker.

---

## Security groups and networking

SIP must be reachable on **`sip_port`** (UDP/TCP) and the **RTP** range from your carrier or peers. With **hostNetwork**, those ports bind on the **node**; open the node (or load balancer targeting the node) accordingly, same idea as the upstream Docker host-network documentation.

---

## Links

- [LiveKit SIP repository](https://github.com/livekit/sip)
- [SIP quickstart / trunks (LiveKit docs)](https://docs.livekit.io/sip/)
