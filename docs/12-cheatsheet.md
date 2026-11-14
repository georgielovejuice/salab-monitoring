# 12 — Command cheat sheet (demo)

Back to [README](../README.md) · [Progress](00-progress.md)

## Status (any VM)
| Command | Shows |
|---|---|
| `systemctl status zabbix-agent2` | Agent running? |
| `systemctl status nginx` (sa-web2: `apache2`) | Web server running? |
| `systemctl status zabbix-proxy` (sa-proxy) | Proxy running? |
| `grep -E '^(Server\|ServerActive\|Hostname)=' /etc/zabbix/zabbix_agent2.conf` | Agent points to proxy .54, correct hostname |
| `ip -br a` | Interfaces + IP |
| `df -h /` · `free -h` · `top` | Disk · memory · CPU |

## Logs
| Command | Shows |
|---|---|
| `sudo tail -f /var/log/zabbix/zabbix_agent2.log` | Agent live log |
| `sudo tail -f /var/log/zabbix/zabbix_proxy.log` | Proxy live log (`received configuration data`) |
| `sudo tail -n 5 /var/log/auth.log` | sudo / ssh logins (alert A) |
| `sudo tail -n 5 /var/log/kern.log` | Kernel log |
| `sudo tail -n 5 /var/log/nginx/error.log` / `access.log` | Web logs (alert B) |

## Proxy / connections (sa-proxy)
| Command | Shows |
|---|---|
| `sudo ss -tnp \| grep 10051` | Agents in (.51–.55) + one connection out to Azure |
| `zabbix_get -s 192.168.1.51 -k agent.ping` | Ask an agent directly (1 = alive) |
| `nc -zv 192.168.1.51 80` | Is port 80 open (what the HTTP check does) |
| `sudo zabbix_proxy -R config_cache_reload` | Pull config from server now |
| `sudo sqlite3 /var/lib/zabbix/zabbix_proxy.db "SELECT host FROM hosts;"` | Hosts the proxy got from the server |
| `ps aux \| grep "[i]cmp pinger"` | 2 pinger processes (StartPingers=2) |

## Trigger each alert (`~/test-load/`)
| Alert | Command |
|---|---|
| D CPU | `./cpu_stress.sh` |
| E Memory | `./memory_stress.sh` |
| G Disk read/write | `./disk_rw_stress.sh` |
| F Network in | `./inbound_stress.sh` |
| H Disk space | `./disk_fill.sh` (Enter deletes the file) |
| C Web down | `./web_down.sh` (or `sudo systemctl stop nginx`) |
| B Web error log | `./nginx_err.sh` (sa-web2: `curl -I http://127.0.0.1/secret/`) |
| A Sudo | `for i in 1 2 3; do echo wrong \| sudo -S -k true; done` |
| Kernel log | `./kernel_log.sh` (promisc on/off → real kernel message) |
| Agent down | `sudo systemctl stop zabbix-agent2` → ~3 min → `start` |
| VM down | Proxmox → Shutdown → ~30 s → Start |

## Cleanup / check after
| Command | Why |
|---|---|
| `sudo systemctl start nginx zabbix-agent2` | Make sure services are back |
| `df -h /` | Disk back to normal (fill file removed) |
| `systemctl is-active nginx zabbix-agent2` | Both `active` |
