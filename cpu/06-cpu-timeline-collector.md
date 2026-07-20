# CPU Timeline Collector

This chapter captures timestamped CPU and load samples using built-in Linux tools.

A current snapshot can show CPU state now. Historical CPU spike analysis requires timestamped data from tools like `sar`, `vmstat` logs, OSWatcher/ExaWatcher, or a manual collector.

## Why Timeline Data Matters

Live commands can miss the spike if the node has already recovered.

Timestamped data helps prove:

```text
when load started
when it peaked
whether it was CPU, I/O wait, D-state, memory, or swap
which processes were involved
whether recovery happened before or after application failure
```

## Case Example: OSWatcher Proves High Load Was I/O Wait

In one incident, live CPU commands alone would not have been enough. OSWatcher provided timestamped evidence showing why load average spiked.

### Timeline Evidence

```text
06:42:01  vmstat blocked queue b rises to 43-47
06:42:01  iowait rises to ~56%
06:42:31  iowait remains ~56-57%
06:43:01  blocked queue rises to ~112-113
06:43:01  iowait rises to ~72-76%
06:43:02  top shows load average around 60 and CPU wait around 74.6%
06:43:31  many Oracle processes are in D state
06:44:01  DB background processes including dbw, ckp, lg0, arc, and ctw are in D state
```

## Numeric Timeline From OSWatcher

| Time UTC | Evidence Source | Load / Queue | CPU Wait | Process State Evidence | Interpretation |
|---|---|---:|---:|---|---|
| 06:41:31 | OSWatcher `vmstat` | `b=16-23` | not yet peak | blocked queue rising | I/O wait beginning |
| 06:42:01 | OSWatcher `vmstat` | `b=43-47` | `wa=56%` | blocked queue jump | storage/I/O stall active |
| 06:42:31 | OSWatcher `vmstat` | `b=61-62` | `wa=56-57%` | blocked queue still rising | sustained I/O wait |
| 06:43:01 | OSWatcher `vmstat` | `b=112-113` | `wa=72-76%` | severe blocked queue | high load is I/O wait driven |
| 06:43:02 | OSWatcher `top` | load average `60.36` | `wa=74.6%` | high wait in top | load spike confirmed |
| 06:43:31 | OSWatcher `vmstat/top` | `b=124-126` | `wa=52-60%` | many Oracle processes in `D` state | DB processes blocked in kernel I/O |
| 06:44:01 | OSWatcher `top` | high load continuing | high wait | `ora_dbw`, `ora_ckp`, `ora_lg0`, `ora_arc`, `ora_ctw` in `D` state | database write/checkpoint/log/archive paths impacted |

## OS Messages Evidence

At the same time, OS messages showed iSCSI TCP connection closures and session recovery attempts.

| Time UTC | Evidence | Interpretation |
|---|---|---|
| 06:41:48 | iSCSI TCP connection closure messages begin | storage/network path instability begins before DB kill |
| 06:42-06:43 | many iSCSI sessions reopen to `169.254.2.x:3260` | storage sessions recovering while DB is stuck |
| 06:44:01-06:44:02 | connections reported operational after recovery attempts | storage path recovers after the DB hang window |

## Correlated Database Timeline

The OS-level stall aligned with database hang evidence.

| Time UTC | Evidence | Interpretation |
|---|---|---|
| 06:42:51 | `LG00` waited on `LGWR worker group ordering` for 70 seconds | LGWR worker path not progressing |
| 06:42:51 | `LG01` waited on `log file parallel write` for 70 seconds | redo write path blocked |
| 06:43:12 | `ORA-29770` for LG01 hung more than 70 seconds | database hang detector fires |
| 06:43:12-06:43:38 | `LMHB` terminates the instance | DB instance killed due to hung critical process |
| 06:43:13 | CRS detects DB abnormal termination | cluster layer observes downstream failure |
| 06:43:17 | CSSD shutdown on impacted node | cluster stack destabilizes after DB/storage stall |

## Interpretation

The high load was not caused by CPU saturation.

It was caused by storage/I/O stall:

```text
storage/iSCSI disruption
-> processes blocked in D state
-> iowait spike
-> load average spike
-> Oracle LGWR/LG01 hang
-> DB instance termination
```

## Why This Proves I/O Wait Instead Of CPU Saturation

The key evidence is:

```text
CPU wait was high: wa=56-76%
Blocked queue was high: b=43-126
Oracle processes were in D state
Storage messages showed iSCSI reconnect/recovery
DB wait was log file parallel write
```

CPU saturation would instead show:

```text
low idle CPU
high user/system CPU
many R-state runnable tasks
little or no D-state pileup
```

In this case, the spike was driven by blocked I/O, not lack of CPU cores.

## Key Takeaway

OSWatcher or similar timestamped OS data is extremely valuable for CPU/load analysis because it can prove whether high load came from:

```text
CPU run queue
I/O wait
D-state process pileup
memory pressure
swap pressure
```

A single live snapshot after recovery may miss the spike completely.
