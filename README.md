# Linux Resource Investigation Runbook

This repository records a command-by-command Linux OS-level investigation flow for memory, CPU, network, process lifecycle, and resource spike analysis.

The goal is to build an expert-level operational runbook using built-in Linux tools first, without installing external utilities.

## Scope

- Process lifecycle and resource usage
- Node-level memory consumption
- Node-level CPU consumption
- Node-level network and interconnect behavior
- Per-process CPU and memory usage
- Timeline creation for spikes and dips
- Evidence-based interpretation of why resource usage changed

## Chapters

- [Memory Investigation](memory/README.md)
- [CPU Investigation](cpu/README.md)
- [Network Investigation](network/README.md)
- Process Lifecycle

## Built-In Linux Evidence Sources

```text
/proc
ps
top
free
vmstat
dmesg
journalctl
systemctl
cgroup files
ip
netstat
sar/sysstat if already present
iostat if already present
lsblk
multipath if already present
OSWatcher / ExaWatcher if already present
```

## Current Status

- Memory investigation: drafted
- CPU and load investigation: drafted
- Network investigation: drafted
- Process lifecycle: pending
