# 09 — Q&A cheat sheet

Back to [README](../README.md) · [Progress](00-progress.md)

## Architecture

| Question | Answer |
|---|---|
| Why a Zabbix proxy? | Home is behind CGNAT, so Azure can't connect in. The proxy sends everything over one outbound PSK-encrypted connection and buffers 24 h if the link drops. |
| Walk through the data flow | The proxy downloads its host list from the server and collects from the agents on the LAN. It then pushes batches out to the server, so every connection to Azure starts at home. |
| How does the proxy know which hosts to monitor? | Each host is set to "Monitored by: Proxy sa-proxy" on the server. The proxy identifies itself and gets only its own hosts. |
| Where did CPU/memory/disk metrics come from? | From the built-in template "Linux by Zabbix agent active", linked to each host. We shortened its intervals to 10 s for the demo. |
| Active vs passive checks? | Passive: the proxy polls the agent on 10050, which we only use for `zabbix_get` debugging. Active: the agent connects to 10051 and pushes everything, logs included. |
| Why VMs, not LXC? | Each VM has its own kernel, so `/proc` metrics are real. |
| Why ballooning off? | Fixed RAM keeps the memory alert threshold meaningful. |
| What if Azure dies? | The proxy buffers 24 h. Plan B is a Zabbix server VM at home and one changed line (`Server=`) on the proxy. |

## Networking

| Question | Answer |
|---|---|
| What is CGNAT? | The ISP shares one public IPv4 between many customers, so nothing can connect in. An IP allowlist would also admit strangers, so PSK does the authentication. |
| Public vs private IP / NAT? | VMs use private 192.168.1.x and the router rewrites outgoing traffic to the public IP. Outbound works, inbound doesn't. |
| Overlapping home subnets? | Tailscale 4via6 maps the LAN into a unique IPv6 range with site ID 7. The subnet router translates back to IPv4. |
| Refused vs timed out? | Refused means the host was reached but nothing is listening. Timed out means a firewall dropped it. |

## Security

| Question | Answer |
|---|---|
| How do teammates get in? | Tailscale controls what they can reach. Proxmox accounts with PVEVMAdmin on pool `salab` control what they can do. |
| PSK vs SSH keys? | PSK is symmetric, with the same secret on both ends. SSH is asymmetric, and only the public key is copied. |
| Why the Proxmox firewall instead of ufw? | It runs on the host, so a compromised VM can't turn it off. One Security Group gives all web VMs the same rules. |
| Who can talk to the agent? | Only the proxy, enforced by a Proxmox rule and the agent's `Server=192.168.1.54`. A test from sa-web2 to 10050 timed out. |
| Is the proxy outbound-only? | No. Agents connect to it on 10051 for active checks, and the firewall allows that only from the 4 web VMs. |
| Why add `zabbix` to `adm`? | Ubuntu logs are `syslog:adm 640`, so the agent needs that group to read them. `adm` gives read-only log access, not sudo. |
| What is the NSG? | Azure's network firewall in front of the VM. |

## Packages

| Question | Answer |
|---|---|
| dpkg vs apt? Why the zabbix-release .deb? | Ubuntu has no agent2 and only Zabbix 6.0, so the .deb adds Zabbix's repo for 8.0. dpkg installs a local .deb, while apt downloads packages and resolves dependencies. |
| What is the `2:` in the version? | It's the epoch, which overrides version comparison. It makes Zabbix's package beat Ubuntu's `1:`. |
| `free` vs `available` memory? | Linux uses spare RAM as cache. "Available" is what apps can actually use. |

## Added during the build (Nov 12–13)

| Question | Answer |
|---|---|
| Agent-based or agentless? | Both. Agent2 sends metrics and logs, and the proxy pings each VM and checks port 80. |
| Can PSK work without a proxy? | Yes, agents can use PSK directly. We still need the proxy because Azure can't reach the LAN, and it buffers 24 h over one connection. |
| Why does the nginx status tile stay "active" when the VM is off? | No agent means no new value, so the last one stays. That's why web state uses the proxy's TCP check. |
| How do you tell VM down from agent down? | Ping up with no heartbeat means the agent died. Ping down means the VM is down. |
| What are trigger dependencies for? | One root-cause alert. Agent-down and web-down depend on VM-down, so a powered-off VM sends one email. |
| What is low-level discovery / a prototype? | Discovery finds disks, NICs and filesystems. Prototypes like `{#IFNAME}` are blueprints that create the per-device items and triggers. |
| Why `min(…,30s)>65` and not `last()>65`? | `last()` fires on a single spike. `min` over 30 s means it stayed above the threshold the whole time. |
| Why lower the CPU threshold to 65%? | The assignment gives 75% only as an example. 65% with a 30 s window fires reliably in a 90 s test. |
| How fast does an alert arrive? | Item interval (10 s) + agent buffer (up to 5 s) + proxy send (up to 1 s) + trigger window + email. For the 30 s metric alerts that's under a minute. |
| What happens if the internet drops? | The proxy buffers up to 24 h. It happened when our public IP changed, and the data arrived later. |
| How is the kernel log tested? | With a real kernel event, `ip link set ens18 promisc on/off`. Userspace writes to `/dev/kmsg` get labelled `user`, not kernel. |
| Why did email fail once? | Gmail rejected mail with `550` after many test emails. We switched to a new sender with an app password. |

## Known limitations (say these before he finds them)

Zabbix UI on HTTP and open to the internet (fix: Cloudflare Access) · NSG 10051 open to Any (PSK-only protects it; better: Tailscale link) · vendor template edited directly (clone/macros would survive upgrades) · no management VLAN · self-signed Proxmox cert · CT 103 is a single point of failure for teammate access · `adm` gives the agent read access to all logs.
