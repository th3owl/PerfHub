# Memory Timeline Collector And Case Summary

## Memory Case Summary: NODE-4

### Current Snapshot

The node is busy but not under active memory pressure.

Evidence:

```text
MemAvailable: ~575 GiB
Swap used: ~244 MiB
vmstat si/so: 0/0
Dirty: ~536 kB
Writeback: 0
CPU iowait: 0
```

### Current Memory Attribution

Approximate memory classes:

```text
HugePages used        ~581 GiB
Oracle/session memory ~279 GiB current cgroup usage
Oracle visible RSS    ~287 GiB by user-level RSS rollup
Logstash memory       ~9.5 GiB cgroup usage
File cache            ~131-139 GiB
Kernel slab           ~12.8 GiB
```

### Historical OOM Evidence

Kernel logs show historical OOM events.

Important events:

```text
2026-03-02 15:11:24 global OOM killed java/logstash
2026-03-02 15:11:35 global OOM killed ohasd.bin
2026-03-02 15:11:36 global OOM killed orarootagent.bin
2026-04-30 23:58:47 global OOM killed java/logstash
2026-05-21 02:39:39 memory cgroup OOM killed exadata-dbproc
```

### March 2 OOM Interpretation

The March 2 event was a true node-level OOM:

```text
oom-kill:constraint=CONSTRAINT_NONE ... global_oom
```

Kernel `Mem-Info` showed memory dominated by anonymous memory:

```text
active_anon   ~751 GiB
inactive_anon ~12 GiB
file cache    nearly depleted
free memory   ~8.3 GiB
```

This is very different from the current healthy snapshot.

### Final Memory Conclusion

- The node is currently healthy from a memory perspective, but it has a history of serious OOM events.
- The current memory footprint is mostly explained by database HugePages/shared memory, Oracle process memory, and file cache.
- The historical March 2 OOM appears to have been driven by anonymous memory exhaustion, not kernel slab growth, dirty writeback, or active swap pressure in the current snapshot.
- The killed process should not automatically be treated as the root cause. Logstash Java was killable because it had normal OOM protection, while many Oracle processes had `oom_score_adj=-1000`.

## Why Timeline Collection Is Needed

A current snapshot can explain the node now. Kernel logs can prove selected historical events such as OOM kills. But exact memory spike timing requires timestamped samples collected before or during the spike.

Without timestamped samples, we can infer from logs and current counters, but we cannot reconstruct exact per-minute process RSS retroactively.

## What The Timeline Should Capture

```text
Timestamp
Node memory classes
Top process RSS
Memory by OS user
Swap activity
Dirty/writeback state
HugePages state
Important cgroup memory counters
Recent OOM events
```

## Manual Snapshot

Use this when taking a one-time evidence sample.

```bash
date '+%F %T'
free -wh
grep -E 'MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|SwapCached|Active|Inactive|AnonPages|Mapped|Shmem|Slab|SReclaimable|SUnreclaim|Dirty|Writeback|PageTables|KernelStack|HugePages_Total|HugePages_Free|Hugepagesize|Hugetlb' /proc/meminfo
ps -eo pid,ppid,user,comm,%mem,rss,vsz,etime,args --sort=-rss | head -15
vmstat 1 2
```

## Loop Collector

This collects one sample every 60 seconds.

```bash
while true; do
  echo "===== $(date '+%F %T') ====="
  echo "### free"
  free -wh
  echo "### meminfo"
  grep -E 'MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|SwapCached|Active|Inactive|AnonPages|Mapped|Shmem|Slab|SReclaimable|SUnreclaim|Dirty|Writeback|PageTables|KernelStack|HugePages_Total|HugePages_Free|Hugepagesize|Hugetlb' /proc/meminfo
  echo "### top rss processes"
  ps -eo pid,ppid,user,comm,%mem,rss,vsz,etime,args --sort=-rss | head -15
  echo "### memory by user"
  ps -eo user=,rss= | awk '{mem[$1]+=$2} END {for (u in mem) printf "%-20s %.2f GiB\n", u, mem[u]/1024/1024}' | sort -k2 -nr
  echo "### vmstat"
  vmstat 1 2 | tail -1
  echo
  sleep 60
done >> /tmp/memory_timeline.log
```

