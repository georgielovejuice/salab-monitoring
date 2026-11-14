# 02 — Architecture

Back to [README](../README.md) · [Progress](00-progress.md)

## Overview

```
HOME: Proxmox "kim-sever" (192.168.1.10), LAN 192.168.1.0/24, behind ISP CGNAT
  VM 211 sa-web1   192.168.1.51   Ubuntu 24.04 + NGINX     + agent2
  VM 212 sa-web2   192.168.1.52   Ubuntu 24.04 + Apache    + agent2
  VM 213 sa-web3   192.168.1.53   Ubuntu 24.04 + NGINX     + agent2
  VM 215 sa-web4   192.168.1.55   Ubuntu 24.04 + NGINX     + agent2
  VM 214 sa-proxy  192.168.1.54   Ubuntu 24.04 + Zabbix proxy 8.0 (SQLite) + agent2
        |
        |  agents push to the proxy (active checks, :10051)
        |  proxy pings each VM + checks TCP :80 (agentless, from outside)
        v
  sa-proxy --- one outbound TCP 10051 connection, PSK-encrypted ---> AZURE

AZURE VM <AZURE_PUBLIC_IP> (managed by a teammate)
  Zabbix server 8.0.0 + web UI, installed with Docker Compose, UI at http://<AZURE_PUBLIC_IP>/
  Database, triggers, alert emails, dashboards
```

## Machines

| VM | Name | IP | Spec | OS user | Web server | Status |
|---|---|---|---|---|---|---|
| 211 | sa-web1 | 192.168.1.51 | 2 vCPU, 2 GB (no balloon) | member account | NGINX | Monitored |
| 212 | sa-web2 | 192.168.1.52 | same | member account | Apache HTTP | Monitored |
| 213 | sa-web3 | 192.168.1.53 | same | member account | NGINX | Monitored |
| 215 | sa-web4 | 192.168.1.55 | same | member account | NGINX | Monitored |
| 214 | sa-proxy | 192.168.1.54 | 1 vCPU, 1 GB (no balloon), 15 GB | member account | none | Proxy online + monitored as a host |
| Azure | Zabbix server | <AZURE_PUBLIC_IP> | — | teammate | nginx (UI) | Zabbix 8.0.0 |

- All lab VMs are in Proxmox pool **`salab`**.
- VMs, not LXC containers: each has its own kernel, so `/proc` metrics (CPU, memory, disk) are real.
- Ballooning off: fixed RAM keeps the memory alert threshold meaningful.
- Timezone on every VM: `sudo timedatectl set-timezone Asia/Bangkok`.

## Why a Zabbix proxy (core design decision)

| Problem | How the proxy solves it |
|---|---|
| Home is behind **CGNAT**: Azure can't connect in, and public IP `<HOME_PUBLIC_IP>` is shared with other ISP customers | The proxy only makes **outbound** connections. **PSK** authenticates it regardless of the shared IP |
| Passive checks impossible across NAT | Proxy and agents share a LAN, so passive and active checks both work |
| Agent traffic would cross the internet in plaintext | Agent traffic stays in the house; only the encrypted proxy link leaves |
| Home internet drops | Proxy **buffers up to 24 h** (`ProxyOfflineBuffer=24`) and sends later |

Standard Zabbix pattern for remote sites / branch offices.

## Data flow, step by step

```
   HOME                                                        AZURE
 sa-web1 agent2            sa-proxy .54                    Zabbix server
                                  │ ② "any job for me?" ─────────►│  ① host added in UI → server DB
                                  │◄──────── config ───────────────│
        │◄── ③a "CPU value?" ─────│  proxy → agent :10050          │
        │──── value ─────────────►│                                │
        │── ③b "my check list?" ─►│  agent → proxy :10051          │
        │── log lines, pushed ───►│                                │
                                  │ ④ buffer in SQLite             │
                                  │──── batch of values ──────────►│  ⑤ save, check triggers → email
                                  │   proxy → server :10051, PSK   │  ⑥ UI draws dashboards
```

| Step | Who starts | What happens |
|---|---|---|
| ① | You | Host added in UI → stored in server DB |
| ② | Proxy → server :10051 | Proxy identifies as `sa-proxy` (Hostname + PSK identity), downloads only hosts whose **Monitored by = sa-proxy**, with interfaces and items |
| ③a Passive | Proxy → agent :10050 | Proxy asks for one value at a time. Final setup uses the active template, so passive is only used for `zabbix_get` debugging |
| ③b Active | Agent → proxy :10051 | Agent says "I'm `sa-web1`", gets its check list, collects itself (logs), pushes results. Unknown name → `host [sa-web1] not found` |
| ④ | Proxy | Values stored in local SQLite, buffered if Azure unreachable |
| ⑤ | Proxy → server :10051 | Batches sent (itemid, time, value); server saves, evaluates triggers → action → email |
| ⑥ | UI | Dashboards read the server DB |

**Every connection to Azure starts at home**, so CGNAT doesn't matter.

### How the proxy knows which hosts to collect
The host form's **Monitored by → Proxy → sa-proxy** is stored on the server. The server sends each proxy only its own hosts. The proxy saves a copy in `/var/lib/zabbix/zabbix_proxy.db` (tables `hosts`, `interface`, `items`). Verified Nov 10: after adding sa-web2 and running `zabbix_proxy -R config_cache_reload`, sa-web2 appeared in the proxy's `hosts` table. Commands: [08-cli-labs.md](08-cli-labs.md).

## Templates (where the default metrics come from)

We linked the built-in template **Linux by Zabbix agent active** (first **Linux by Zabbix agent**, the passive version). A template is a bundle copied onto the host:

| Part | Example |
|---|---|
| Items (key + interval) | `system.cpu.util`, `vm.memory.size[pavailable]` |
| Triggers | High CPU utilization, Lack of available memory, low disk space |
| Discovery rules (LLD) | Finds every disk, filesystem and NIC and creates items |
| Graphs / dashboards | "Linux Dashboard" on the host |

Final triggers: see [07-alerts.md](07-alerts.md). Template items we changed: intervals of CPU idle/user, available memory (+%), load 1m, `Get network interfaces`, `Get block devices`, `{#DEVNAME}: Get stats` (10 s), `FS [{#FSNAME}]: Get data` (30 s); added `net.tcp.service[http]` simple check and value map `Service state`. Note: editing the vendor template means a re-import would overwrite it; cloning or macros is the cleaner production approach.

## Inside vs outside checks (agent-based + agentless)

| View | Who checks | Items | When the VM is powered off |
|---|---|---|---|
| Inside (agent-based) | agent2 on the VM, active checks | CPU, memory, disk, network, uptime, all logs | Goes silent; widgets keep the last (stale) value |
| Outside (agentless) | sa-proxy | `icmpping` (ICMP Ping template, 10 s, fping, `StartPingers=2`) · simple check `net.tcp.service[http]` (10 s, value map Service state 0 Down / 1 Up) | Reports **Down** |

| Ping | Agent heartbeat | Diagnosis |
|---|---|---|
| Up | available | All fine |
| Up | not available | VM fine, agent crashed/stopped |
| Down | not available | VM down |

Global view shows two honeycombs: **VM state (ping from proxy)** and **Web service (HTTP from proxy)**.

## Plan B: everything at home

If Azure fails: build a Zabbix server VM on Proxmox and change **one line** on the proxy (`Server=`). Agents never change. Keep exports of hosts/templates (Data collection → Hosts → Export).
