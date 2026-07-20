# CPU Topology And Load

## Step 1: Load Average

Command:

```
uptime
```
Sample output:
```
11:05:01 up 418 days, 46 min,  1 user,  load average: 4294967324.18, 4294967326.29, 4294967325.10
```
#### Interpretation:
- This load average is not normal. The values are around 2^32, which suggests a counter overflow, kernel accounting issue, or corrupted load average display.
- A load average in the billions is not physically meaningful for CPU diagnosis.
#### Expert note:
Do not trust uptime load average blindly. Always validate load with CPU count, vmstat, process state, and /proc/loadavg.

## Step 2: Cross-Check Load Average

Commands:

```
cat /proc/loadavg
nproc
lscpu | egrep 'CPU\(s\)|Thread|Core|Socket|Model name|NUMA'
vmstat 1 5
```
Sample output:
/proc/loadavg:
```
4294967322.23 4294967324.68 4294967324.70 16/6973 175899
```
nproc:
```
252
```
lscpu:
```
CPU(s):              252
On-line CPU(s) list: 0-251
Thread(s) per core:  2
Core(s) per socket:  63
Socket(s):           2
NUMA node(s):        2
Model name:          AMD EPYC 7J13 64-Core Processor
NUMA node0 CPU(s):   0-125
NUMA node1 CPU(s):   126-251
```
vmstat summary:
```
r  = 19-24
b  = 0-1
us = 4-6%
sy = 1-3%
id = 90-95%
wa = 0%
st = 0%
```
#### Interpretation:
- The load average from uptime and /proc/loadavg is not physically meaningful. It is around 2^32, which suggests a kernel accounting/display issue or counter overflow.
- The node has 252 CPUs, and vmstat shows the run queue is only 19-24 with 90-95% idle CPU.
#### Conclusion:
The node is not CPU saturated. The load average is invalid for diagnosis on this node and must be cross-checked with vmstat, CPU count, process state, and actual CPU usage.

## Rule: Validate Load Average

Load average must be interpreted relative to CPU count and task state.

A high load average can come from:

```
Runnable CPU tasks
Uninterruptible I/O wait tasks
Kernel accounting issues
Stale or invalid values
```
Do not diagnose CPU pressure from load average alone.