## Spike Classification

| Evidence | Likely Meaning |
|---|---|
| `MemAvailable` falling | Node has less allocatable memory |
| `AnonPages` rising | Application/private memory growth |
| `Cached` rising | File cache growth |
| `Cached` falling sharply | Kernel is reclaiming file cache |
| `SUnreclaim` rising | Kernel unreclaimable slab growth |
| `HugePages_Free` falling | HugePages consumption increased |
| `si`/`so` non-zero in `vmstat` | Active swap activity |
| `Dirty` rising | Data waiting to be written to disk |
| `Writeback` rising | Dirty pages being flushed to disk |
| OOM messages in `dmesg` | Kernel killed process due to memory exhaustion |
| cgroup `failcnt` rising | cgroup memory limit was hit |
| cgroup `oom_kill` rising | cgroup OOM killed a process |

## Key Takeaway

For future incidents, start the collector early. It turns memory analysis from a snapshot exercise into a timeline investigation.

## Historical `sar` Data For Memory

If `sar` is already installed and collecting data, it is one of the most useful sources for historical memory analysis.

### Commands

```bash
sar -r
sar -S
sar -W
```
Meaning
```
sar -r = physical memory usage history
sar -S = swap usage history
sar -W = swap page-in/page-out history
```
What To Check
`sar -r`
Use this to see:
```
when memory usage increased
when available/free memory dropped
whether the spike was sustained or short-lived
```
`sar -S`
Use this to see:
```
when swap usage increased
whether swap stayed flat or kept growing
```
`sar -W`
Use this to see:
```
whether active swap-in or swap-out was happening
whether the node was under memory reclaim pressure
```
#### Interpretation
```
high memory use with stable swap
  memory may be busy but not pressured

rising swap use with non-zero page-in/page-out
  memory pressure is more likely

memory spike followed by recovery
  transient workload or reclaim event

sustained memory growth
  possible leak, workload growth, or shared memory expansion
```
Example Workflow
```
sar -r
sar -S
sar -W
```
#### Then correlate with:
- dmesg OOM timestamps
- application logs
- OSWatcher/ExaWatcher
- current /proc/meminfo state
#### Key Takeaway
Use sar when the memory issue happened in the past and live commands now look normal.

## Example: Historical `sar` Data For Memory

Commands:

```bash
sar -r 1 3
sar -S 1 3
sar -W 1 3
```
Sample data:
```
sar -r 1 3
Linux 5.15.0-308.179.6.16.el8uek.x86_64 (xxx 	07/22/26 	_x86_64_	(380 CPU)

09:20:34    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
09:20:35    270684416 522681884 1174651532     81.27   2049792 329357392 309139512     21.13 104694628 413761652     76392
09:20:36    270539160 522555700 1174796788     81.28   2049792 329376208 309181292     21.14 104694568 413807984     91480
09:20:37    270744400 522780248 1174591548     81.27   2049792 329395312 309041488     21.13 104694516 413712736    107520
Average:    270655992 522672611 1174679956     81.27   2049792 329376304 309120764     21.13 104694571 413760791     91797

sar -S 1 3
Linux 5.15.0-308.179.6.16.el8uek.x86_64 (xxx) 	07/22/26 	_x86_64_	(380 CPU)

09:20:50    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
09:20:51     17365748    128260      0.73      5460      4.26
09:20:52     17365748    128260      0.73      5460      4.26
09:20:53     17365748    128260      0.73      5460      4.26
Average:     17365748    128260      0.73      5460      4.26

sar -W 1 3
Linux 5.15.0-308.179.6.16.el8uek.x86_64 (xxx) 	07/22/26 	_x86_64_	(380 CPU)

09:21:06     pswpin/s pswpout/s
09:21:07         0.00      0.00
09:21:08         0.00      0.00
09:21:09         0.00      0.00
Average:         0.00      0.00
```
Sample interpretation:
```
Average kbavail  : ~522672611 KB (~498.5 GiB)
Average %memused : 81.27%
Average %swpused : 0.73%
Average pswpin/s : 0.00
Average pswpout/s: 0.00
```
Conclusion:
- Memory is heavily utilized but not under active pressure.
- Available memory remains high.
- Swap usage is tiny.
- There is no active swap-in or swap-out.
