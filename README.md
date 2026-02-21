# matrix-synapse-helm
Written by AI, audited by danishkodemonkey

Helm chart for running a Matrix stack on Kubernetes/k3s:

- Synapse (homeserver)
- Element Web (client)
- Coturn (TURN relay)
- Optional MatrixRTC stack (LiveKit + JWT service)

This README explains how to configure and deploy the chart safely.

## File Structure

```text
matrix-synapse/
├── Chart.yaml
├── values.yaml
├── .secrets.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── synapse-*.yaml
│   ├── element-*.yaml
│   ├── coturn-*.yaml
│   ├── livekit-*.yaml
│   ├── matrixrtc-jwt-*.yaml
│   └── matrix-rtc-ingress.yaml
└── README.md
```

What goes where and why:

- `Chart.yaml`: Helm chart metadata (name, version, app version). Helm uses this to identify and package the chart.
- `values.yaml`: Default, non-secret configuration. This is your baseline config committed to git.
- `.secrets.yaml`: Secret overrides (tokens, shared secrets, private keys). Passed at deploy time and kept out of git.
- `templates/_helpers.tpl`: Shared Helm template helpers used by multiple manifests.
- `templates/synapse-*.yaml`: Synapse deployment, service, ingress, PVC, and extra config.
- `templates/element-*.yaml`: Element deployment, service, ingress, and `config.json`.
- `templates/coturn-*.yaml`: Coturn deployment/config/secret/certificate resources.
- `templates/livekit-*.yaml`: LiveKit config, deployment, and service for MatrixRTC media.
- `templates/matrixrtc-jwt-*.yaml`: JWT service resources used by Element/MatrixRTC auth flow.
- `templates/matrix-rtc-ingress.yaml`: Public routing for MatrixRTC endpoints (`/livekit/sfu`, `/livekit/jwt`).
- `README.md`: Human runbook for install, operations, and troubleshooting.

## What This Chart Deploys

### Synapse

- Stateful Matrix homeserver
- Persistent data on PVC `synapse-data`
- Ingress on `https://<global.domain>`

### Element

- Web client
- Ingress on `https://<global.elementDomain>`

### Coturn

- TURN server for VoIP reliability
- Host-networked deployment (binds node network directly)
- Uses shared-secret auth with Synapse

### MatrixRTC (optional)

Enabled with `matrixrtc.enabled: true`:

- LiveKit server (`/livekit/sfu`)
- JWT service (`/livekit/jwt`)
- Ingress on `https://<matrixrtc.host>`
- Synapse well-known metadata for Element Call discovery

## Prerequisites

- k3s or Kubernetes cluster
- Namespace `matrix` (or your preferred namespace)
- Ingress controller (Traefik assumed in current templates)
- cert-manager + ClusterIssuer (`letsencrypt` by default)
- Public DNS records for:
  - `matrix.<domain>`
  - `element.<domain>`
  - `turn.<domain>`
  - `rtc.<domain>` (if MatrixRTC enabled)
- google recaptcha keys, or alternative, see more on this here https://matrix-org.github.io/synapse/v1.45/CAPTCHA_SETUP.html
## Files

- `values.yaml`: non-secret defaults and infrastructure settings
- `.secrets.yaml`: sensitive values (do not commit)

## Required Configuration

### Minimal `.secrets.yaml`

```yaml
synapse:
  recaptcha:
    siteKey: "<recaptcha-site-key>"
    secretKey: "<recaptcha-secret-key>"
  registration_shared_secret: "<random-long-secret>"

coturn:
  turn_shared_secret: "<random-long-secret>"
  public_ip_static: "<public-ip-for-turn>"

matrixrtc:
  livekit:
    key: "<livekit-key>"
    secret: "<livekit-secret>"
```

### Important `values.yaml` settings

- `global.domain`: Synapse hostname
- `global.elementDomain`: Element hostname
- `coturn.realm` and `coturn.turn_url`: TURN hostname
- `matrixrtc.ingressClassName`: set explicitly (for k3s + Traefik, set `traefik`)
- `matrixrtc.livekit.rtc.portRangeStart/End`: UDP range for LiveKit media

## Customization Examples

Keep non-secret configuration in `values.yaml` and secret overrides in `.secrets.yaml`.

### Element branding and embedded pages

```yaml
element:
  branding:
    appName: "Danish Kode Matrix"
    authHeaderLogoUrl: "https://element.danishkodemonkey.net/static-assets/logo.svg"
    welcomeBackgroundUrl: "https://element.danishkodemonkey.net/static-assets/welcome-bg.svg"
    userNotice: "By logging in, you agree to server rules."
    authFooterLinks:
      - text: "Server Rules"
        url: "https://element.danishkodemonkey.net/pages/rules.html"
  embeddedPages:
    enabled: true
    homeUrl: "https://element.danishkodemonkey.net/pages/home.html"
    welcomeUrl: "https://element.danishkodemonkey.net/pages/welcome.html"
```

### Synapse onboarding, retention, and moderation

