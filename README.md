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

