# Proxmox firewall rules

Explained in [docs/04-firewall.md](../docs/04-firewall.md).

**Node `kim-sever`**
| Dir | Action | Proto | Source | Dest | Port | Comment |
|---|---|---|---|---|---|---|
| in | ACCEPT | tcp | 192.168.1.0/24 | 192.168.1.10 | 8006 | Proxmox web UI from LAN |

**Security Group `sa-web`** (inserted on VMs 211, 212, 213)
| Dir | Action | Proto | Source | Port | Comment |
|---|---|---|---|---|---|
| in | ACCEPT | tcp | 192.168.1.54 | 10050 | zabbix proxy polls agent |
| in | ACCEPT | tcp | 192.168.1.0/24 | 22 | ssh from LAN + tailscale |
| in | ACCEPT | tcp | 192.168.1.0/24 | 80 | web server |
| in | ACCEPT | icmp | 192.168.1.0/24 | — | ping |

**VM 214 `sa-proxy`**
| Dir | Action | Proto | Source | Port | Comment |
|---|---|---|---|---|---|
| in | ACCEPT | tcp | 192.168.1.0/24 | 22 | ssh |
| in | ACCEPT | icmp | 192.168.1.0/24 | — | ping |
| in | ACCEPT | tcp | 192.168.1.51,192.168.1.52,192.168.1.53 | 10051 | agents active checks to proxy |

Default VM input policy: DROP. The firewall is stateful.
