# Progress log

**Course:** 01276521 Practice of System Administration, KMITL
**Assignment:** Network Monitoring System (35%), demo + Q&A with the instructor
**Status:** **DONE. Demo + Q&A completed (Nov 13, 2026) and went well.** Official deadline was Nov 18.
**Team:** 4 students (homelab + proxy + firewall, one web VM each, Azure server)

> Big picture of what we built and every decision along the way. Details live in the topic files.

---

## Architecture (final)

Four Ubuntu web servers run on a Proxmox homelab, each with Zabbix agent2. A Zabbix **proxy** on the same LAN collects their metrics and logs (agent active checks) and also probes them from outside (ICMP ping + TCP port 80). It sends everything over **one outbound, PSK-encrypted connection** to the **Zabbix server on Azure**, which stores the data, runs triggers, sends email alerts and serves the dashboards. We use a proxy because the home internet is behind **CGNAT**: Azure can't connect in, so every connection must start from home.

```
HOME: Proxmox "kim-sever" (192.168.1.10), LAN 192.168.1.0/24, behind CGNAT
  VM 211 sa-web1   .51  NGINX  + agent2 ─┐  agent -> proxy :10051 (active checks, heartbeat)
  VM 212 sa-web2   .52  Apache + agent2 ─┤  proxy -> VM  ICMP ping (VM up/down)
  VM 213 sa-web3   .53  NGINX  + agent2 ─┤  proxy -> VM  TCP :80   (web up/down)
  VM 215 sa-web4   .55  NGINX  + agent2 ─┘
                     ▼
  VM 214 sa-proxy  .54  Zabbix proxy 8.0 (SQLite, 24 h buffer, StartPingers=2)
                        + its own agent2 (monitored as a host too)
        │  one outbound TCP 10051, PSK-encrypted
        ▼
AZURE VM: Zabbix server 8.0 (Docker Compose) + DB + web UI — triggers, email alerts, dashboards

Access: teammates -> Tailscale (4via6 site 7, CT 103) -> Proxmox UI :8006 / lab VMs
Firewall: Proxmox firewall (Security Group sa-web, sa-proxy rules) + agent Server= allowlist + Azure NSG
```

**Monitoring style:** agent-based for metrics and logs (inside view) + agentless checks from the proxy for availability (outside view). Together they tell "VM down" apart from "only the agent/web server died".

**Log centralization:** Zabbix log items (auth, kernel, web access, web error logs from every VM) shown and searchable in the Zabbix UI (Global view widgets + Latest data → History filter). rsyslog forwarding was considered but not used.

---

## Topic files

| File | What's in it |
|---|---|
| [01-assignment.md](01-assignment.md) | Requirements, marking scheme, how the design meets them |
| [02-architecture.md](02-architecture.md) | Machines, why a proxy, data flow, templates, inside vs outside checks |
| [03-access.md](03-access.md) | Proxmox accounts, Tailscale policy + 4via6, Zabbix UI accounts |
| [04-firewall.md](04-firewall.md) | Proxmox firewall levels + rules, Azure NSG, agent allowlist, tests |
| [05-zabbix-proxy.md](05-zabbix-proxy.md) | How sa-proxy was built, command by command |
| [06-zabbix-agent.md](06-zabbix-agent.md) | Runbook: add a web VM to monitoring (each command explained) |
| [07-alerts.md](07-alerts.md) | Final triggers (naming rule, expressions, severities, dependencies) + tests |
| [08-cli-labs.md](08-cli-labs.md) | Terminal commands to watch every step (zabbix_get, tcpdump, sqlite, API) |
| [09-qa-cheatsheet.md](09-qa-cheatsheet.md) | Answers for the instructor Q&A |
| [10-timing.md](10-timing.md) | Every timer: agent ⇄ proxy ⇄ server, item intervals, dashboard refresh |
| [11-demo.md](11-demo.md) | How the demo was run, test scripts |
| [12-cheatsheet.md](12-cheatsheet.md) | Command cheat sheet |

