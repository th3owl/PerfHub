# Linux Resource Investigation Runbook

This repository records a command-by-command Linux OS-level investigation flow for memory, CPU, process lifecycle, and resource spike analysis.

The goal is to build an expert-level operational runbook using built-in Linux tools first, without installing external utilities.

## Scope

- Process lifecycle and resource usage
- Node-level memory consumption
- Node-level CPU consumption
- Per-process CPU and memory usage
- Timeline creation for spikes and dips
- Evidence-based interpretation of why resource usage changed

## Current Focus

Memory investigation using built-in Linux commands.

## Memory Investigation

### Step 1: Node Memory Summary

Command:

```bash
free -wh
```
Example output:
```
              total        used        free      shared     buffers       cache   available
Mem:          1.3Ti       746Gi       480Gi       9.0Gi       551Mi       139Gi       585Gi
Swap:          15Gi       243Mi        15Gi
```
#### Observation:
```
Total memory: 1.3 TiB
Used memory: 746 GiB
Free memory: 480 GiB
Shared memory: 9.0 GiB
Buffers: 551 MiB
Cache: 139 GiB
Available memory: 585 GiB
Swap used: 243 MiB out of 15 GiB
```
#### Interpretation:
- The node is busy but not under active memory pressure. The most important field is available, not used. Linux uses free memory for cache, so high used memory alone is not a problem.
#### Current memory status:
- Healthy / no active memory pressure visible from free output

### Step 2: Detailed Memory Breakdown
Command:
```
cat /proc/meminfo
```
#### Important fields from the sample:
```
MemTotal:       1434068596 kB
MemFree:        493474824 kB
MemAvailable:   603295628 kB
Cached:         137168412 kB
AnonPages:      165797836 kB
Shmem:           9431988 kB
Slab:           13420340 kB
SReclaimable:    9417104 kB
SUnreclaim:      4003236 kB
PageTables:      1441208 kB
SwapTotal:      16777212 kB
SwapFree:       16527480 kB
HugePages_Total:   297989
HugePages_Free:      355
Hugepagesize:       2048 kB
Hugetlb:        610281472 kB
```
#### Key interpretation:

- MemAvailable is around 575 GiB, so the node still has breathing room.
- AnonPages is around 158 GiB, representing application/private anonymous memory.
- Cached is around 131 GiB, mostly file cache.
- Slab is around 12.8 GiB, kernel memory.
- Hugetlb is around 582 GiB, showing a very large HugePages pool.
- HugePages_Free is very small, meaning most huge pages are in use.
- Swap usage is low, around 243 MiB.
HugePages calculation:
```
HugePages_Total * Hugepagesize
= 297989 * 2 MiB
= approximately 582 GiB
```
- This suggests the largest memory consumer class is HugePages, commonly used by databases such as Oracle for SGA/shared memory.
  
### Step 3: Top Process RSS
Command:
```
ps -eo pid,ppid,user,comm,%mem,rss,vsz,etime,args --sort=-rss | head -30
```
#### Important findings from the sample:
```
PID 388791 root   java            RSS ~9.3 GiB   VSZ ~68 GiB
PID 359597 oracle ora_ipc0...     RSS ~6.1 GiB   VSZ ~588 GiB
Several oracle processes          RSS ~1-3 GiB   VSZ ~589 GiB
```
#### Interpretation:
- The largest normal process RSS is a Java Logstash process around 9.3 GiB. Its command line shows:
```
-Xms8192m -Xmx8192m
```
- So an RSS around 9 GiB is expected.
- Oracle processes show very large VSZ, around 614-619 GiB, but much smaller RSS. This usually means the processes are attached to a large shared memory area such as Oracle SGA.
#### Important warning:
- Do not blindly add Oracle process RSS values together. Shared memory can be visible in many processes, so summing RSS can overcount real physical memory usage.
### Memory Terms
- `MemTotal`
Total physical RAM visible to Linux.
- `MemFree`
RAM doing nothing right now. This is usually not the best health indicator.
- `MemAvailable`
Estimated memory available for new workloads without serious pressure.
- `Used`
Memory used by processes, kernel, cache, buffers, huge pages, and other consumers.
- `Buffers`
Block device metadata cache.
- `Cached`
File contents cached in RAM. Usually reclaimable.
- `RSS`
Resident Set Size. Physical RAM currently mapped into a process.
- `VSZ`
Virtual memory size. Address space mapped or reserved by a process. This is not the same as real RAM usage.
- `AnonPages`
Anonymous memory, usually process heap, stack, and private allocations.
- `Shmem`
Shared memory and tmpfs-backed memory.
- `HugePages`
Large memory pages, commonly used by databases.
- `Hugetlb`
Memory reserved or used for HugeTLB huge pages.
- `Slab`
Kernel memory cache.
- `SReclaimable`
Reclaimable slab memory.
- `SUnreclaim`
Kernel slab memory not easily reclaimable.
- `Swap`
Disk-backed memory overflow area.
- `Dirty`
Modified memory pages waiting to be written to disk.
- `Writeback`
Dirty pages currently being written to disk.
#### Current Conclusion From Sample Node
- The sample node is busy, but not under active memory pressure.
#### The main memory story is:
- Large HugePages footprint, likely database shared memory
- Healthy MemAvailable
- Low swap usage
- No dirty/writeback pressure
- Top process RSS does not explain total used memory by itself
  
