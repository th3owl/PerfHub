### Step 9: OOM History Check

Command:

```
dmesg -T | grep -Ei 'out of memory|oom|killed process'
```
Sample findings:
```
[Mon Mar  2 15:11:24 2026] Out of memory: Killed process 95755 (java)
[Mon Mar  2 15:11:35 2026] Out of memory: Killed process 95802 (ohasd.bin)
[Mon Mar  2 15:11:36 2026] Out of memory: Killed process 96315 (orarootagent.bi)
[Mon Mar  2 15:11:37 2026] Out of memory: Killed process 56756 (oracle_56756_em)
[Thu Apr 30 23:58:47 2026] Out of memory: Killed process 288139 (java)
[Thu May 21 02:39:39 2026] Memory cgroup out of memory: Killed process 55552 (exadata-dbproc-)
```
#### Interpretation:
- The current memory snapshot is healthy, but kernel logs confirm historical OOM events.
There are two different OOM types:
`global_oom` : The full node was under memory pressure.
`CONSTRAINT_MEMCG` : Memory cgroup out of memory. A specific service or cgroup hit its memory limit. This does not always mean the whole node was out of RAM.
#### Expert note:
- Always separate current memory health from historical memory events.
- A node can look healthy now but still have had serious OOM kills earlier.

#### Current expert conclusion:

- The node is not memory-stressed now, but it has a history of real OOM events.
- March 2 was a global OOM sequence.
- April 30 was another global OOM killing Logstash Java.
- May 21 was a memory cgroup OOM, scoped to exadata-dbproc-bind.service.

### Step 10: OOM Context Analysis

Command:

```
dmesg -T | grep -Ei -A40 -B20 'Out of memory: Killed process 95755|oom-kill:constraint=CONSTRAINT_NONE'
```
#### Important evidence:
```
oom-kill:constraint=CONSTRAINT_NONE ... global_oom
```
#### Interpretation:
`global_oom` means the full node was under memory pressure. This is different from a memory cgroup OOM, where only one service or container hits its configured memory limit.
Example OOM victims:
```
2026-03-02 15:11:24 java from logstash.service
2026-03-02 15:11:35 ohasd.bin from oracle-ohasd.service
2026-03-02 15:11:36 orarootagent.bin from oracle-ohasd.service
```
Important Mem-Info fields:
```
active_anon:197065649
inactive_anon:3219910
active_file:3486
inactive_file:5665
free:2169681
```
Approximate conversion using 4 KiB pages:
```
active_anon   ~751 GiB
inactive_anon ~12 GiB
free          ~8.3 GiB
file cache    nearly depleted
```
#### Conclusion:
At the time of OOM, memory was dominated by anonymous memory. File cache was almost gone and free memory was very low. This is a true global memory exhaustion event.
#### Expert note:
The OOM victim is not always the root cause. The kernel chooses victims using OOM score, memory usage, cgroup context, and oom_score_adj.
Processes with:
```
oom_score_adj=-1000
```
are strongly protected from OOM killing.

### Concept: cgroup

`cgroup` means control group.

A cgroup is a Linux kernel mechanism for grouping processes together and applying resource accounting or limits to that group.

Cgroups can control or account for:

```
CPU
memory
I/O
process count
devices
```
Under systemd, services usually run inside cgroups.
Example:
```
/system.slice/logstash.service
/system.slice/oracle-ohasd.service
```
If a process belongs to /system.slice/logstash.service, Linux can track resource usage for the whole Logstash service, not just one PID.
Important OOM distinction:
`global_oom`
The full node is out of allocatable memory.
`Memory cgroup out of memory`
A specific cgroup hit its memory limit. The whole node may still have available RAM.
Useful commands:
```
cat /proc/<pid>/cgroup
systemctl status <service>
systemctl show <service> | grep -Ei 'Memory|CPU|Tasks'
```
For cgroup v1:
```
cat /sys/fs/cgroup/memory/system.slice/<service>/memory.limit_in_bytes
cat /sys/fs/cgroup/memory/system.slice/<service>/memory.usage_in_bytes
```
For cgroup v2:
```
cat /sys/fs/cgroup/system.slice/<service>/memory.max
cat /sys/fs/cgroup/system.slice/<service>/memory.current
cat /sys/fs/cgroup/system.slice/<service>/memory.events
```
#### Expert note:
Cgroups let us separate node-level resource exhaustion from service-level resource limits.

### Step 11: OOM Score And OOM Protection

Command:

```
for p in 388791 359597; do echo "PID=$p $(tr '\0' ' ' < /proc/$p/cmdline | cut -c1-120)"; cat /proc/$p/oom_score /proc/$p/oom_score_adj; done
```
Sample output:
```
PID=388791 /usr/lib/jvm/jdk-11-oracle-x64/bin/java -Xms8192m -Xmx8192m ...
670
0

PID=359597 ora_ipc0_em71pod6
0
-1000
```
#### Interpretation:
`oom_score` is the kernel's current score for selecting a process as an OOM victim. Higher means more likely to be killed.
`oom_score_adj` modifies the process's OOM risk.
Common values:
```
-1000 = strongly protected from OOM killing
0     = normal
1000  = very likely to be killed
```
In this sample:
- java/logstash has oom_score 670 and oom_score_adj 0
- Oracle ipc0 has oom_score 0 and oom_score_adj -1000

#### Conclusion:
Java/logstash is much more likely to be killed during OOM than the protected Oracle background process.