```yaml
synapse:
  registration:
    enabled: true
    requiresToken: true
  onboarding:
    autoJoinRooms:
      - "#announcements:matrix.danishkodemonkey.net"
      - "#rules:matrix.danishkodemonkey.net"
    autoJoinMxidLocalpart: "system"
  retention:
    enabled: true
    defaultPolicy:
      maxLifetime: "90d"
  media:
    maxUploadSize: "100M"
  serverNotices:
    enabled: true
    systemMxidLocalpart: "notices"
    roomName: "Server Notices"
```

### Advanced Element overrides

Set any additional Element `config.json` keys via:

```yaml
element:
  extraConfig:
    permalink_prefix: "https://matrix.danishkodemonkey.net"
```

### Serve local `/pages` with in-cluster Nginx

```yaml
element:
  staticPages:
    enabled: true
    pathPrefix: /pages
    assetsPathPrefix: /static-assets
```

This serves files from chart folder `static/pages/*` at:

- `https://<element-domain>/pages/home.html`
- `https://<element-domain>/pages/welcome.html`
- `https://<element-domain>/pages/rules.html`
- `https://<element-domain>/static-assets/logo.svg`
- `https://<element-domain>/static-assets/welcome-bg.svg`

## Install / Upgrade

```bash
kubectl create namespace matrix --dry-run=client -o yaml | kubectl apply -f -

cd matrix-synapse
helm upgrade --install matrix . \
  -n matrix \
  -f values.yaml \
  -f .secrets.yaml
```

Recommended safer upgrade:

```bash
helm upgrade --install matrix . \
  -n matrix \
  -f values.yaml \
  -f .secrets.yaml \
  --atomic --timeout 10m
```

## Verify Deployment

```bash
kubectl get pods -n matrix
kubectl get ingress -n matrix
helm status matrix -n matrix
```

Check live logs for startup issues:

```bash
kubectl logs -n matrix deploy/synapse --tail=100
kubectl logs -n matrix deploy/livekit --tail=100
kubectl logs -n matrix deploy/matrixrtc-jwt --tail=100
```

## Register an Admin User

```bash
kubectl exec -n matrix -it <synapse-pod> -- \
  register_new_matrix_user \
  -c /data/homeserver.yaml \
  --user <username> \
  --password <password> \
  --admin
```

## Networking and Firewall

Open/forward these public inbound ports to the correct k3s node(s):

- `80/tcp` and `443/tcp` for ingress
- Coturn:
  - `3478/udp`
  - `3478/tcp`
  - `5349/tcp` (TURN-TLS)
  - `49160-49200/udp` (coturn relay range)
- LiveKit (if enabled):
  - `7881/tcp` (fallback)
  - `matrixrtc.livekit.rtc.portRangeStart-End/udp` (media range)

Example if using `50300-50400` for LiveKit:

```yaml
matrixrtc:
  livekit:
    rtc:
      portRangeStart: 50300
      portRangeEnd: 50400
```

## Cloudflare Notes

If using Cloudflare proxy/tunnel:

- `turn.<domain>` should be DNS-only (no orange-cloud proxy) for TURN ports.
- `rtc.<domain>` is best as DNS-only for stable WebRTC/media behavior.
- If Cloudflare Access is enabled, bypass auth for:
  - `https://matrix.<domain>/_matrix/*`
  - `https://matrix.<domain>/_synapse/*`
  - `https://rtc.<domain>/livekit/*`
  - all `turn.<domain>` traffic

## MatrixRTC Functional Check

1. Verify Synapse well-known advertises RTC focus:

```bash
curl -s https://matrix.example.com/.well-known/matrix/client
```

Expected key:

- `org.matrix.msc4143.rtc_foci`
- includes `livekit_service_url: https://rtc.example.com/livekit/jwt`

2. In Element, start a room call.
3. Confirm `livekit` and `matrixrtc-jwt` pods stay `Running`.

## Troubleshooting

### `MISSING_MATRIX_RTC_FOCUS`

Cause: Synapse is not advertising MatrixRTC focus in well-known.

Check that chart rendered and applied:

- `extra_well_known_client_content.org.matrix.msc4143.rtc_foci`
- `public_baseurl`

Then restart Synapse deployment.

### LiveKit crash with `LIVEKIT_PORT=tcp://...`

Cause: Kubernetes service-link env var collision.

Fix in chart: `enableServiceLinks: false` for LiveKit pod.

### Upgrade error: Secret `type` is immutable

If you changed secret type in templates, delete and recreate the secret once:

```bash
kubectl delete secret coturn-secret -n matrix
helm upgrade --install matrix . -n matrix -f values.yaml -f .secrets.yaml
```

## Rollback

```bash
helm history matrix -n matrix
helm rollback matrix <revision> -n matrix
```

Note: Rollback restores Kubernetes manifests, not historical PVC data.

## Security Notes

- Keep `.secrets.yaml` out of git.
- Rotate Synapse/Coturn/LiveKit secrets if exposed.
- Back up Synapse PVC regularly.