### Step 4: Memory Usage By OS User
Command:

```
ps -eo user=,rss= | awk '{mem[$1]+=$2} END {for (u in mem) printf "%-20s %.2f GiB\n", u, mem[u]/1024/1024}' | sort -k2 -nr
```
#### Sample output:
```
oracle               287.47 GiB
root                 11.04 GiB
grid                 9.30 GiB
exawatch             0.05 GiB
opc                  0.02 GiB
polkitd              0.01 GiB
dbmsvc               0.01 GiB
```
#### Observation:
- Most visible process RSS belongs to the oracle user. The root user is mainly explained by the Logstash Java process seen earlier. The grid user has a smaller but still visible memory footprint.
#### Interpretation:
- This command gives a rough RSS total by OS user. It is useful for identifying ownership, but it is not final memory accounting.
- On database nodes, RSS-by-user can overcount shared memory mappings. Oracle processes may map the same large SGA/shared memory area, so blindly summing RSS can produce misleading totals.
#### Expert note:
Use this command to identify direction:
- Who should I investigate first?
Do not use it alone to conclude:
- Who physically owns every GiB of RAM?

### Step 5: Deep Dive Into One Process

Command:

```
cat /proc/388791/status
```
#### Important output:
```
Name:       java
State:      S (sleeping)
VmSize:     71452208 kB
VmRSS:       9758508 kB
RssAnon:     9726252 kB
RssFile:       32256 kB
RssShmem:          0 kB
VmSwap:          432 kB
Threads:         879
HugetlbPages:      0 kB
````
#### Interpretation:
- This Java process uses about 9.3 GiB of real RAM. Almost all of it is anonymous memory, which usually means heap/private process memory.
- The process has very little file-backed RSS, no shared RSS, and almost no swap usage.
- Because the Java command line showed -Xms8192m -Xmx8192m, this memory footprint is expected: 8 GiB Java heap plus JVM/native overhead.
#### Expert note:
- For a single process, /proc/<pid>/status is better than plain ps because it separates:
```
VmRSS    = total resident memory
RssAnon  = private/anonymous memory
RssFile  = file-backed memory
RssShmem = shared memory
VmSwap   = swapped memory
```
### Step 6: Deep Dive Into A Database Process

Command:

```
cat /proc/359597/status
```
#### Important output:
```
Name:         ora_ipc0_em71po
State:        S (sleeping)
VmSize:       616994908 kB
VmRSS:          6399768 kB
RssAnon:          30716 kB
RssFile:          77596 kB
RssShmem:       6291456 kB
VmSwap:               0 kB
HugetlbPages:  591718400 kB
Threads:              1
```
#### Interpretation:
- This Oracle process has a very large virtual memory size because it maps a large shared memory / SGA region.
- Its resident memory is around 6.1 GiB, but almost all of that is shared memory:
```
RssShmem: ~6.0 GiB
RssAnon : ~30 MiB
```
- The process also maps around 564 GiB of HugePages:
```
HugetlbPages: ~564 GiB
```
- This means the process privately owns very little normal memory. Its large memory footprint is mainly due to database shared memory mappings.
#### Expert note:
For database processes, plain RSS and VSZ can be misleading.
Use /proc/<pid>/status to separate:
```
RssAnon      = private process memory
RssShmem     = shared memory
HugetlbPages = huge page backed database memory
VmSwap       = swapped memory
````
#### Conclusion:
Do not add RSS across many Oracle processes to calculate total Oracle memory. Many processes may map the same shared memory region.

