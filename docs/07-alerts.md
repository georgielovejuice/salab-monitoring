# 07 — Alerts (plan, not built yet)

Back to [README](../README.md) · [Progress](00-progress.md)

| Alert | Item idea | Trigger idea | Demo trigger | Source |
|---|---|---|---|---|
| A sudo abuse | `log[/var/log/auth.log,"NOT in sudoers"]` | ≥ 3 matches in 5 min | non-sudo user runs `sudo ls` 3× | custom |
| B web errors | `log[/var/log/nginx/error.log,"\[(error\|crit)\]"]` (Apache: `/var/log/apache2/error.log`) | any new match | `curl` a broken location | custom |
| C web down | `systemd.unit.info[nginx.service]` / `net.tcp.service[http]` | not active / 0 | `systemctl stop nginx` | custom |
| Uptime | `system.uptime`, agent availability | uptime < 10 min / agent unavailable | reboot / shut down VM | template |
| D CPU | `system.cpu.util` | `min(...,1m) > 75` | `stress-ng --cpu 0 -t 300` | template (tune) |
| E memory | `vm.memory.utilization` | `min(...,1m) > 75` | `stress-ng --vm 1 --vm-bytes 80% -t 300` | template (tune) |
| F network in | `net.if.in[ens18]` | > set rate | `iperf3` from another VM | custom |
| G disk write | `vfs.dev.write.rate[sda]` | > set rate | `dd if=/dev/zero of=/tmp/t bs=1M count=3000 oflag=direct` | custom |
| H disk free | `vfs.fs.size[/,pfree]` | < 20 | `fallocate -l 8G /fill` | template (tune) |

- `log[]` items are **active checks** (agent reads the file and pushes new lines) → need `ServerActive` + the sa-proxy 10051 rule.
- Demo tips: short item intervals (10–30 s) and `min(...,1m)` instead of 5-min averages, so emails arrive within ~2 min.
- Email: Alerts → Media types → Email (SMTP 587), user media, then Alerts → Actions → Trigger actions.
- Kernel log (marking item 3) also needed: `/var/log/kern.log`.

Test keys locally before building items: `sudo zabbix_agent2 -t 'vfs.fs.size[/,pfree]'` (see [08-cli-labs.md](08-cli-labs.md)).