---

## Final status

| Component | State |
|---|---|
| Proxmox pool `salab` / per-person accounts / Tailscale 4via6 | Done |
| Proxmox firewall (Security Group `sa-web` on 211, 212, 213, 215; sa-proxy rules) | Done, tested |
| sa-proxy (VM 214) | Online, PSK; monitored as a host (Linux active + Zabbix proxy health + ICMP Ping) |
| sa-web1–4 | All monitored (template Linux by Zabbix agent active + ICMP Ping + HTTP simple check) |
| Global view dashboard (metrics, logs, VM + web honeycombs) | Done, 10 s widget refresh |
| Triggers: all 8 alerts + VM down + agent down | Done, one naming rule, dependencies |
| Email alerts | Working (switched to a new Gmail sender + app password after Gmail 550 rate limit) |
| Test scripts `~/test-load/` | On every web VM |
| Demo + Q&A | Done, went well |
| Plan 2 (private all-local Zabbix sandbox, sa-zbx) | Dropped |

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
| Teammate set up **Zabbix 8.0 on Azure** (Docker Compose); team switched to Zabbix | Instructor's example tool; already running |
| Direct agent → Azure public IP tried, then dropped: home is behind **CGNAT** | Allowlist admits strangers; Azure can't reach in |
| Design changed to **Zabbix proxy at home** (VM 214) | Outbound-only, PSK, buffering, standard pattern |
| sa-proxy built; PSK-only; proxy Online | — |
| Proxmox VM firewall on with no rules → everything blocked (default DROP). Fixed with node rule (8006), Security Group `sa-web`, sa-proxy rules | Least privilege at the hypervisor |
| Mistake: sa-proxy lacked a 10051 rule → agent `i/o timeout`. Fixed: in tcp 10051 from the web VMs | Active checks are agent → proxy |
| **sa-web1 monitored** end to end. Tests: proxy→10050 OK, sa-web2→10050 timed out | First full path works |
| Docs split into topic files; teammates did sa-web2/3/4 with the runbook | Learn by doing |
| Home public IP changed → proxy Offline 3.5 h, data buffered and delivered later. Fix: NSG 10051 source = Any, Proxy address cleared; PSK is the authentication | CGNAT IP is dynamic |
| sa-web4 (VM 215, user member account) added: agent first pointed at Azure and installed from the wrong repo (7.0) → repointed to proxy, upgraded to 8.0; static IP .55; firewall extended | All agents through the proxy, same version |
| Hosts switched to template **Linux by Zabbix agent active** | Push model, matches log items |
| Item intervals tuned for live demo (CPU/memory/network/disk masters 10 s, FS 30 s) + widget refresh 10 s | Responsive dashboard without overloading the server |
| sa-proxy monitored as a host (agent2 → 127.0.0.1, Zabbix proxy health template) | Who monitors the proxy |
| Outside-in checks: **ICMP Ping** template (fping on sa-proxy, 10 s) + simple check `net.tcp.service[http]` (10 s, value map Service state). `StartPingers=2` (icmp pinger hit 61%) | Agent items go stale when a VM dies; proxy probes don't |
| Old systemd-based web triggers replaced by the HTTP check | They couldn't detect a powered-off VM |
| All triggers cleaned up: one naming rule, 30 s windows, F/G as discovery trigger prototypes (+ disk read trigger), duplicates disabled, dependencies (agent + web → VM down), web window 45 s so VM-down always fires first | One readable alert per problem |
| Test scripts `~/test-load/` fixed (runtime ≥ 90 s, fio `--time_based`, real kernel message via promisc on/off) and copied to all web VMs | Every alert reliably fires |
| Gmail SMTP `DATA failed: 550` (rate limit after testing) → new Gmail sender + app password, one email per alert | Mail delivery |
| **Demo + Q&A done, went well** | — |
| Plan 2 (all-local sandbox) dropped | Project complete |
