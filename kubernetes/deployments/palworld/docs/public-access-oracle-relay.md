# Plan: public access via an Oracle Always Free VPS on the tailnet

**Status: not built.** This is the escape hatch for when a friend won't install Tailscale.
Nothing in `manifests/` needs to change to adopt it — it reuses the `palworld-tailscale`
Service that is already deployed.

## The idea

Put Tailscale on the VPS instead of on your friends' machines. The VPS is the only thing
with a public IP; it joins the tailnet, and forwards the game port down to the cluster.

```
friend (no client at all)
      │  UDP 8211 → palworld.duarte-correia.pt (public A record → VPS public IP)
      ▼
Oracle Always Free VM ── tailscaled, tag:relay
      │  iptables DNAT → 100.x.y.z:8211  (the palworld-tailscale proxy device)
      ▼  (WireGuard, over the tailnet)
ts-palworld proxy pod ──→ palworld Service :8211 ──→ the server
```

Compared with the raw WireGuard version of this same idea, Tailscale carries the key
exchange, NAT traversal, reconnects and rekeying, so what you own on the VPS is one
`tailscale up` and two iptables rules.

## Why this is worth the trouble

- **Friends install nothing.** They type `palworld.duarte-correia.pt` and connect. That was
  the original requirement.
- **No router change.** `tailscaled` dials outbound from the VPS; nothing is forwarded at
  the house.
- **Split horizon falls out for free.** Pi-hole answers `palworld.duarte-correia.pt` with
  the in-cluster Gateway IP, Cloudflare answers with the VPS IP. LAN players keep going
  direct at full speed and never touch the relay; only outside players pay the extra hop.
- **The blast radius is one port.** A tailnet ACL limits `tag:relay` to udp/8211 on the
  proxy device, so a compromised VPS cannot reach the rest of the tailnet or the cluster.

## Cost

- €0 — Oracle Always Free, 10 TB/month egress. Palworld runs ~1-2 Mbps per player, so a
  full 16-player session for a month is nowhere near the cap.
- The AMD micro shape (1/8 OCPU, 1 GB) is enough; this box only shuffles packets. Don't
  bother fighting for ARM capacity.
- Latency: one extra hop for remote players (~10-20 ms if you pick `eu-madrid-1`).

## Build steps

### 1. Tailscale ACL — do this first

In the Tailscale admin console, create the tag and restrict it. `tag:relay` must be able to
reach the Palworld proxy device and nothing else:

```jsonc
"tagOwners": {
  "tag:relay": ["autogroup:admin"],
},
"acls": [
  {
    "action": "accept",
    "src":    ["tag:relay"],
    "dst":    ["palworld:8211"],   // the device the k8s operator registers
    "proto":  "udp",
  },
],
```

Then mint a reusable, pre-authorized auth key tagged `tag:relay`. Store it in Bitwarden as
`rafael-palworld-relay-tailscale-authkey` — nothing else should ever hold it.

### 2. The VM

Oracle Cloud → Compute → Instance, Always Free eligible shape, Ubuntu LTS, region
`eu-madrid-1`. Give it a reserved (not ephemeral) public IPv4 so the DNS record stays valid
across a stop/start.

### 3. Open udp/8211 in BOTH firewalls

This is where Oracle eats an hour if you forget. Two independent layers:

```bash
# a) VCN security list / NSG — in the Oracle console:
#    Ingress, source 0.0.0.0/0, IP protocol UDP, destination port 8211

# b) the instance's own rules — Oracle's images ship with a DROP-everything INPUT chain
sudo iptables -I INPUT -p udp --dport 8211 -j ACCEPT
sudo netfilter-persistent save
```

Neither one alone is enough, and a packet dropped by (b) looks exactly like a packet dropped
by (a). Test with `sudo tcpdump -ni any udp port 8211` from the VM while a friend tries.

### 4. Tailscale on the VM

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --authkey "tskey-auth-..." --hostname palworld-relay --advertise-tags tag:relay
tailscale status | grep palworld     # note the 100.x.y.z of the proxy device
```

### 5. Forward the port

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-relay.conf
sudo sysctl --system

PAL=100.x.y.z          # the palworld proxy device from step 4
sudo iptables -t nat -A PREROUTING  -p udp --dport 8211 -j DNAT --to-destination $PAL:8211
sudo iptables -t nat -A POSTROUTING -p udp -d $PAL --dport 8211 -j MASQUERADE
sudo netfilter-persistent save
```

MASQUERADE is not optional: without it the reply comes from the tailnet address and the
friend's client drops it as an unsolicited packet.

Pin the tailnet IP for that device in the admin console, or a re-register moves it and the
DNAT rule silently points at nothing.

### 6. Public DNS

In Cloudflare, `palworld` A → the VPS public IP, **DNS only (grey cloud)**. Proxying it
would send the traffic through Cloudflare, which does not carry UDP outside of Spectrum
Enterprise. Do not let cloudflare-ddns manage this record — it points at the house, not the
VPS.

## Operating it

- **Oracle reclaims idle Always Free instances.** A relay that only sees traffic when
  friends are online looks idle. If it gets reclaimed the LAN server keeps working and only
  outside players break, which is a confusing failure — so if this becomes load-bearing,
  either keep a small cron busying the box or accept the risk knowingly.
- Health check from anywhere: `nc -u -z -w3 palworld.duarte-correia.pt 8211`.
- If remote players drop but LAN players are fine, the relay is the suspect: check
  `tailscale status` on the VM before touching anything in the cluster.

## Rollback

`sudo tailscale down` on the VM and delete the Cloudflare A record. The LAN path is
untouched by all of this.
