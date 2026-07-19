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
