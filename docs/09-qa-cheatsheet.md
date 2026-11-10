# 09 — Q&A cheat sheet

Back to [README](../README.md) · [Progress](00-progress.md)

## Architecture

| Question | Answer |
|---|---|
| Why a Zabbix proxy? | Home is behind CGNAT: the cloud can't reach in. The proxy collects over the LAN and sends to the server over one outbound PSK-encrypted connection, buffering up to 24 h if the WAN fails. Standard branch-office pattern. |
| Walk through the data flow | (1) Host added in UI → server DB. (2) Proxy connects OUT to server:10051, downloads its config. (3a) Passive: proxy → agent:10050. (3b) Active: agent → proxy:10051, collects logs itself, pushes. (4) Proxy buffers in SQLite, sends batches OUT. (5) Server saves, evaluates triggers → email. (6) UI reads DB. Every connection to Azure starts at home. |
| How does the proxy know which hosts to monitor? | Each host has "Monitored by: Proxy sa-proxy" on the server. The proxy identifies itself (Hostname + PSK) and the server sends only its hosts; the proxy stores them in its SQLite DB. |
| Where did CPU/memory/disk metrics come from? | The built-in template "Linux by Zabbix agent": items, triggers, discovery rules and dashboards, copied onto the host when linked. |
| Active vs passive checks? | Passive: proxy polls the agent on 10050. Active: agent connects to 10051, gets its list, pushes. Log monitoring is active-only. |
| Why VMs, not LXC? | Own kernel, so real `/proc` metrics |
| Why ballooning off? | Fixed RAM keeps the memory alert threshold meaningful |
| What if Azure dies? | Proxy buffers 24 h. Plan B: Zabbix server VM at home, change one line (`Server=`) on the proxy |

## Networking

| Question | Answer |
|---|---|
| What is CGNAT? | The ISP shares one public IPv4 between many customers, so an IP allowlist also admits strangers and inbound connections are impossible. PSK does the real authentication. |
| Public vs private IP / NAT? | VMs use private 192.168.1.x; the router rewrites outgoing traffic to the public IP. Outbound works, inbound doesn't. |
| Overlapping home subnets? | Tailscale 4via6 maps the LAN into a unique IPv6 range with site ID 7; the subnet router translates back to IPv4 |
| Refused vs timed out? | Refused = reached the host, nothing listening. Timed out = a firewall dropped it. |

## Security

| Question | Answer |
|---|---|
| How do teammates get in? | Tailscale (network lock, deny by default) + Proxmox per-user PVEVMAdmin on pool `salab` (permission lock) |
| PSK vs SSH keys? | PSK symmetric: same secret both ends (mode 600). SSH asymmetric: only the public key is copied. |
| Why the Proxmox firewall instead of ufw? | Enforced on the host, outside the VM, so a compromised VM can't disable it. One Security Group = same rules on all web VMs. |
| Who can talk to the agent? | Only the proxy: Proxmox rule (10050 from .54) + agent `Server=192.168.1.54`. Proven: sa-web2 → 10050 timed out. |
| Is the proxy outbound-only? | No — agents connect to it on 10051 for active checks; firewall allows that from the 3 web VMs only |
| Why add `zabbix` to `adm`? | Ubuntu logs are `syslog:adm 640`; `adm` = read-only log access, not sudo. Agent runs as unprivileged `zabbix`. |
| What is the NSG? | Azure's network firewall in front of the VM |

## Packages

| Question | Answer |
|---|---|
| dpkg vs apt? Why the zabbix-release .deb? | Ubuntu has no agent2 and only Zabbix 6.0. The release .deb (installed with `dpkg -i`) adds repo.zabbix.com + its GPG key so apt can install 8.0, matching server/proxy. apt downloads and resolves dependencies; dpkg installs a local .deb. |
| What is the `2:` in the version? | The epoch: overrides version comparison so Zabbix's package beats Ubuntu's `1:` |
| `free` vs `available` memory? | Linux uses spare RAM as cache; "available" is what apps can actually use |

## Known limitations (say these before he finds them)

Zabbix UI on HTTP and open to the internet (fix: Cloudflare Access) · no management VLAN · self-signed Proxmox cert · CT 103 is a single point of failure for teammate access · `adm` gives the agent read access to all logs.
