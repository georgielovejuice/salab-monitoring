# 10 — Timing: how often every part talks

Back to [README](../README.md) · [Progress](00-progress.md)

## The chain

```
+---------------------+
|  sa-webX (agent2)   |
+---------------------+
   |
   | 1. every 5s   "what should I collect?" --> proxy
   |               <-- item list + intervals
   |
   | 2. measures each item on its own interval
   |    (CPU 10s, memory 10s, disk 10s, FS 30s ...)
   |
   | 3. every 5s   send collected values    --> proxy
   |
   | 4. every 60s  heartbeat "I'm alive"    --> proxy
   |
   v
+---------------------+
|  sa-proxy           |
|  (SQLite buffer)    |
+---------------------+
   |
   | 5. every 10s  "any config changes?"    --> server
   |               <-- hosts + items
   |
   | 6. every 1s   send stored values       --> server
   |
   | 7. server unreachable? keep data up to 24h
   |
   v
+---------------------+
|  Zabbix server      |
|  (Azure)            |
+---------------------+
   save to DB --> check triggers --> send email
```

## sa-webX ⇄ sa-proxy (`/etc/zabbix/zabbix_agent2.conf`)

| What | Direction | Interval | Setting |
|---|---|---|---|
| Ask for item list | agent → proxy | 5 s | `RefreshActiveChecks` (default) |
| Send collected values | agent → proxy | 5 s | `BufferSend` (default) |
| Send early if buffer full | agent → proxy | 1000 values | `BufferSize` (default) |
| Heartbeat | agent → proxy | 60 s | `HeartbeatFrequency` (default) |

## sa-proxy ⇄ Azure (`/etc/zabbix/zabbix_proxy.conf`)

| What | Direction | Interval | Setting |
|---|---|---|---|
| Ask for config changes | proxy → server | 10 s | `ProxyConfigFrequency` (default) |
| Send stored values | proxy → server | 1 s | `DataSenderFrequency` (default, already the minimum) |
| Keep data if server unreachable | proxy | up to 24 h | `ProxyOfflineBuffer=24` (ours) |

Check defaults yourself: `grep -A8 "### Option: DataSenderFrequency" /etc/zabbix/zabbix_proxy.conf`. A line starting with `#` = not set = default.

## How often each metric is measured (template `Linux by Zabbix agent active`, in the UI)

Rule: if an item's Interval is empty it is a **Dependent item** — change its master (shown as `Master: Item` in the name).

| Metric | Item that controls it | Was | Now |
|---|---|---|---|
| CPU utilization | `CPU idle time` (master; utilization = 100 − idle) | 1m | **10s** |
| CPU user time | `CPU user time` | 1m | **10s** |
| Memory utilization | `Available memory in %` (master) | 1m | **10s** |
| Available memory | `Available memory` | 1m | **10s** |
| Load average | `Load average (1m avg)` | 1m | **30s** |
| Network in/out | `Get network interfaces` (Items tab; master of all NIC items) | 1m | **10s** |
| Disk read/write rate + throughput | `Get block devices` (Items tab) | 1m | **10s** |
| Disk queue / time | prototype `{#DEVNAME}: Get stats` | 1m | **10s** |
| Disk free space | prototype `FS [{#FSNAME}]: Get data` | 1m | **30s** |
| Discovery rules | 3 rules | 1h | 1h (keep) |
| Everything else | — | 1m | 1m |

Prototype changes reach the hosts only when that discovery rule runs again. Network/disk discovery depend on 10 s masters, so they update fast. Filesystem discovery is 1h: set it to 1m, wait 2 min, set back to 1h. Verify on a host → Items → Interval column.

Match the interval to how fast the metric can change: CPU/memory jump instantly (10s), load/disk space move slowly (30s+), total memory never changes (1m).

## Display

| What | Interval | Where |
|---|---|---|
| Dashboard widgets | 10 s | each widget's ⋯ → Refresh interval (template dashboards: edit on the template) |
| Other pages (Latest data, Problems) | 5 s | User settings → Profile → Refresh |

## Worst case, event → screen

```
measure 10s + agent buffer 5s + proxy send 1s + dashboard 10s = ~26 s   (average ~13 s)
```
`BufferSend=1` would save at most 4 s — not worth changing. The item interval and dashboard refresh dominate.
