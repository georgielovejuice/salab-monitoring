# 05 — Zabbix proxy (sa-proxy, VM 214)

Back to [README](../README.md) · [Progress](00-progress.md)

User member account, IP 192.168.1.54, Ubuntu 24.04, package `zabbix-proxy-sqlite3` 8.0.0.

## Build steps

| Step | Command / setting | Why |
|---|---|---|
| Repo | `wget https://repo.zabbix.com/zabbix/8.0/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_8.0+ubuntu24.04_all.deb`, `sudo dpkg -i ...`, `sudo apt update` | Proxy major version must match the server (8.0); Ubuntu only ships 6.0 |
| Install | `sudo apt install -y zabbix-proxy-sqlite3` | SQLite: no separate DB server, enough for a few hosts |
| PSK | `openssl rand -hex 32 \| sudo tee /etc/zabbix/zabbix_proxy.psk`, `chown zabbix:zabbix`, `chmod 600` | 256-bit random shared secret, readable only by the `zabbix` user |
| DB folder | `sudo mkdir -p /var/lib/zabbix`, `chown zabbix:zabbix` | The proxy creates `zabbix_proxy.db` on first start |
| Config | see below | Active proxy, encrypted link, 24 h buffer |
| Start | `sudo systemctl enable --now zabbix-proxy` | Start now and at every boot |
| Hardening (todo) | `chmod 750 /var/lib/zabbix`, `chmod 640 zabbix_proxy.db` | DB not world-readable |

`/etc/zabbix/zabbix_proxy.conf`:
```
ProxyMode=0                                  # 0 = active: proxy connects to the server
Server=<AZURE_PUBLIC_IP>                            # the Zabbix server on Azure
Hostname=sa-proxy                            # must match the proxy name in the UI
DBName=/var/lib/zabbix/zabbix_proxy.db
ProxyOfflineBuffer=24                        # keep up to 24 h of data if Azure is unreachable
TLSConnect=psk
TLSPSKIdentity=sa-proxy
TLSPSKFile=/etc/zabbix/zabbix_proxy.psk
StartPingers=2                               # 2 ICMP pinger processes (1 was at ~61% busy)
FpingLocation=/usr/bin/fping                 # fping for the ICMP Ping template
```

UI: Administration → Proxies → `sa-proxy`, mode **Active**, Encryption **PSK only** (same identity + key), Proxy address **empty** (it was `<HOME_PUBLIC_IP>` until the home IP changed, see [04-firewall.md](04-firewall.md)).

## Result

- Log: `received configuration data from server at "<AZURE_PUBLIC_IP>"`.
- UI: **Online, 8.0.0, PSK**.
- First-start `cannot open database file` is normal (the DB is created right after).

## PSK vs SSH keys

PSK is **symmetric**: the same secret on both ends. SSH keys are **asymmetric**: only the public key is copied; the private key never leaves. `openssl rand -hex 32` = 32 random bytes printed as 64 hex characters.

## Health checks

```bash
systemctl status zabbix-proxy                 # running/failed, last log lines
sudo tail -f /var/log/zabbix/zabbix_proxy.log # live log (Ctrl+C to stop)
ps aux | grep [z]abbix_proxy                  # worker processes; [z] stops grep matching itself
sudo ss -tnp | grep 10051                     # connections: out to <AZURE_PUBLIC_IP>, in from agents
```
UI: Administration → Proxies → "Last seen (age)" should stay under ~10 s.

More: [08-cli-labs.md](08-cli-labs.md).
