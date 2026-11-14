# 06 — Zabbix agent2 runbook (add a web VM)

Back to [README](../README.md) · [Progress](00-progress.md)

Used for sa-web1 → sa-web4 (all done). Replace `X` with the VM number. Exception: sa-web4 is `192.168.1.55`, because .54 is sa-proxy. **Understand each line — the instructor will ask.**

Pitfalls we actually hit:
- sa-web4 first had `Server=`/`ServerActive=` pointing at **Azure** → must point at the proxy `192.168.1.54`.
- sa-web4 had agent **7.0** from the wrong repo → discovery rules "Not supported (invalid number of parameters)", 51 instead of ~89 items. Fix: add the 8.0 repo, `apt install zabbix-agent2`, answer **N** (keep your config) at the conffile prompt, restart.
- New VM left on DHCP → set a static IP.
- Hosts now use template **Linux by Zabbix agent active** (+ ICMP Ping); `ServerActive` and `Hostname` are the lines that matter.

| VM | Hostname | IP | Web server | Error log |
|---|---|---|---|---|
| sa-web1 | `sa-web1` | 192.168.1.51 | NGINX | `/var/log/nginx/error.log` |
| sa-web2 | `sa-web2` | 192.168.1.52 | Apache | `/var/log/apache2/error.log` |
| sa-web3 | `sa-web3` | 192.168.1.53 | NGINX | `/var/log/nginx/error.log` |
| sa-web4 | `sa-web4` | 192.168.1.55 | NGINX | `/var/log/nginx/error.log` |

## 0. Web server + time zone

```bash
sudo timedatectl set-timezone Asia/Bangkok
sudo apt update && sudo apt install -y apache2      # sa-web2 (sa-web1/3/4: nginx)
systemctl status apache2 --no-pager
curl -I localhost
```
- `timedatectl`: log times must match alert times.
- `curl -I`: requests only the headers; expect `HTTP/1.1 200 OK`.

## 1. Add the Zabbix 8.0 repo

```bash
wget https://repo.zabbix.com/zabbix/8.0/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_8.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_8.0+ubuntu24.04_all.deb
sudo apt update
apt-cache policy zabbix-agent2
```
- Ubuntu's repo has **no** `zabbix-agent2` and only Zabbix 6.0 — without this, `apt install` says "Unable to locate package".
- The `.deb` is not the agent: it installs a repo file in `/etc/apt/sources.list.d/` + Zabbix's GPG key. Check with `dpkg -L zabbix-release`.
- `dpkg -i` (= `--install`) installs a local .deb. `apt` downloads from repos and calls dpkg underneath.
- `apt update` makes apt read the new repo. `apt-cache policy` should show Candidate `2:8.0.0-1+ubuntu24.04` (`2:` = epoch, beats Ubuntu's `1:`).

## 2. Install

```bash
sudo apt install -y zabbix-agent2
```
Agent2 = the Go rewrite of the agent with built-in plugins.

## 3. Point it at the proxy

```bash
sudo cp /etc/zabbix/zabbix_agent2.conf /etc/zabbix/zabbix_agent2.conf.bak
sudo sed -i 's/^Server=127.0.0.1/Server=192.168.1.54/; s/^ServerActive=127.0.0.1/ServerActive=192.168.1.54/; s/^Hostname=Zabbix server/Hostname=sa-webX/' /etc/zabbix/zabbix_agent2.conf
grep -E '^(Server|ServerActive|Hostname)=' /etc/zabbix/zabbix_agent2.conf
```
- `cp .bak`: backup to roll back.
- `sed -i 's/old/new/'`: find/replace in place; `^` = line starts with, so commented `# Server=` lines are untouched.
- Expected grep output: `Server=192.168.1.54`, `ServerActive=192.168.1.54`, `Hostname=sa-webX`.

| Setting | Meaning |
|---|---|
| `Server` | Who may poll this agent on 10050 (passive). Anyone else is rejected |
| `ServerActive` | Where the agent connects to on 10051 for active checks. Log items are active-only |
| `Hostname` | Must equal the host name in the UI exactly (case-sensitive) |

## 4. Log permission + start

```bash
sudo usermod -aG adm zabbix
sudo systemctl enable --now zabbix-agent2
sudo systemctl restart zabbix-agent2
sudo -u zabbix test -r /var/log/auth.log && echo "auth.log OK"
sudo -u zabbix test -r /var/log/apache2/error.log && echo "apache error.log OK"
```
- `adm` = Ubuntu's log-reader group. Logs are `-rw-r----- syslog:adm`, so `adm` gives read-only, not sudo.
- `-a` = append (without it `-G` replaces all groups).
- `enable --now` = start at boot + now. `restart` = a process only gets new groups after restart.
- `sudo -u zabbix test -r FILE` = "can the zabbix user read this?" Both lines must print OK.
- Trade-off: `adm` reads **all** logs. Tighter = per-file ACL (`setfacl`), but logrotate makes new files daily.

## 5. Verify on the VM and from the proxy

On sa-webX:
```bash
systemctl status zabbix-agent2 --no-pager
sudo tail -n 20 /var/log/zabbix/zabbix_agent2.log
```
`host [sa-webX] not found` is normal until step 6. The startup banner (list of plugins) is not an error.

On **sa-proxy**:
```bash
nc -zv 192.168.1.5X 10050
zabbix_get -s 192.168.1.5X -k agent.ping        # expect: 1
```
- `nc -z` only checks the port is open; `-v` prints the result.
- Before the agent was installed this said `Connection refused` (reached the VM, nothing listening).

## 6. Zabbix UI

For a new host:
1. Data collection → Hosts → Create host
2. Host name `sa-webX`, Templates `Linux by Zabbix agent active` + `ICMP Ping`, Host group `salab`
3. Interfaces → Add → Agent: the VM's IP from the table above, port `10050`
4. Monitored by → **Proxy → sa-proxy** → Add
5. On sa-proxy: `sudo zabbix_proxy -R config_cache_reload` (pushes the new host to the proxy now)

Done when **ZBX is green** and Monitoring → Latest data shows values. Then **snapshot the VM** (`agent-ok`) and update [00-progress.md](00-progress.md).

## Firewall

For a NEW VM: insert Security Group `sa-web` on it and add its IP to sa-proxy's 10051 rule (done for 211, 212, 213, 215). See [04-firewall.md](04-firewall.md).

## Troubleshooting

| Symptom | Cause |
|---|---|
| `Unable to locate package zabbix-agent2` | Repo not added (step 1) |
| `host [sa-webX] not found` | Host not in UI, or name differs from `Hostname=` |
| `cannot connect to [192.168.1.54:10051]: i/o timeout` | sa-proxy firewall missing the 10051 rule |
| `connection from "x.x.x.x" rejected` | Someone other than the proxy polled the agent (expected) |
| ZBX red, `Connection refused` | Agent not running |
| ZBX red, timeout | Firewall dropping 10050 |
