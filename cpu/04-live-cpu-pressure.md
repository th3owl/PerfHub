
## Step 1: Cross-Check CPU With `mpstat`

Command:

```bash
mpstat 1 5
```

Sample output:

```text
Average: all  %usr 5.27  %sys 2.28  %iowait 0.00  %steal 0.02  %idle 92.18
```

## Interpretation

`mpstat` confirms the node is not CPU saturated.

Important fields:

`%usr`
CPU used by user-space/application code.

`%sys`
CPU used by kernel code.

`%iowait`
CPU waiting for I/O.

`%steal`
CPU time taken by the hypervisor.

`%idle`
Unused CPU capacity.

In this sample:

```text
%idle = 92.18
%iowait = 0.00
%steal = 0.02
```

Conclusion:

The node has plenty of idle CPU. The high load average is invalid for diagnosis on this host.
For your CPU case summary so far:
## CPU Case Summary: NODE-CPU-1

### Current Snapshot

The load average is invalid or corrupted:

```text
load average: ~4294967324
/proc/loadavg: ~4294967322
The value is near 2^32, so it is not physically meaningful.
Cross-Checks
CPU count: 252
vmstat run queue: 19-24
process state R: 22
D-state tasks: 0
mpstat idle: 92.18%
mpstat iowait: 0.00%
mpstat steal: 0.02%
Process CPU
oracle CPU by user: ~2440% = ~24.4 CPU cores
top process: ora_scmn/ofsd around 123%
```
### Conclusion
The node is not CPU saturated. Oracle owns most current CPU usage, but total usage is small relative to 252 CPUs. The load average is invalid and should not be used alone for alerting or diagnosis.
