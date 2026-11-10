# 08 — CLI labs: watch every step from the terminal

Back to [README](../README.md) · [Progress](00-progress.md) · Flow steps ①–⑥ are defined in [02-architecture.md](02-architecture.md).

Tools on sa-proxy: `sudo apt install -y zabbix-get sqlite3 tcpdump`
(`zabbix-get` asks an agent for a value like the proxy does; `sqlite3` reads the proxy DB; `tcpdump` shows packets.)

## Lab 1 — what the agent can measure (on a web VM)
```bash
ip -br a                                         # interface names (probably ens18)
sudo zabbix_agent2 -t system.cpu.util            # -t = test one item key locally
sudo zabbix_agent2 -t vm.memory.size[pavailable]
sudo zabbix_agent2 -t 'vfs.fs.size[/,pfree]'     # percent free on /
```

## Lab 2 — step ③a, passive polling (on sa-proxy)
```bash
zabbix_get -s 192.168.1.51 -k agent.ping         # 1 = alive (basis of the ZBX icon)
zabbix_get -s 192.168.1.51 -k system.cpu.util
zabbix_get -s 192.168.1.51 -k 'net.if.in[ens18]'
```
From sa-web2 → blocked by the Proxmox firewall (timeout). From the VM itself → rejected by `Server=`.

## Lab 3 — on the wire (two terminals)
```bash
sudo tcpdump -ni any port 10050      # on sa-web1: proxy .54 polling (③a)
sudo tcpdump -ni any port 10051      # on sa-proxy: agents in (③b) + proxy out to <AZURE_PUBLIC_IP> (②⑤)
```
`-n` no name lookup, `-i any` all interfaces. Proxy↔Azure payload is unreadable (PSK).

## Lab 4 — the proxy's copy of the config (step ②)
```bash
sudo sqlite3 /var/lib/zabbix/zabbix_proxy.db "SELECT hostid, host FROM hosts;"
sudo sqlite3 /var/lib/zabbix/zabbix_proxy.db "SELECT ip, port FROM interface;"
sudo sqlite3 /var/lib/zabbix/zabbix_proxy.db "SELECT i.key_, i.delay FROM items i JOIN hosts h ON i.hostid=h.hostid WHERE h.host='sa-web1' LIMIT 15;"
```
Nov 10 output: hosts `10818|sa-web1`, `10819|sa-web2`, `10001|` (likely the template — templates live in the same table); interfaces `.51:10050`, `.52:10050`.

## Lab 5 — the buffer (step ④)
```bash
sudo sqlite3 /var/lib/zabbix/zabbix_proxy.db "SELECT COUNT(*) FROM proxy_history;"
```
Normally small. Demo: block proxy → Azure briefly, count grows; unblock, it drains.

## Lab 6 — control the running proxy
```bash
sudo zabbix_proxy -R config_cache_reload   # force step ② now (after adding a host)
sudo zabbix_proxy -R diaginfo              # internal stats → proxy log
```

## Lab 7 — live logs
```bash
sudo tail -f /var/log/zabbix/zabbix_agent2.log
sudo tail -f /var/log/zabbix/zabbix_agent2.log | grep --line-buffered -E "working again|fail|cannot|not found|rejected"
sudo grep -E "working again|fail|cannot|not found|rejected" /var/log/zabbix/zabbix_agent2.log | tail -n 10
```
`--line-buffered` = print each match immediately when piped. Demo: agent log + proxy log side by side while breaking/fixing something.

## Lab 8 — step ⑥ via the API (no GUI)
Token: User settings → API tokens.
```bash
curl -s -X POST http://<AZURE_PUBLIC_IP>/api_jsonrpc.php \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -d '{"jsonrpc":"2.0","method":"host.get","params":{"output":["host"],"selectInterfaces":["ip"]},"id":1}'
```
Swap `host.get` for `item.get` / `history.get`. Never share the token (UI is still plain HTTP).

## Firewall checks
```bash
nc -zv 192.168.1.51 10050           # from sa-proxy: succeeded
nc -zv -w 3 192.168.1.51 10050      # from sa-web2: timed out (blocked) — -w 3 = give up after 3 s
nc -zv 192.168.1.54 10051           # from a web VM: active-check path open
```