### Step 7: Process Memory Map Summary

Command:

```
cat /proc/359597/smaps_rollup
```
#### Important output:
```
Rss:             6400008 kB
Pss:             4577238 kB
Pss_Anon:          30956 kB
Pss_File:            241 kB
Pss_Shmem:       4546041 kB
Shared_Hugetlb: 565311488 kB
Private_Hugetlb: 44216320 kB
Swap:                  0 kB
```
#### Interpretation:
- smaps_rollup gives a summarized memory-map view for one process.
#### Important terms:
- `RSS`
Resident memory mapped by the process.
- `PSS`
Proportional memory attributed to the process. If memory is shared, each process gets only its fair share.
- `Pss_Anon`
Proportional anonymous/private memory.
- `Pss_File`
Proportional file-backed memory.
- `Pss_Shmem`
Proportional shared memory.
- `Shared_Hugetlb`
HugePages-backed memory shared with other processes.
- `Private_Hugetlb`
HugePages-backed memory private to this process mapping.
- `Swap`
Memory from this process currently swapped out.
#### Conclusion:
This Oracle process is not a private memory-heavy process. It is attached to a very large HugePages-backed database memory area. Swap is zero for this process.

### Step 8: Live Memory Pressure Check

Command:

```
vmstat 1 5
```
#### Sample output:
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
102  0 249732 485676640 566696 146695712    0    0    31     2    0    0 33  5 61  0  0
107  1 249732 484387808 566696 146696560    0    0 19544  9651 617776 421713 37  6 56  0  0
89   0 249732 482358432 566696 146696688    0    0 22280    38 558897 377153 35  5 59  0  0
121  1 249732 481530464 566696 146696992    0    0 21620   948 580718 386897 36  5 58  0  0
115  1 249732 481183776 566696 146697008    0    0 20060     9 610101 427377 36  6 58  0  0
```
#### Important fields:
- `swpd`
Amount of swap currently allocated.
- `si`
Swap-in rate. Memory being read back from swap.
- `so`
Swap-out rate. Memory being pushed to swap.
- `free`
Free memory at the moment.
- `b`
Blocked processes, often waiting on I/O.
- `wa`
CPU time spent waiting on I/O.

#### Interpretation:
- The node has around 244 MiB swap allocated, but si and so are zero. This means the node is not actively swapping.
- Free memory remains very high, around 481-485 GiB. I/O wait is zero. There is no live memory pressure visible in this sample.
#### Expert note:
Swap used is historical/current allocation. Active memory pressure is better identified by non-zero si/so, falling MemAvailable, OOM messages, and reclaim/writeback symptoms.

### Concept: Run Queue

In `vmstat`, the `r` column shows the run queue.

It means the number of tasks that are runnable: either currently running on CPU or ready to run and waiting for CPU.

Example:

```
procs
 r  b
102 0
```
- r = 102 means 102 tasks are runnable.
This must be compared with CPU count.
Rule of thumb:
```
r <= CPU count      usually acceptable
r > CPU count       possible CPU queueing
r >> CPU count      CPU saturation likely
```
- A high r value on a small 8-core system may be serious. The same r value on a 252-CPU system may be normal.
Always compare r with:
```
id = CPU idle percentage
wa = I/O wait
b  = blocked tasks
```
In the sample:
```
r  = 89-121
id = 56-61%
wa = 0%
b  = 0-1
```
#### Interpretation:
The node is busy, but not CPU saturated. There is also no evidence of I/O blocking in this sample.

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

### Step 12: Process cgroup Membership

Command:

```
for p in 388791 359597; do echo "### PID $p"; cat /proc/$p/cgroup; done
```
Sample output summary:
```
PID 388791 java/logstash:
memory:/system.slice/logstash.service
cpu,cpuacct:/system.slice/logstash.service
name=systemd:/system.slice/logstash.service

