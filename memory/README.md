# Memory Investigation

This chapter covers Linux OS-level memory analysis using built-in tools only.

The goal is to answer:

- Is the node under memory pressure now?
- Who is using memory?
- Is memory used by process RSS, HugePages, page cache, slab, shared memory, or swap?
- Did the node have historical OOM events?
- Can we build a timeline of memory spikes and dips?

## Investigation Flow

1. Node memory summary with `free`
2. Detailed memory classes with `/proc/meminfo`
3. Top process RSS with `ps`
4. Memory by OS user
5. Per-process memory deep dive with `/proc/<pid>/status`
6. Process memory map summary with `/proc/<pid>/smaps_rollup`
7. Live pressure check with `vmstat`
8. OOM history from `dmesg`
9. cgroup membership and memory counters
10. Kernel slab analysis
11. HugePages and Transparent HugePages
12. Swap and writeback checks
13. Timeline collection

## Case Study

Current working case:

```
Node: NODE-4
Memory: 1.3 TiB
Workload type: database node
```
### Summary:
Current state: busy but not under active memory pressure
Historical state: confirmed global OOM events and one cgroup OOM event
Primary current memory footprint: HugePages/database shared memory

## Chapter Files

- [01 - Current Memory Snapshot](01-current-memory-snapshot.md)
- [02 - Process Memory Analysis](02-process-memory-analysis.md)
- [03 - Process Memory Map Summary](03-process-memory-map-summary.md)
- [04 - Live Memory Pressure](04-live-memory-pressure.md)
- [05 - OOM Analysis](05-oom-analysis.md)
- [06 - cgroup Memory Analysis](06-cgroup-memory-analysis.md)
- [07 - Kernel Slab, HugePages, Swap](07-kernel-slab-hugepages-swap.md)
- [08 - Memory Timeline Collector](08-memory-timeline-collector.md)
