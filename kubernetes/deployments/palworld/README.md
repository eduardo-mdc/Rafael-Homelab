# Palworld dedicated server

Palworld 1.0 dedicated server on the porto cluster, in the `rafael-homelab` namespace.

| | |
|---|---|
| Image | `hydrodog11/palworld:latest`, built from [rafaelmdc/palworld](https://github.com/rafaelmdc/palworld) |
| Connect | `palworld.duarte-correia.pt:8211` |
| Storage | 40Gi RWO on `iscsi-zfs`, mounted at `/palworld` |
| Admin | RCON on `palworld-rcon:25575`, in-cluster only |

## Before the first sync

Create the two Bitwarden items the ExternalSecret expects — the pod will not start without them:

- `rafael-palworld-admin-password` — in-game admin + RCON password
- `rafael-palworld-server-password` — the password players type to join

## How the hostname works

Palworld speaks raw UDP, so it can't ride the Cloudflare tunnel or an HTTPS Gateway — both
are HTTP only. Instead it gets a UDP listener on an Envoy Gateway, which takes its own IP
from the Cilium pool, and a `UDPRoute` carrying the external-dns hostname annotation:

```
UDPRoute (external-dns.alpha.kubernetes.io/hostname: palworld.duarte-correia.pt)
      │
      ▼ (external-dns, --source=gateway-udproute, ~1 min)
A record in Pi-hole → the palworld-gateway IP
      │
      ▼
LAN clients type palworld.duarte-correia.pt:8211 in Palworld
```

Same mechanism terraria uses for `terraria.duarte-correia.pt`.

### Reaching it from outside the house

That record lives in **Pi-hole**, so it resolves on the LAN only — external-dns doesn't
write to Cloudflare. The current answer for remote players is the `palworld-tailscale`
Service (`tailscale.com/expose`): the operator publishes the server on the tailnet as
`palworld.<tailnet>.ts.net`, you share the node from the Tailscale admin console, and your
friend installs Tailscale and connects to that name. Same pattern bazarr/sonarr/transmission
already use here. Connections are direct peer-to-peer, so this is the lowest-latency remote
option — better than any relay.

Its one cost is that friends have to install a client. When that becomes a problem, see
[docs/public-access-oracle-relay.md](docs/public-access-oracle-relay.md): an Oracle Always
Free VM joins the same tailnet and forwards `udp/8211` publicly, so friends install nothing
and connect straight to `palworld.duarte-correia.pt`. It reuses this same Service, so
adopting it changes nothing in `manifests/`.

Note what is *not* an option: the Cloudflare tunnel is HTTP-only, and pushing raw UDP
through Cloudflare needs Spectrum (Enterprise). A plain router port-forward would also work,
but is deliberately avoided here.

## First boot takes a while

SteamCMD pulls ~10 GB before the server binds anything. The startup probe allows 30 minutes;
watch it with:

```bash
kubectl -n rafael-homelab logs -f deploy/palworld
```

## Admin commands

```bash
kubectl -n rafael-homelab exec deploy/palworld -- rcon-cli "ShowPlayers"
kubectl -n rafael-homelab exec deploy/palworld -- rcon-cli "Save"
kubectl -n rafael-homelab exec deploy/palworld -- rcon-cli "Broadcast Server_restarting_in_5_minutes"
```

## Saves and backups

Hourly tarballs land in `/palworld/backups`, pruned after 7 days. The world itself lives in
`/palworld/Pal/Saved/SaveGames/0/<world-id>/`. Both are on the same PVC — if you want a copy
off the cluster, pull one down with `kubectl cp`.

## Versions

Two layers move independently:

- **The image** — `hydrodog11/palworld`, built by
  [rafaelmdc/palworld](https://github.com/rafaelmdc/palworld) and rolled here by
  argocd-image-updater (digest strategy on `latest`), exactly like cladewright.
- **The game build** — comes from Steam at container start. `UPDATE_ON_BOOT=true` means
  every restart takes the current public build; set it to `false` to freeze on whatever is
  installed on the PVC.

## Gameplay settings

Rates, difficulty and the rest are the `PalWorldSettings.ini` knobs, exposed as env vars on
the Deployment (e.g. `DIFFICULTY`, `EXP_RATE`, `PAL_CAPTURE_RATE`, `DAY_TIME_SPEEDRATE`).
Anything not set keeps the vanilla default.

One default is deliberately overridden: `DEATH_PENALTY=None`, so nothing is dropped on
death. Vanilla is `All` — items, equipment and your party Pals all stay on the corpse.

Full list of knobs:
<https://palworld-server-docker.loef.dev/configuration/server-settings>
