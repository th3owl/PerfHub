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
