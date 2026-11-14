# 03 — Access control

Back to [README](../README.md) · [Progress](00-progress.md)

Two locks: **Tailscale** decides who can reach the network (network lock); **Proxmox accounts** decide what they can do (permission lock). Defense in depth.

## Proxmox accounts (done)

| Object | Value |
|---|---|
| Pool | `salab` = VMs 211, 212, 213, 214, 215 |
| Group | `salab-team` = `<name>@pve`, `<name>@pve`, `<name>@pve` |
| Permission | path `/pool/salab`, group `salab-team`, role `PVEVMAdmin`, propagate |
| Login realm | **Proxmox VE authentication server** (not Linux PAM) |

Teammates can console/start/stop/snapshot/delete lab VMs, can't create VMs, can't see other homelab services. One account per person = accountability (task log shows who did what). Nobody uses `root`.

## Tailscale

- Teammates invited as **Member** (not Admin).
- Subnet router: CT 103 (192.168.1.14), also an exit node (teammates can't use it).
- Linux clients need `--accept-routes`.

### 4via6 (overlapping home subnets)
Every teammate's home LAN is also `192.168.1.0/24`, so plain subnet routes conflict. Fix: **4via6, site ID 7**.

CT 103 setup:
```bash
tailscale debug via 7 192.168.1.0/24          # prints the IPv6 prefix: fd7a:115c:a1e0:b1a:0:7:c0a8:100/120
# /etc/sysctl.d/99-tailscale.conf: net.ipv4.ip_forward = 1 ; net.ipv6.conf.all.forwarding = 1
tailscale set --advertise-routes=192.168.1.0/24,fd7a:115c:a1e0:b1a:0:7:c0a8:100/120
```
Route approved in the admin console.

- Names: `192-168-1-<last octet>-via-7` (e.g. `https://192-168-1-10-via-7:8006`, `ssh user@192-168-1-51-via-7`).
- These names have AAAA records only: `nslookup -type=AAAA ...`; on Mac use `ping6`.
- Path: name → `fd7a:...:7:c0a8:XXXX` (last 32 bits = IPv4 in hex, e.g. `c0a8:133` = 192.168.1.51) → Tailscale routes to CT 103 → policy checked → CT 103 decodes site 7 + IPv4 → forwards as IPv4 from 192.168.1.14 (SNAT).

### Policy (grants)

```jsonc
{
  "groups": {"group:team": ["member1@example.com", "member2@example.com", "member3@example.com"]},
  "hosts": {
    "proxmox": "fd7a:115c:a1e0:b1a:0:7:c0a8:10a",
    "sa-web1": "fd7a:115c:a1e0:b1a:0:7:c0a8:133",
    "sa-web2": "fd7a:115c:a1e0:b1a:0:7:c0a8:134",
    "sa-web3": "fd7a:115c:a1e0:b1a:0:7:c0a8:135",
    "sa-proxy": "fd7a:115c:a1e0:b1a:0:7:c0a8:136",
    "sa-web4": "fd7a:115c:a1e0:b1a:0:7:c0a8:137",
  },
  "grants": [
    {"src": ["autogroup:admin"], "dst": ["*"], "ip": ["*"]},
    {"src": ["group:team"], "dst": ["proxmox"], "ip": ["tcp:8006"]},
    {"src": ["group:team"], "dst": ["sa-web1", "sa-web2", "sa-web3", "sa-web4", "sa-proxy"], "ip": ["tcp:22", "tcp:80", "icmp:*", "ipv6-icmp:*"]},
  ],
  "tests": [ /* each member: accept proxmox:8006 + sa-web ports; deny proxmox:22 */ ],
}
```

| Who | Can reach |
|---|---|
| Owner (`autogroup:admin`) | Everything |
| `group:team` | Proxmox UI tcp:8006 only |
| `group:team` | sa-web1..4 + sa-proxy: SSH, HTTP, ping |

`tests` = assertions Tailscale checks every time the policy is saved; a policy that breaks them is rejected.

Done: sa-proxy (`c0a8:136` = .54) and sa-web4 (`c0a8:137` = .55) added to `hosts` and the lab grant; tests also cover sa-web4 and sa-proxy.

Troubleshooting a teammate: dorm networks may block Tailscale (`tailscale netcheck`; phone hotspot works).

## Zabbix UI accounts (planned, never set up)

- One account per member; `Admin` kept as break-glass with a long password.
- Professor: read-only user, permission limited to host group **`salab`** (group exists).
- Cloudflare Tunnel + Access (`zabbix.<our-domain>`) was planned for instructor access but not needed: the demo was done in person.
