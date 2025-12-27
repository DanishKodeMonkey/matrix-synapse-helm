# matrix-synapse-helm

Work in progress – a k3s-based Helm chart for deploying a Matrix server stack on a homelab.

This Helm chart provides a complete starting point for running your own **Matrix Synapse server**, the **Element frontend**, and a **Coturn TURN server** for voice and video support. The goal is a ready-to-use setup with baseline configurations that you can expand and customize for your needs.

---

## Overview of Components

### Matrix Synapse

Matrix Synapse is the server component of the Matrix protocol. It handles:

- User accounts and authentication
- Federated communication with other Matrix servers
- Storage of chat history, media, and state

**Persistent storage is crucial**:  

Synapse stores all its data in a `/data` volume, including:

- User database
- Media uploads
- Signing keys
- Server configuration and state

Losing this volume can result in **lost users, federations, and server identity**.

**Admin user creation**:

Use the official command:

```bash
kubectl exec -n matrix -it <synapse-pod> -- \
  register_new_matrix_user \
  -c /data/homeserver.yaml \
  --user <username> \
  --password <password> \
  --admin
```

# Element (Web Client)

Element provides the user interface for interacting with your Matrix server. The chart deploys Element alongside Synapse, including:

Ingress for access via your domain

Baseline configuration for connecting to your Synapse server

Media proxying through Synapse

Users can register and log in immediately, but reCAPTCHA and registration secrets are configurable via .secrets.yaml to prevent spam or unauthorized accounts.

# Coturn (TURN Server)

Coturn provides voice and video relay capabilities for WebRTC in Matrix. Without it, peer-to-peer voice/video often fails due to NAT or firewall restrictions.

Key features configured in this chart:

Listening on standard TURN ports (3478 UDP/TCP) and TLS (5349)

Relay IP set to your internal homelab IP

External IP set to your public static IP (or the IP used by your DNS record)

Secret-based authentication (static-auth-secret) for secure usage

UDP port range (49160-49200) for relayed traffic

Logging to stdout for easy debugging in Kubernetes

Notes on TLS:

TLS for Coturn is currently not enabled. Certificates can be provisioned via Cert-Manager and ACME (Let’s Encrypt).

Once certificates exist, the Coturn config can be updated to use cert-file and pkey-file for encrypted voice/video connections.

Testing Coturn:

A quick test for connectivity:
```bash
nc -vuz turn.example.com 3478
```

# Deployment
## Prerequisites

k3s cluster (or other Kubernetes cluster)

A dedicated namespace (recommended: matrix)

Ingress controller (e.g., Traefik)

Cert-Manager for TLS (optional, for Element and future Coturn TLS)

Install / Upgrade

Secrets are stored in .secrets.yaml and values in values.yaml. Example:
```bash
helm upgrade --install matrix ./ -n matrix -f values.yaml -f .secrets.yaml
```

example `.secrets.yaml`
```yaml
synapse:
  recaptcha:
    siteKey: "<Google reCAPTCHA site key>"
    secretKey: "<Google reCAPTCHA secret key>"
  registration_shared_secret: "<random-secret-string>"

coturn:
  turn_shared_secret: "<random-long-secret>"
  public_ip_static: "<your public IP reachable by UDP>"
```

Persistent Volume Claims (PVCs)

PVCs store all essential Synapse data:

- Database files

- Media uploads

- Signing keys

- Configuration

Without the PVC, data loss is likely, including user accounts and federation trust.


# Ingress and DNS

- The chart uses Ingresses for Element and Synapse traffic.

- Example Traefik annotations:
```yaml
annotations:
  kubernetes.io/ingress.class: traefik
  traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
  cert-manager.io/cluster-issuer: letsencrypt
```
- Your domain should point to your public IP via DNS (e.g., matrix.example.com for Synapse, turn.example.com for Coturn).

# Notes and TODOs

TLS for Coturn is not yet configured.

Make sure UDP ports for Coturn are open on your firewall/router.

Regularly back up the /data PVC to avoid losing users, media, or server identity.

# This chart provides a homelab-ready setup with Synapse, Element, and Coturn. From here, you can expand with:

- Custom Synapse modules

- Advanced Coturn configuration

- Additional Element settings for branding or privacy