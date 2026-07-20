# CPU Investigation

This chapter covers Linux OS-level CPU and load analysis using built-in tools only.

The goal is to answer:

- How many CPUs does the node have?
- Is CPU currently saturated?
- Is load average valid or misleading?
- Is load caused by runnable CPU tasks, blocked I/O tasks, memory pressure, or swap?
- Which processes are using CPU?
- Which threads are using CPU inside a process?
- Can we build a timeline of CPU spikes and dips?

## Investigation Flow

1. CPU topology and load with `uptime`, `/proc/loadavg`, `nproc`, and `lscpu`
2. Process CPU usage with `ps`
3. Thread-level CPU analysis with `ps -L`
4. Live CPU pressure with `vmstat` and `mpstat` if available
5. High-load classification using `R` state, `D` state, `wchan`, memory, and swap checks
6. CPU timeline collection

## Chapter Files

- [01 - CPU Topology And Load](01-cpu-topology-and-load.md)
- [02 - Process CPU Usage](02-process-cpu-usage.md)
- [03 - Thread-Level CPU Analysis](03-thread-level-cpu-analysis.md)
- [04 - Live CPU Pressure](04-live-cpu-pressure.md)
- [05 - High Load Triage Checklist](05-high-load-triage-checklist.md)
- [06 - CPU Timeline Collector](06-cpu-timeline-collector.md)

## Current Case Summary

```text
Node: NODE-CPU-1
CPU count: 252
Load average: invalid value near 2^32
Run queue: low relative to CPU count
CPU idle: about 90-95%
I/O wait: 0%
Conclusion: node is not CPU saturated; load average is invalid for diagnosis
```
