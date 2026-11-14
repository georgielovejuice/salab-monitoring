# 01 — Assignment

Back to [README](../README.md) · [Progress](00-progress.md)

- At least **4 VMs** (personal computer or public cloud).
- Log centralization + network monitoring: **open source, installed manually on Linux**. No cloud-based or managed monitoring services.
- Collect from **at least 3 Linux VMs**, each running a **web server**.
- Demo + instructor Q&A by Nov 18 (official). **Done on Nov 13, 2026.**

| # | Marking item | Pts |
|---|---|---|
| 1 | Access to the log centralization system | 1 |
| 2 | Access to the network monitoring system | 1 |
| 3 | System logs (auth, kernel) + alert | 2 |
| 4 | Web server access/error logs + alert | 2 |
| 5 | Uptime + alert | 2 |
| 6 | Web service state + alert | 2 |
| 7 | CPU + alert | 3 |
| 8 | Memory + alert | 3 |
| 9 | Network in/out rate + alert | 3 |
| 10 | Disk read/write throughput + alert | 3 |
| 11 | Disk free space + alert | 3 |
| 12 | **Instructor Q&A** | **10** |

## Required email alerts

| ID | Alert |
|---|---|
| A | Unauthorized sudo usage |
| B | Web server error/critical log |
| C | Web service stopped |
| D | CPU > 75% |
| E | Memory > 75% |
| F | Network inbound > rate |
| G | Disk write > rate |
| H | Disk free < threshold |

How each is implemented: [07-alerts.md](07-alerts.md).

## How our design maps to the requirements

| Requirement | Ours |
|---|---|
| ≥ 4 VMs | sa-web1, sa-web2, sa-web3, sa-web4, sa-proxy (Proxmox) + Zabbix server (Azure) = 6 |
| Log centralization system | Zabbix server (log items from auth/kern/web access/web error logs stored centrally) |
| Network monitoring system | Zabbix server (metrics, availability from the proxy, triggers, email) |
| Open source, manual install | Zabbix 8.0: proxy and agents from the official repo, server with Docker Compose, all set up by hand |
| ≥ 3 Linux VMs with web servers | sa-web1 NGINX, sa-web2 Apache, sa-web3 NGINX, sa-web4 NGINX |
