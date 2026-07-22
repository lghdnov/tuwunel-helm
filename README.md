# tuwunel Helm Chart

A Helm chart for deploying [Tuwunel](https://github.com/maunium/tuwunel) — a lightweight Matrix homeserver.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.12+
- cert-manager (for TLS)
- Traefik (or another ingress controller)

## Installation

```bash
helm repo add tuwunel https://lghdnov.github.io/tuwunel-helm
helm install my-tuwunel tuwunel/tuwunel -f values.yaml
```

Or from source:

```bash
helm dependency build .
helm install my-tuwunel . -f my-values.yaml
```

## Required Values

| Parameter | Description | Example |
|-----------|-------------|---------|
| `serverName` | Your Matrix server domain (the part after `@` in MXIDs) | `example.com` |

## Core Values

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image.repository` | `jevolk/tuwunel` | Container image |
| `image.tag` | `latest` | Image tag |
| `replicaCount` | `1` | Number of replicas |
| `databasePath` | `/data` | Path to SQLite database |
| `port` | `8008` | Tuwunel HTTP port |
| `persistence.enabled` | `true` | Enable persistent storage |
| `persistence.size` | `10Gi` | PVC size |
| `persistence.storageClass` | `local-path` | Storage class |
| `ingress.enabled` | `true` | Enable ingress |
| `ingress.className` | `traefik` | Ingress class |

## Federation & Well-Known

Set `wellKnown.enabled: true` to deploy a sidecar serving `.well-known/matrix/client` and `.well-known/matrix/server`.

```yaml
wellKnown:
  enabled: true
  className: traefik
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod

config:
  wellKnown:
    server: "matrix.example.com"
    client:
      homeserverUrl: "https://matrix.example.com"
      identityServerUrl: ""
```

## Registration Token

Enable token-based registration:

```yaml
config:
  allowRegistration: true
registrationToken: "my-secret-token"
```

Or use an existing Secret:

```yaml
registrationTokenSecret:
  enabled: true
  name: "tuwunel-registration-token"
  key: "token"
```

---

# MatrixRTC (Voice/Video Calling)

This chart includes optional support for [MatrixRTC](https://github.com/element-hq/matrix-rich-voip-msc) voice/video calling via a [LiveKit](https://livekit.io/) SFU and the Element `lk-jwt-service` authorization bridge.

> **Note:** Tuwunel itself does not support MSC4140 (delayed events) which is required for native MatrixRTC. This integration uses LiveKit as an external SFU and announces it via `.well-known` (MSC4143).

## Enabling MatrixRTC

```yaml
matrixRtc:
  enabled: true

  wellKnown:
    enabled: true
    foci:
      - type: "livekit"
        livekit_service_url: "https://matrix-rtc.example.com/livekit/jwt"

  authService:
    deploy: true
    env:
      LIVEKIT_URL: "wss://livekit.example.com"
      LIVEKIT_KEY: "my-livekit-key"
      LIVEKIT_SECRET: "my-livekit-secret"
      LIVEKIT_FULL_ACCESS_HOMESERVERS: "example.com"

    ingress:
      enabled: true
      className: "traefik"
      annotations:
        cert-manager.io/cluster-issuer: "letsencrypt-prod"
      host: "matrix-rtc.example.com"
      pathPrefix: "/livekit/jwt"
```

## MatrixRTC Value Reference

### `matrixRtc.enabled`
**Default:** `false`

Master switch for all MatrixRTC resources.

### `matrixRtc.wellKnown.enabled`
**Default:** `true`

When enabled, injects `org.matrix.msc4143.rtc_foci` into `.well-known/matrix/client` so compatible clients discover the LiveKit focus.

### `matrixRtc.wellKnown.foci`
**Default:** `[{ type: "livekit", livekit_service_url: "" }]`

List of RTC foci to announce. The `livekit_service_url` must be the public URL of the `lk-jwt-service` ingress.

### `matrixRtc.livekit`
Settings for the LiveKit SFU. The chart does **not** deploy LiveKit itself — point to an existing instance or add the [livekit-server Helm chart](https://github.com/livekit/livekit-helm) as a subchart.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `livekit.deploy` | `false` | Deploy LiveKit via subchart |
| `livekit.externalUrl` | `""` | WebSocket URL of external LiveKit (e.g. `wss://livekit.example.com`) |
| `livekit.externalApiUrl` | `""` | HTTP API URL of external LiveKit |
| `livekit.config.port` | `7880` | LiveKit signaling port |
| `livekit.config.rtc.udp_port` | `7882` | WebRTC UDP port |
| `livekit.config.rtc.tcp_port` | `7881` | WebRTC TCP fallback port |
| `livekit.config.rtc.use_external_ip` | `true` | Advertise external IP for NAT traversal |
| `livekit.config.rtc.port_range_start` | `50000` | Start of UDP media port range |
| `livekit.config.rtc.port_range_end` | `60000` | End of UDP media port range |
| `livekit.config.keys.devkey` | `"secret"` | API key for SFU authentication |
| `livekit.config.room.auto_create` | `false` | **Must be `false`** — rooms are created by `lk-jwt-service` |
| `livekit.hostNetwork` | `false` | Run LiveKit in host network mode (required for wide UDP ranges in some clusters) |
| `livekit.nodePorts.enabled` | `false` | Expose UDP ports via NodePort |

> **Warning:** `hostNetwork: true` exposes all host network interfaces to the pod, breaking network isolation. Use only on dedicated nodes.

### `matrixRtc.authService`
Deploys the Element `lk-jwt-service` — a bridge that validates Matrix OpenID tokens and issues LiveKit JWTs.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `authService.deploy` | `true` | Deploy the auth service |
| `authService.image.repository` | `ghcr.io/element-hq/lk-jwt-service` | Container image |
| `authService.image.tag` | `latest` | Image tag |
| `authService.port` | `8080` | Container port |
| `authService.env.LIVEKIT_URL` | `""` | LiveKit WebSocket URL |
| `authService.env.LIVEKIT_KEY` | `""` | LiveKit API key |
| `authService.env.LIVEKIT_SECRET` | `""` | LiveKit API secret |
| `authService.env.LIVEKIT_FULL_ACCESS_HOMESERVERS` | `""` | Comma-separated list of homeservers with full access |
| `authService.env.LIVEKIT_JWT_BIND` | `":8080"` | Bind address |
| `authService.env.LIVEKIT_SANITY_CHECK_INTERVAL_SECONDS` | `"0"` | Health check interval (0 = disabled) |
| `authService.env.LIVEKIT_LOG_LEVEL` | `"info"` | Log level |
| `authService.resources` | `limits: {cpu: 500m, memory: 256Mi}` | Resource limits |
| `authService.ingress.enabled` | `true` | Enable ingress for auth service |
| `authService.ingress.className` | `traefik` | Ingress class |
| `authService.ingress.host` | `""` | Ingress host |
| `authService.ingress.pathPrefix` | `/livekit/jwt` | URL path for JWT endpoint |

## MatrixRTC Quick-Start (Copy-Paste)

### 1. External LiveKit (recommended for production)

```yaml
matrixRtc:
  enabled: true
  wellKnown:
    enabled: true
    foci:
      - type: livekit
        livekit_service_url: "https://rtc.example.com/livekit/jwt"
  livekit:
    deploy: false
    externalUrl: "wss://livekit.example.com"
    externalApiUrl: "https://livekit.example.com"
  authService:
    deploy: true
    env:
      LIVEKIT_URL: "wss://livekit.example.com"
      LIVEKIT_KEY: "mykey"
      LIVEKIT_SECRET: "mysecret"
      LIVEKIT_FULL_ACCESS_HOMESERVERS: "example.com"
    ingress:
      enabled: true
      className: traefik
      host: "rtc.example.com"
      pathPrefix: "/livekit/jwt"
```

### 2. Subchart LiveKit (self-contained)

Add the LiveKit server chart as a dependency in `Chart.yaml`:

```yaml
dependencies:
  - name: livekit-server
    version: "1.x"
    repository: "https://livekit.github.io/livekit-helm"
    condition: matrixRtc.livekit.deploy
```

Then enable deployment:

```yaml
matrixRtc:
  enabled: true
  livekit:
    deploy: true
    hostNetwork: true          # required for UDP port ranges
    config:
      keys:
        devkey: "my-secret-key"
  authService:
    deploy: true
    env:
      LIVEKIT_URL: "ws://{{ .Release.Name }}-livekit-server:7880"
      LIVEKIT_KEY: "devkey"
      LIVEKIT_SECRET: "my-secret-key"
      LIVEKIT_FULL_ACCESS_HOMESERVERS: "example.com"
```

## Development

### Lint

```bash
helm lint . --set serverName=test.local
```

### Template (dry-run)

```bash
# Default
helm template tuwunel . --set serverName=test.local

# With MatrixRTC
helm template tuwunel . --set serverName=test.local \
  --set matrixRtc.enabled=true \
  --set matrixRtc.authService.env.LIVEKIT_URL=ws://livekit.test \
  --set matrixRtc.authService.env.LIVEKIT_KEY=devkey \
  --set matrixRtc.authService.env.LIVEKIT_SECRET=secret \
  --set matrixRtc.authService.env.LIVEKIT_FULL_ACCESS_HOMESERVERS=test.local
```

## Upgrading

```bash
helm upgrade my-tuwunel tuwunel/tuwunel -f values.yaml
```

## Uninstall

```bash
helm uninstall my-tuwunel
```

> **Warning:** The PersistentVolumeClaim for the database is NOT deleted by default. Remove it manually if you want to wipe data:
> ```bash
> kubectl delete pvc data-my-tuwunel-0
> ```
