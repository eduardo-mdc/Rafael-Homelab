# Examples

Reference manifests for common patterns used in this homelab. None of these are deployed — copy what you need into your app's `manifests/` directory and rename accordingly.

| File | What it covers |
|---|---|
| `certificate.yaml` | TLS certificate via cert-manager (Let's Encrypt + Cloudflare DNS-01) |
| `gateway.yaml` | Envoy Gateway + HTTPRoute + traffic policies |
| `gateway-with-oidc.yaml` | Same as above + Keycloak OIDC enforcement via SecurityPolicy |
| `tunnel-binding.yaml` | Cloudflare Tunnel binding on the shared cluster tunnel |
| `external-secret.yaml` | Pulling secrets from Bitwarden, with and without value templating |
| `postgres-cluster.yaml` | CloudNativePG 3-node HA cluster with WAL archiving + scheduled S3 backup |
| `pvc.yaml` | Persistent volume claim (Longhorn) |

---

## How DNS works when you deploy a Gateway

When you create an `HTTPRoute` with a hostname (e.g. `my-app.duarte-correia.pt`), DNS is updated automatically through this chain:

```
HTTPRoute created
      │
      ▼ (every ~1 minute)
external-dns watches HTTPRoute resources
      │
      ▼
Adds A record to Pi-hole (172.16.0.254 / 172.16.0.253)
      │
      ▼
Local network clients resolve the hostname to the cluster ingress IP
```

external-dns polls for changes on a **1-minute interval**, so there is a short delay between deploying your app and the DNS record appearing in Pi-hole. On top of that, your client machine may have cached a negative response (NXDOMAIN) from before the record existed — flushing your local DNS cache speeds this up:

```bash
# macOS
sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder

# Windows
ipconfig /flushdns

# Linux (systemd-resolved)
resolvectl flush-caches
```

---

## Pi-hole as a local override for Cloudflare Tunnels

Cloudflare Tunnels expose your services publicly via `*.duarte-correia.pt`. Inside the local network, Pi-hole holds A records for the same hostnames pointing directly to the cluster ingress IP.

Because Pi-hole is the upstream DNS resolver for the local network, **local clients resolve the hostname to the cluster directly** — traffic never leaves the network to go through Cloudflare. This gives you:

- Lower latency on the local network (no round-trip through Cloudflare)
- Services remain accessible locally even if the Cloudflare tunnel is down
- The same hostname works both inside and outside the network without any client configuration
