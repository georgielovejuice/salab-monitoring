# 07 — Alerts: thresholds and tests

Back to [README](../README.md) · [Progress](00-progress.md)

Status: **done and demonstrated.** Every alert tested end to end (problem → email → recovery). Email sender switched to a new Gmail + app password after Gmail rejected mail with `DATA failed: 550` (rate limit); one email per alert (duplicate media/operations removed).

## Final triggers (built, naming rule applied)

Rule: Name `[X] What happened` · Event name `[X] {HOST.NAME}: what happened (threshold)` · Operational data = current value · Severity: High = down/security, Average = resource, Warning = error log · Tags `scope: salab`, `alert: X`.

| Alert | Name | Expression (template `Linux by Zabbix agent active` unless noted) | Severity |
|---|---|---|---|
| UP | `[UP] VM is down` | `max(/ICMP Ping/icmpping,#3)=0` (ICMP Ping template, ping from proxy every 10s) | High |
| UP | `[UP] Zabbix agent not responding` | `nodata(/…/agent.ping,3m)=1` · depends on `[UP] VM is down` | High |
| A | `[A] Unauthorized sudo attempts` | `count(/…/log[/var/log/auth.log,"sudo.*(authentication failure\|incorrect password\|NOT in sudoers\|command not allowed)"],5m)>=3` | High |
| B | `[B] Web server error log` | per host: `count(/<host>/log[<nginx\|apache2 error.log>],2m,"regexp","…(error\|crit\|alert\|emerg)…")>0` | Warning |
| C | `[C] Web service is down` | `max(/…/net.tcp.service[http],45s)=0` (simple check from proxy) · depends on `[UP] VM is down`. 45 s (not 30 s) so VM-down always fires first; at 30 s the web check won the race and both emails went out | High |
| D | `[D] High CPU usage` | `min(/…/system.cpu.util,30s)>65` (65% and 30 s so the test fires reliably; assignment says "75% for example") | Average |
| E | `[E] High memory usage` | `max(/…/vm.memory.size[pavailable],1m)<25` | Average |
| F | `[F] High inbound traffic on {#IFNAME}` | trigger prototype: `min(/…/net.if.get.in["{#IFNAME}"],30s)>100M` | Average |
| G | `[G] High disk write on {#DEVNAME}` | trigger prototype: `min(/…/vfs.dev.write[{#DEVNAME}],30s)>39321600` (37.5 MiB/s) | Average |
| G | `[G] High disk read on {#DEVNAME}` | trigger prototype: `min(/…/vfs.dev.read[{#DEVNAME}],30s)>39321600` (marking item 10 = read **and** write) | Average |
| H | `[H] Low disk space` | `last(/…/vfs.fs.size[/,pfree])<20` (built-in "Space is low" at 90% used stays as second level) | Average |

Disabled (kept for rollback): hand-made per-host F/G duplicates, old `Web Service Stop` (systemd), vmware cache trigger on sa-proxy. ICMP Ping also linked to sa-proxy (needed for the dependency).

## Lessons from testing

- `last()` fires on one 10 s sample; `min(…,30s)` = above the threshold the whole window. 1-min windows were too slow for a 60–90 s test.
- Test scripts must run longer than the trigger window (≥ 90 s); fio needs `--time_based --runtime=90` or it finishes in seconds.
- New trigger prototypes reach hosts only when discovery re-runs; dependent discovery rules discard unchanged data for 1 h → `sudo systemctl restart zabbix-proxy` forces it.
- Dependencies only suppress a child if the parent is already a problem → give the child a longer window.
- Userspace can't write kernel-facility logs (`/dev/kmsg` from userspace is relabelled `user`) → real kernel event: `ip link set ens18 promisc on/off`.
- Agent items (`systemd.unit.info`) go stale when the VM dies → web state alert uses the proxy's TCP check.

## Test commands

The demo versions of these live in `~/test-load/` on every web VM, see [11-demo.md](11-demo.md). Run each test for at least 90 s so it outlasts the trigger window.

Tools: `sudo apt install -y stress-ng apache2-utils fio iperf3` (`ab` is in `apache2-utils`).

| Alert | Command (on the web VM unless noted) | Expect |
|---|---|---|
| D CPU | `stress-ng --cpu 4 --cpu-load 100 --timeout 90s` | CPU > 65% |
| E memory | `stress-ng --vm 1 --vm-bytes 80% --timeout 90s` | Available memory < 25% |
| F network in | on target: `iperf3 -s` · from another VM: `iperf3 -c 192.168.1.51 -t 90` | Bits received > 100 Mbps |
| G disk write | `fio --name=disk-write-test --filename=/tmp/zabbix-testfile --size=512M --rw=write --bs=1M --ioengine=libaio --direct=1 --iodepth=16 --numjobs=1 --runtime=90 --time_based` | Write > 37.5 MiB/s |
| G disk read | same with `--name=disk-read-test --rw=read` | Read > 37.5 MiB/s |
| H disk free | `df -h /` then `sudo fallocate -l <size> /fill` (enough to pass 80% used) · cleanup `sudo rm /fill` | Free < 20% |
| C web down | `sudo systemctl stop nginx` (Apache: `apache2`) · then `start` | Problem, then resolved |
| Uptime | `sudo reboot` | "Host has been restarted" |
| B web error (NGINX) | `curl http://localhost/does-not-exist` (missing file is logged as `[error]` in error.log) | New error line + problem |
| B web error (Apache) | `curl --path-as-is "http://localhost/../../etc/passwd"` → `[core:error]` invalid URI (verify on sa-web2) | New error line |
| A sudo, wrong password | `for i in 1 2 3; do echo wrong \| sudo -S -k true; done` | 3 "incorrect password" lines |
| A sudo, not in sudoers | `sudo adduser --gecos "" intruder` → `su - intruder` → `sudo ls` | "user NOT in sudoers" |
| fio cleanup | `rm /tmp/zabbix-testfile` | — |

Command notes:
- `fio --direct=1` bypasses the RAM cache so the disk really works; `--time_based --runtime=90` runs for exactly 90 s.
- `sudo -S` reads the password from stdin, `-k` forgets cached credentials so every try is a real attempt.
- **`ab -n 10000 -c 50 http://127.0.0.1/` does NOT test network in**: 127.0.0.1 is loopback (`lo`), which the template doesn't monitor. It does fill the access log and load CPU. For network in, traffic must arrive on `ens18` from another machine (iperf3, or `ab` against the VM's LAN IP from another VM).
- iperf3 uses tcp 5201, which Security Group `sa-web` allows from `192.168.1.0/24`.

Demo plan: [11-demo.md](11-demo.md). Timing: [10-timing.md](10-timing.md).
