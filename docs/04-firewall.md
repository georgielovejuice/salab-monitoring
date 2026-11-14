# 04 — Firewalls

Back to [README](../README.md) · [Progress](00-progress.md)

Layers a packet passes, outside → in:

```
Internet ─► [Azure NSG] ─► Zabbix server            (cloud side)
Teammate ─► [Tailscale policy] ─► CT 103 ─► [Proxmox firewall] ─► VM ─► [agent Server= allowlist]
```

## Proxmox firewall

### Three levels
| Level | Filters traffic to | Ours |
|---|---|---|
| **Datacenter** | All Proxmox hosts in the cluster (not VMs). Master on/off switch. Where Security Groups / Aliases / IPSets are **defined** | Firewall = Yes |
| **Node** (`kim-sever`) | The Proxmox host itself | 8006 rule |
| **VM** | That VM only | Security Group `sa-web`, sa-proxy rules |

- Host rules never filter VM traffic.
- A VM's firewall works only if Datacenter switch + VM Options switch + NIC firewall checkbox are all on.
- Default VM input policy = **DROP**. Output = ACCEPT.
- **Stateful**: replies to connections the VM started are allowed automatically.
- Enforced on the host, outside the VM: root inside a compromised VM can't turn it off (unlike ufw).
- ufw inside the VMs is **inactive**; Proxmox is the single filtering layer.

### Rules (done Nov 10)

**Node kim-sever**
| Dir | Action | Proto | Source | Dest | Port | Why |
|---|---|---|---|---|---|---|
| in | ACCEPT | tcp | 192.168.1.0/24 | 192.168.1.10 | 8006 | Proxmox UI from LAN |

**Security Group `sa-web`** (Datacenter → Firewall → Security Group; inserted on VMs 211, 212, 213, 215)
| Dir | Action | Proto | Source | Port | Why |
|---|---|---|---|---|---|
| in | ACCEPT | tcp | 192.168.1.54 | 10050 | Only the proxy may poll the agent |
| in | ACCEPT | tcp | 192.168.1.0/24 | 22 | SSH from LAN + Tailscale |
| in | ACCEPT | tcp | 192.168.1.0/24 | 80 | Web server |
| in | ACCEPT | icmp | 192.168.1.0/24 | — | Ping (also used by the proxy's ICMP checks) |
| in | ACCEPT | tcp | 192.168.1.0/24 | 5201 | iperf3 for the network-in test |

Port 80 from the LAN is also what the proxy's `net.tcp.service[http]` check uses.

**VM 214 sa-proxy**
| Dir | Action | Proto | Source | Port | Why |
|---|---|---|---|---|---|
| in | ACCEPT | tcp | 192.168.1.0/24 | 22 | SSH |
| in | ACCEPT | icmp | 192.168.1.0/24 | — | Ping |
| in | ACCEPT | tcp | 192.168.1.51,192.168.1.52,192.168.1.53,192.168.1.55 | 10051 | Agents' active checks connect **to** the proxy |

`192.168.1.0/24` also covers teammates: Tailscale traffic leaves CT 103 with source 192.168.1.14.

**Lesson learned:** the proxy is not outbound-only. Without the 10051 rule the agent logged `cannot connect to [192.168.1.54:10051]: i/o timeout` and active checks (log monitoring) stopped.

### Test results (Nov 10)
| Test | From | Result | Proves |
|---|---|---|---|
| `nc -zv 192.168.1.51 10050` | sa-proxy | succeeded | Proxy allowed |
| `nc -zv -w 3 192.168.1.51 10050` | sa-web2 | **timed out** | Everyone else blocked |
| ping / ssh via `-via-7` names | laptop | OK | Team access still works |

## Agent allowlist (inside the agent)

`Server=192.168.1.54` in `zabbix_agent2.conf`: the agent itself rejects anyone else. Seen in the log:
`connection from "192.168.1.14" rejected, allowed hosts: "192.168.1.54"` (a test via Tailscale/CT 103).

## Azure side

| Place | Rule | Owner |
|---|---|---|
| Azure **NSG** (Azure's network firewall in front of the VM) | Inbound TCP 10051 from **Any** (was the home IP; it changed under CGNAT) | Azure teammate |
| Zabbix UI → proxy → "Proxy address" | **empty** (cleared after the IP change) | homelab owner |
| Zabbix UI → proxy → Encryption | **PSK only** | homelab owner |

The home public IP **did** change (proxy Offline 3.5 h, buffered data delivered later). Final setup: NSG source Any + Proxy address empty; the PSK-only setting is the real authentication. Cleaner future fix: run the proxy → server link over Tailscale and close 10051 on the NSG.

**Open risk:** NSG port 80 (Zabbix UI, plain HTTP) is open to the internet → restrict, then Cloudflare Access.

## Reading errors: refused vs timed out

| Error | Meaning |
|---|---|
| **Connection refused** | Packet reached the VM; nothing listening on that port (service missing/stopped) |
| **Timed out** | No reply at all; a firewall silently dropped it |
