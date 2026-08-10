# Network Investigation

This chapter covers Linux OS-level network and RAC private interconnect analysis using built-in tools and live OS counters.

The goal is to answer:

- Is the network interface actually saturated?
- Are packets being missed or dropped at the interface level?
- Are UDP receive/send buffer errors happening?
- Is the host CPU-starved and amplifying communication issues?
- Is the issue a raw NIC bandwidth problem, or an Oracle/RAC IPC retry storm after an earlier disturbance?
- Can we correlate OS counters with Oracle trace behavior?

## Investigation Flow

1. Interface counters with `ip -s link`
2. UDP counters with `netstat -su`
3. Interface throughput with `sar -n DEV`
4. CPU and scheduler state with `vmstat` and `top`
5. Multi-node comparison across RAC private interfaces
6. Trace correlation with Oracle IPC symptoms
7. Short-interval recheck to distinguish historical counters from actively worsening counters

## Case Study

Current working case:

- Node family: `db*`
- Workload type: RAC private interconnect trace storm
- Focus area: `enp1s0` private interface and Oracle IPC traces

## Summary

- The observed issue was not explained by NIC bandwidth saturation.
- Private-interface counters showed historical `missed` packets and UDP buffer errors.
- A 30-second recheck showed traffic still flowing, while most error counters were not actively increasing.
- That pattern supported an Oracle/RAC IPC retry and deferred-delete trace storm sustained after an earlier disturbance, rather than a continuously worsening live NIC drop storm.

## Chapter Files

- [01 - RAC Private Interconnect Triage](01-rac-private-interconnect-triage.md)
