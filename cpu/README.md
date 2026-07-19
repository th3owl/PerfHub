# CPU Investigation

This chapter covers Linux OS-level CPU analysis using built-in tools only.

The goal is to answer:

- How many CPUs does the node have?
- Is CPU currently saturated?
- Is load average high because of CPU, I/O wait, or runnable tasks?
- Which processes are using CPU?
- Which threads are using CPU inside a process?
- Can we build a timeline of CPU spikes and dips?

## Investigation Flow

1. CPU topology with `nproc` and `lscpu`
2. Load average with `uptime`
3. Live CPU pressure with `vmstat`
4. Top CPU processes with `ps`
5. Thread-level CPU with `ps -L`
6. Historical CPU timeline collector

## Chapter Files

- [01 - CPU Topology And Load](01-cpu-topology-and-load.md)
- [02 - Live CPU Pressure](02-live-cpu-pressure.md)
- [03 - Process CPU Usage](03-process-cpu-usage.md)
- [04 - Thread-Level CPU Analysis](04-thread-level-cpu-analysis.md)
- [05 - CPU Timeline Collector](05-cpu-timeline-collector.md)
