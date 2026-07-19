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
