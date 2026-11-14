# 11 — Demo (done Nov 13, 2026 — went well)

Back to [README](../README.md) · [Progress](00-progress.md)

## How we ran it

- Simple bash scripts in `~/test-load/` on every web VM (copied from sa-web4 with `scp -rp`). Rule: keep scripts simple enough to explain line by line.
- Screen: Zabbix Problems page + Global view (VM and web honeycombs, metric graphs, log widgets) + inbox, terminal per VM.
- Independent tests run in parallel on different VMs.

## Scripts (`~/test-load/`)

| Script | Alert | Command inside |
|---|---|---|
| `cpu_stress.sh` | D | `stress-ng --cpu 4 --cpu-load 100 --timeout 90s` |
| `memory_stress.sh` | E | `stress-ng --vm 1 --vm-bytes 1700M --vm-keep --timeout 90s` |
| `disk_rw_stress.sh` | G (write + read) | `fio … --rw=write … --runtime=90 --time_based` then `--rw=read … --runtime=60 --time_based`, then `rm` |
| `inbound_stress.sh` | F | 8 parallel `curl` downloads of a 100 MB file (internet) — LAN alternative: `iperf3 -c 192.168.1.51 -R -t 90` |
| `disk_fill.sh` | H | `sudo fallocate -l 7G /fill.bin`, Enter deletes it |
| `web_down.sh` | C | `systemctl stop nginx` (sa-web2: `apache2`), `sleep 60`, `start` |
| `nginx_err.sh` | B | empty dir without index → nginx logs `[error] directory index … is forbidden` (sa-web2: `curl -I http://127.0.0.1/secret/` → `[authz_core:error]`) |
| `sudo_fail.sh` | A | `for i in 1 2 3; do echo wrong \| sudo -S -k true; done` |
| `kernel_log.sh` | kernel log | `ip link set ens18 promisc on` / `off` → real kernel message in `/var/log/kern.log` |
| `agent_down.sh` | UP (agent) | `systemctl stop zabbix-agent2`, `sleep 240`, `start` |
| (Proxmox) | UP (VM) | Shutdown → ~30 s → `[UP] VM is down` only → Start |

Commands: [12-cheatsheet.md](12-cheatsheet.md).
