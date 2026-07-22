# Palworld dedicated server

Palworld 1.0 dedicated server on the porto cluster, in the `rafael-homelab` namespace.

| | |
|---|---|
| Image | `hydrodog11/palworld:latest`, built from [rafaelmdc/palworld](https://github.com/rafaelmdc/palworld) |
| Connect | `palworld.duarte-correia.pt:8211` |
| Storage | 40Gi RWO on `iscsi-zfs`, mounted at `/palworld` |
| Admin | RCON on `palworld-rcon:25575`, in-cluster only |

## Before the first sync

Create the Bitwarden items the ExternalSecrets expect — the pod will not start without the
first two:

- `rafael-palworld-admin-password` — in-game admin + RCON password
- `rafael-palworld-server-password` — the password players type to join
- `rafael-palworld-tailscale-authkey` — auth key for the Tailscale node. Generate at
  login.tailscale.com/admin/settings/keys with **Reusable ON, Ephemeral OFF, no tags**;
  a tagged key produces a machine that can't be shared. The relay pod waits harmlessly
  until this exists.

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
write to Cloudflare. Remote players go over Tailscale, via the dedicated node in
[`manifests/tailscale-relay.yaml`](manifests/tailscale-relay.yaml).

That node runs on **Rafael's own tailnet** with an untagged auth key, which matters: the
Tailscale operator (`tailscale.com/expose`, as bazarr/sonarr/transmission use) tags every
proxy it creates, and a tagged machine **cannot be shared** — *"a machine cannot be shared
with a tag ... as only users can accept machine shares"*. The operator is also joined to
`munchkin-cloud.ts.net`, which is Eduardo's. An untagged, user-owned node sidesteps both.

To let a friend in:

1. Tailscale admin console → **Machines** → `palworld` → **Share…**, send them the link
2. Give them the server password
3. They install Tailscale, accept the share, and join at `<tailnet-ip>:8211`

Sharing a single machine gives them that one node and nothing else — no seat in the tailnet,
no access to the rest of the cluster. Connections are direct peer-to-peer, so this is also
the lowest-latency remote option; better than any relay.

Its one cost is that friends have to install a client. When that becomes a problem, see
[docs/public-access-oracle-relay.md](docs/public-access-oracle-relay.md): an Oracle Always
Free VM joins the same tailnet and forwards `udp/8211` publicly, so friends install nothing
and connect straight to `palworld.duarte-correia.pt`. It builds on this same node, so
adopting it changes nothing already deployed.

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
