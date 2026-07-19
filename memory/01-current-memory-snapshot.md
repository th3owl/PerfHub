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
