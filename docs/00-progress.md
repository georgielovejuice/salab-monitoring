# Progress log

**Course:** 01276521 Practice of System Administration, KMITL
**Assignment:** Network Monitoring System (35%), demo + Q&A with the instructor
**Planned submission / demo:** this **Thursday–Friday** (official deadline Nov 18)
**Team:** me (homelab owner, sa-web3), my teammate (sa-web1), my teammate (sa-web2), my teammate, my teammate (Azure VM owner)

> Big picture: what we're building, what's done, what's next. Details live in the topic files.

---

## Architecture

Three Ubuntu web servers run on a Proxmox homelab, each with a Zabbix agent. A Zabbix **proxy** on the same LAN collects their metrics and logs and sends everything over **one outbound, PSK-encrypted connection** to the **Zabbix server on Azure**, which stores the data, runs triggers, sends the email alerts and serves the dashboard. We use a proxy because the home internet is behind **CGNAT**: Azure can't connect in, so every connection must start from home.

```
HOME: Proxmox "kim-sever" (192.168.1.10), LAN 192.168.1.0/24, behind CGNAT
  VM 211 sa-web1   192.168.1.51   NGINX   + agent2 ─┐
  VM 212 sa-web2   192.168.1.52   Apache  + agent2 ─┤  passive: proxy → agent :10050
  VM 213 sa-web3   192.168.1.53   (TBD)   + agent2 ─┤  active:  agent → proxy :10051
                                                    ▼
  VM 214 sa-proxy  192.168.1.54   Zabbix proxy 8.0 (SQLite, 24 h buffer)
        │
        │  one outbound TCP 10051, PSK-encrypted
        ▼
AZURE VM <AZURE_PUBLIC_IP>: Zabbix server 8.0 + DB + web UI — triggers, email alerts, dashboards

Access: teammates → Tailscale (4via6 site 7, CT 103) → Proxmox UI :8006 / web VMs
Firewall: Proxmox firewall (Security Group sa-web, sa-proxy rules) + agent Server= allowlist
```

---

## Topic files

| File | What's in it |
|---|---|
| [01-assignment.md](01-assignment.md) | Requirements, marking scheme, the 8 alerts |
| [02-architecture.md](02-architecture.md) | Machines, why a proxy, data flow, templates, Plan B |
| [03-access.md](03-access.md) | Proxmox accounts, Tailscale policy + 4via6, Zabbix UI accounts |
| [04-firewall.md](04-firewall.md) | Proxmox firewall levels + rules, Azure NSG, agent allowlist, tests |
| [05-zabbix-proxy.md](05-zabbix-proxy.md) | How sa-proxy was built, command by command |
| [06-zabbix-agent.md](06-zabbix-agent.md) | **Runbook for sa-web2 / sa-web3** (each command explained) |
| [07-alerts.md](07-alerts.md) | Item keys, triggers and demo method for the 8 alerts |
| [08-cli-labs.md](08-cli-labs.md) | Terminal commands to watch every step (zabbix_get, tcpdump, sqlite, API) |
| [09-qa-cheatsheet.md](09-qa-cheatsheet.md) | Answers for the instructor Q&A |

---

## Status

| Component | State |
|---|---|
| Proxmox pool / accounts / Tailscale 4via6 | Done |
| Proxmox firewall | Done, tested |
| sa-proxy (VM 214) | Online, PSK |
| sa-web1 (VM 211) NGINX + agent2 | Monitored, ZBX green |
| sa-web2 (VM 212) | Host added in UI; Apache + agent not installed yet |
| sa-web3 (VM 213) | Not started |
| Alerts + email | Not started |

---

## Next

- **My teammate:** sa-web2 — Apache + agent2, following [06-zabbix-agent.md](06-zabbix-agent.md) steps 0–5.
- **Me:** sa-web3 — pick the web server, then the same runbook.

---

## What we've done (in order)

| What happened / decided | Why |
|---|---|
| First plan: Zabbix + Graylog on the homelab | Purpose-built tools |
| Remote access via Tailscale | No port forwarding |
| Switched to Prometheus + Loki + Grafana on GCP (later dropped) | Team familiarity |
| Hands-on rule: everyone builds by hand, every command explained | Q&A is 10 pts |
| Proxmox pool `salab`, group `salab-team`, PVEVMAdmin; VMs 211–213 created | Least privilege, one account per person |
| Tailscale policy restricting teammates to Proxmox UI + lab VMs | Default policy allowed the whole homelab |
| Subnet conflict (everyone on 192.168.1.0/24) fixed with **4via6 site 7** | Unique addresses without renumbering |
| GCP dropped (trial demanded a THB 1,000 prepayment) | Refunds unreliable |
| Teammate set up **Zabbix 8.0 on Azure**; team switched to Zabbix | Instructor's example tool; already running |
| Direct agent → Azure public IP tried, then dropped: home is behind **CGNAT** (shared IP <HOME_PUBLIC_IP>) | Allowlist admits strangers; Azure can't reach in |
| Design changed to **Zabbix proxy at home** (VM 214); Plan B = all at home | Outbound-only, PSK, buffering, standard pattern |
| sa-proxy built; PSK-only; proxy address <HOME_PUBLIC_IP>; proxy Online | — |
| Azure UI at `http://<AZURE_PUBLIC_IP>/`; HTTP exposure noted as a risk | — |
| Proxmox VM firewall on with no rules → everything blocked (default DROP). Fixed with node rule (8006), Security Group `sa-web`, sa-proxy rules | Least privilege at the hypervisor |
| Mistake: sa-proxy lacked a 10051 rule → agent `cannot connect to [192.168.1.54:10051]: i/o timeout`. Fixed: in tcp 10051 from .51,.52,.53 | Active checks are agent → proxy |
| **sa-web1 monitored** end to end (agent2 8.0 → sa-proxy → Azure), ZBX green. Tests: proxy→10050 OK, sa-web2→10050 timed out | First full path works |
| Host sa-web2 added in UI; after `config_cache_reload` it appeared in the proxy's SQLite | Proved how the proxy learns its hosts |
| Docs split into topic files; sa-web2/3 handed to teammates with the runbook | Teammates learn by doing it themselves |
