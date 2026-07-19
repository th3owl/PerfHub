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
- ===MemTotal===
Total physical RAM visible to Linux.
- MemFree
RAM doing nothing right now. This is usually not the best health indicator.
- MemAvailable
Estimated memory available for new workloads without serious pressure.
- Used
Memory used by processes, kernel, cache, buffers, huge pages, and other consumers.
- Buffers
Block device metadata cache.
- Cached
File contents cached in RAM. Usually reclaimable.
- RSS
Resident Set Size. Physical RAM currently mapped into a process.
- VSZ
Virtual memory size. Address space mapped or reserved by a process. This is not the same as real RAM usage.
- AnonPages
Anonymous memory, usually process heap, stack, and private allocations.
- Shmem
Shared memory and tmpfs-backed memory.
- HugePages
Large memory pages, commonly used by databases.
- Hugetlb
Memory reserved or used for HugeTLB huge pages.
- Slab
Kernel memory cache.
- SReclaimable
Reclaimable slab memory.
- SUnreclaim
Kernel slab memory not easily reclaimable.
- Swap
Disk-backed memory overflow area.
- Dirty
Modified memory pages waiting to be written to disk.
- Writeback
Dirty pages currently being written to disk.
#### Current Conclusion From Sample Node
- The sample node is busy, but not under active memory pressure.
#### The main memory story is:
- Large HugePages footprint, likely database shared memory
- Healthy MemAvailable
- Low swap usage
- No dirty/writeback pressure
- Top process RSS does not explain total used memory by itself
  
### Next Step
Continue with memory investigation:
```
ps -eo user=,rss= | awk '{mem[$1]+=$2} END {for (u in mem) printf "%-20s %.2f GiB\n", u, mem[u]/1024/1024}' | sort -k2 -nr
```
- This gives a rough RSS total by OS user.
- Caution: on database nodes, this can overcount shared memory mappings, so it must be interpreted carefully.
