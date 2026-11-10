# 02 — Architecture

Back to [README](../README.md) · [Progress](00-progress.md)

## Overview

```
HOME: Proxmox "kim-sever" (192.168.1.10), LAN 192.168.1.0/24, behind ISP CGNAT
  VM 211 sa-web1   192.168.1.51   Ubuntu 24.04 + NGINX     + agent2
  VM 212 sa-web2   192.168.1.52   Ubuntu 24.04 + Apache    + agent2
  VM 213 sa-web3   192.168.1.53   Ubuntu 24.04 + (TBD)     + agent2
  VM 214 sa-proxy  192.168.1.54   Ubuntu 24.04 + Zabbix proxy 8.0 (SQLite)
        |
        |  agents talk ONLY to the proxy, over the LAN (10050 / 10051)
        v
  sa-proxy --- one outbound TCP 10051 connection, PSK-encrypted ---> AZURE

AZURE VM <AZURE_PUBLIC_IP> (managed by a teammate)
  Zabbix server 8.0.0 + web UI (nginx + PHP), UI at http://<AZURE_PUBLIC_IP>/
  Database, triggers, alert emails, dashboards
```

## Machines

| VM | Name | IP | Spec | OS login | Web server | Status |
|---|---|---|---|---|---|---|
| 211 | sa-web1 | 192.168.1.51 | 2 vCPU, 2 GB (no balloon), 20 GB | member account | NGINX | Monitored, ZBX green |
| 212 | sa-web2 | 192.168.1.52 | same | member account | Apache HTTP (planned) | Host in UI, agent todo |
| 213 | sa-web3 | 192.168.1.53 | same | member account | TBD — recommend NGINX (standard error.log with `[error]`/`[crit]`) | Ubuntu only |
| 214 | sa-proxy | 192.168.1.54 | 1 vCPU, 1 GB (no balloon), 15 GB | admin account | none | Proxy online |
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
| ③a Passive | Proxy → agent :10050 | Proxy asks for one value at a time |
| ③b Active | Agent → proxy :10051 | Agent says "I'm `sa-web1`", gets its check list, collects itself (logs), pushes results. Unknown name → `host [sa-web1] not found` |
| ④ | Proxy | Values stored in local SQLite, buffered if Azure unreachable |
| ⑤ | Proxy → server :10051 | Batches sent (itemid, time, value); server saves, evaluates triggers → action → email |
| ⑥ | UI | Dashboards read the server DB |

**Every connection to Azure starts at home**, so CGNAT doesn't matter.

### How the proxy knows which hosts to collect
The host form's **Monitored by → Proxy → sa-proxy** is stored on the server. The server sends each proxy only its own hosts. The proxy saves a copy in `/var/lib/zabbix/zabbix_proxy.db` (tables `hosts`, `interface`, `items`). Verified Nov 10: after adding sa-web2 and running `zabbix_proxy -R config_cache_reload`, sa-web2 appeared in the proxy's `hosts` table. Commands: [08-cli-labs.md](08-cli-labs.md).

## Templates (where the default metrics come from)

We linked the built-in template **Linux by Zabbix agent**. A template is a bundle copied onto the host:

| Part | Example |
|---|---|
| Items (key + interval) | `system.cpu.util`, `vm.memory.size[pavailable]` |
| Triggers | High CPU utilization, Lack of available memory, low disk space |
| Discovery rules (LLD) | Finds every disk, filesystem and NIC and creates items |
| Graphs / dashboards | "Linux Dashboard" on the host |

Alerts D, E, H already exist in the template (we tune thresholds). A, B, C, F, G need our own items/triggers.

## Plan B: everything at home

If Azure fails: build a Zabbix server VM on Proxmox and change **one line** on the proxy (`Server=`). Agents never change. Keep exports of hosts/templates (Data collection → Hosts → Export).