PID 359597 ora_ipc0:
memory:/user.slice/user-1001.slice/session-302219.scope
name=systemd:/system.slice/oracle-ohasd.service
```
#### Interpretation:
The node is using cgroup v1 because /proc/<pid>/cgroup shows numbered controllers such as:
```
11:memory:
5:cpu,cpuacct:
1:name=systemd:
```
For Logstash, memory and systemd service accounting are aligned under:
```
/system.slice/logstash.service
```
For the Oracle process, systemd service identity and memory accounting differ:
```
memory controller: /user.slice/user-1001.slice/session-302219.scope
systemd identity : /system.slice/oracle-ohasd.service
```
#### Expert note:
When checking cgroup memory usage or limits, use the path shown for the memory controller, not necessarily the name=systemd path.

### Step 13: cgroup Memory Usage For A Service

Command:

```
cg=/sys/fs/cgroup/memory/system.slice/logstash.service
for f in memory.usage_in_bytes memory.max_usage_in_bytes memory.limit_in_bytes memory.failcnt memory.oom_control; do echo "### $f"; cat "$cg/$f"; done
```
Sample output:
```
memory.usage_in_bytes
10168766464

memory.max_usage_in_bytes
10183327744

memory.limit_in_bytes
9223372036854771712

memory.failcnt
0

memory.oom_control
oom_kill_disable 0
under_oom 0
oom_kill 0
```
#### Interpretation:
The Logstash cgroup is currently using about 9.47 GiB of memory, with a peak around 9.49 GiB.
The memory limit value is extremely large:
```
9223372036854771712
```
This effectively means no practical cgroup memory limit.
`memory.failcnt = 0` means the cgroup has not hit its memory limit.
`under_oom = 0` means the cgroup is not currently under OOM.
`oom_kill = 0` means no cgroup OOM kill is recorded for this cgroup.
#### Conclusion:
Logstash is not currently memory-limited by cgroups. Historical Logstash kills were global OOM victim selections, not cgroup-limit kills.

### Step 14: cgroup Memory Usage For Oracle Session Scope

Command:

```
cg=/sys/fs/cgroup/memory/user.slice/user-1001.slice/session-302219.scope
for f in memory.usage_in_bytes memory.max_usage_in_bytes memory.limit_in_bytes memory.failcnt memory.oom_control; do echo "### $f"; cat "$cg/$f"; done
```
Sample output:
```
memory.usage_in_bytes
300013641728

memory.max_usage_in_bytes
797888757760

memory.limit_in_bytes
9223372036854771712

memory.failcnt
0

