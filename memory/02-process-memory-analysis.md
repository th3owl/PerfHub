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