memory.oom_control
oom_kill_disable 0
under_oom 0
oom_kill 0
```
#### Interpretation:
- The Oracle/session memory cgroup currently accounts for about 279 GiB.
- Its recorded peak is about 743 GiB.
- There is no practical cgroup memory limit, and no cgroup OOM is currently recorded.
#### Expert note:
`memory.max_usage_in_bytes` is useful because it preserves the peak usage since the cgroup was created. However, it does not include the timestamp of that peak.
#### Conclusion:
The Oracle/session cgroup had a much higher historical memory footprint than it has now. This may relate to the historical global OOM, but timing must be proven with time-series logs or other timestamped evidence.

### Step 15: Memory Pressure Stall Information

Command:

```
cat /proc/pressure/memory
```
Sample output:
```
cat: /proc/pressure/memory: Operation not supported
```
#### Interpretation:
- Pressure Stall Information, also called PSI, is not available on this node.
- If available, PSI can show whether workloads are being delayed because of memory pressure.
- Example PSI output would look like:
```
some avg10=0.00 avg60=0.00 avg300=0.00 total=...
full avg10=0.00 avg60=0.00 avg300=0.00 total=...
```
- Meaning:
`some`
At least one task was stalled due to memory pressure.
`full`
All non-idle tasks were stalled due to memory pressure.
#### Conclusion:
Since PSI is not supported here, memory pressure must be inferred from other built-in evidence: MemAvailable, swap activity, OOM logs, cgroup counters, and time-series collection.

### Step 16: Kernel Memory And Slab

Command:

```
grep -E 'Slab|SReclaimable|SUnreclaim|KernelStack|PageTables|Vmalloc|Percpu' /proc/meminfo
```
Sample output:
```
Slab:           13412640 kB
SReclaimable:    9415576 kB
SUnreclaim:      3997064 kB
KernelStack:      110800 kB
PageTables:      1425616 kB
VmallocUsed:     1077288 kB
Percpu:          2158592 kB
```
#### Interpretation:
Kernel memory is not the main memory consumer on this node.
Approximate sizes:
```
Slab         ~12.8 GiB
SReclaimable ~9.0 GiB
SUnreclaim   ~3.8 GiB
KernelStack  ~108 MiB
PageTables   ~1.36 GiB
Percpu       ~2.06 GiB
VmallocUsed  ~1.03 GiB
```
Most slab memory is reclaimable. Unreclaimable slab is relatively small compared with the node size.
#### Conclusion:
Kernel memory does not explain the large used memory. The larger consumers remain HugePages, anonymous memory, and file cache.

### Step 18: Corrected Top Slab Consumers

Command:

```
awk 'NR>2 {mb=($2*$4)/1024/1024; printf "%12.2f MB  %-32s objects=%-12s active=%-12s objsize=%s\n", mb,$1,$2,$3,$4}' /proc/slabinfo | sort -nr | head -20
```
Sample output:
```
2776.66 MB  buffer_head
2710.05 MB  proc_inode_cache
1036.56 MB  dentry
 775.23 MB  radix_tree_node
 767.22 MB  ext4_inode_cache
 649.84 MB  kmalloc-512
```
#### Interpretation:
The largest slab consumers are mostly filesystem and kernel metadata caches.
##### Important terms:
- `buffer_head`
Block/filesystem buffer metadata.
- `proc_inode_cache`
Kernel inode cache for /proc entries.
- `dentry`
Directory entry cache used for pathname lookups.
- `radix_tree_node`
Kernel metadata structure often associated with cached pages or filesystem mappings.
- `ext4_inode_cache`
Inode cache for ext4 filesystems.
#### Conclusion:
The slab footprint does not suggest kernel memory is the primary memory problem. It is modest for a 1.3 TiB node and mostly made of expected filesystem/proc metadata.

### Step 19: HugePages Usage

Command:

```
grep -E 'HugePages_Total|HugePages_Free|HugePages_Rsvd|HugePages_Surp|Hugepagesize|Hugetlb' /proc/meminfo
```
Sample output:
```
HugePages_Total:   297989
HugePages_Free:      355
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:        610281472 kB
```
Calculation:
```
Total HugePages memory
= HugePages_Total * Hugepagesize
= 297989 * 2 MiB
= ~582.0 GiB
```
Free HugePages memory
```
= HugePages_Free * Hugepagesize
= 355 * 2 MiB
= ~0.69 GiB
```
Used HugePages memory
```
= (HugePages_Total - HugePages_Free) * Hugepagesize
= 297634 * 2 MiB
= ~581.3 GiB
```
#### Interpretation:
- Almost the entire HugePages pool is in use.
- On database nodes, this is commonly expected because databases such as Oracle use HugePages for SGA/shared memory.
#### Expert note:
HugePages must be analyzed separately from normal process RSS. A database process can map hundreds of GiB of HugePages while showing much smaller private memory in /proc/<pid>/status.

### Step 20: Transparent HugePages

Command:

```
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```
Sample output:
```
always madvise [never]
always defer defer+madvise [madvise] never
```
#### Interpretation:
- The active value is shown inside brackets.
- Transparent HugePages enabled = never
- Transparent HugePages defrag  = madvise
- This means Transparent HugePages are disabled for automatic use.
#### Important distinction:
- HugeTLB / static HugePages
- Preconfigured huge page pool. Shown in /proc/meminfo using fields such as HugePages_Total, HugePages_Free, and Hugetlb.
- Transparent HugePages / THP
- Kernel-managed automatic huge pages for normal memory. Controlled under /sys/kernel/mm/transparent_hugepage.
#### Conclusion:
This node uses static HugePages heavily, while Transparent HugePages are disabled. This is a common and usually preferred database-node configuration.

### Step 21: Swap Configuration And Usage

Command:

```
swapon --show
grep -E 'SwapTotal|SwapFree|SwapCached' /proc/meminfo
```
Sample output:
```
NAME      TYPE      SIZE   USED PRIO
/dev/dm-9 partition  16G 243.9M   -2

SwapCached:        13236 kB
SwapTotal:      16777212 kB
SwapFree:       16527480 kB
```
### Interpretation:
- Swap is configured on /dev/dm-9.
- Swap usage is about 244 MiB out of 16 GiB, which is tiny for a 1.3 TiB node.
- SwapCached is also small, around 13 MiB.
- Earlier vmstat showed:
```
si = 0
so = 0
```
- So the node is not actively swapping.
#### Expert note:
Do not treat small swap usage alone as active memory pressure. Active pressure is better shown by non-zero si and so in vmstat, falling MemAvailable, reclaim stalls, or OOM events.

### Step 22: Dirty And Writeback Memory

Command:

```
grep -E 'Dirty|Writeback|NFS_Unstable|Bounce|WritebackTmp' /proc/meminfo
```
Sample output:
```
Dirty:               536 kB
Writeback:             0 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
```
#### Interpretation:
- There is no writeback pressure in the current snapshot.
- `Dirty` is memory that has been modified but not yet written to disk.
- `Writeback` is dirty memory currently being written to disk.
- If `Dirty` and `Writeback` are high during memory pressure, reclaim can become slow because pages must be written before they can be freed.
#### Conclusion:
Current memory reclaim is not blocked by dirty page writeback.

# SUMMARY

Now we’re ready to create the first **memory summary timeline** from all evidence.

Before CPU/process lifecycle, let’s add a compact conclusion section to your README:

## Memory Case Summary: NODE-4

### Current Snapshot

The node is busy but not under active memory pressure.

Evidence:

```
MemAvailable: ~575 GiB
Swap used: ~244 MiB
vmstat si/so: 0/0
Dirty: ~536 kB
Writeback: 0
CPU iowait: 0
Current Memory Attribution
Approximate memory classes:
HugePages used        ~581 GiB
Oracle/session memory ~279 GiB current cgroup usage
Oracle visible RSS    ~287 GiB by user-level RSS rollup
Logstash memory       ~9.5 GiB cgroup usage
File cache            ~131-139 GiB
Kernel slab           ~12.8 GiB
```
## Historical OOM Evidence
Kernel logs show historical OOM events.
Important events:
```
2026-03-02 15:11:24 global OOM killed java/logstash
2026-03-02 15:11:35 global OOM killed ohasd.bin
2026-03-02 15:11:36 global OOM killed orarootagent.bin
2026-04-30 23:58:47 global OOM killed java/logstash
2026-05-21 02:39:39 memory cgroup OOM killed exadata-dbproc
```
## March 2 OOM Interpretation
The March 2 event was a true node-level OOM:
```
oom-kill:constraint=CONSTRAINT_NONE ... global_oom
```
Kernel `Mem-Info` showed memory dominated by anonymous memory:
```
active_anon   ~751 GiB
inactive_anon ~12 GiB
file cache    nearly depleted
free memory   ~8.3 GiB
```
This is very different from the current healthy snapshot.
## Final Memory Conclusion
- The node is currently healthy from a memory perspective, but it has a history of serious OOM events.
- The current memory footprint is mostly explained by database HugePages/shared memory, Oracle process memory, and file cache.
- The historical March 2 OOM appears to have been driven by anonymous memory exhaustion, not kernel slab growth, dirty writeback, or active swap pressure in the current snapshot.
- The killed process should not automatically be treated as the root cause. Logstash Java was killable because it had normal OOM protection, while many Oracle processes had `oom_score_adj=-1000`.
