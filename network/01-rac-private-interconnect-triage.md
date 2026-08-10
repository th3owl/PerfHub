# RAC Private Interconnect Triage

This chapter records a live RAC private interconnect investigation using OS counters and trace behavior.

It is based only on the evidence captured in the live case.

## Why This Check Matters

When Oracle RAC trace files flood the trace mount, the issue may look like a storage or filesystem problem at first.

But sometimes the real driver is:

- private interconnect communication failure
- UDP buffer stress
- stale Oracle IPC connection state
- retry / retransmit / deferred-delete loops in Oracle trace

This chapter shows how to validate that from the OS side before jumping to conclusions.

## Commands Used

### Interface counters

```bash
ip -s link show enp1s0
```

Used to check:

- `errors`
- `dropped`
- `missed`

Questions answered:

- Are packets reaching the NIC but not being serviced normally?
- Are hard interface errors visible?
- Are counters actively increasing?

### UDP counters

```bash
netstat -su
```

Used to check:

- `packet receive errors`
- `receive buffer errors`
- `send buffer errors`

Questions answered:

- Are UDP queue/buffer problems happening?
- Is Oracle RAC traffic likely running into receive/send stress?

### Interface throughput

```bash
sar -n DEV 1 5
```

Used to check:

- traffic rate
- `%ifutil`

Questions answered:

- Is the NIC actually saturated?
- Or is utilization low enough that the issue is not raw bandwidth exhaustion?

### CPU and scheduler checks

```bash
vmstat 1 5
top -b -n 1 | head -40
```

Used to check:

- run queue
- blocked tasks
- CPU idle
- CPU wait
- whether Oracle processes are CPU-starved

Questions answered:

- Is the host too busy to service communication in time?
- Is the issue network-only, or is CPU pressure amplifying it?

## Live Case Data

## Private interface and UDP evidence

### Node `ce1pod-gb82p1`

Command outputs showed:

- `ip -s link show enp1s0`
  - `errors 0`
  - `dropped 0`
  - `missed 135800`

- `netstat -su`
  - `62156 packet receive errors`
  - `62154 receive buffer errors`
  - `1596748 send buffer errors`

30-second recheck:

- interface traffic continued increasing
- `missed` did not increase
- `receive buffer errors` did not increase
- `send buffer errors` increased only slightly: `1596743 -> 1596748`

### Node `ce1pod-gb82p3`

Command outputs showed:

- `ip -s link show enp1s0`
  - `errors 0`
  - `dropped 0`
  - `missed 111581`

- `netstat -su`
  - `1169 packet receive errors`
  - `1138 receive buffer errors`
  - `1932987 send buffer errors`

30-second recheck:

- interface traffic continued increasing
- `missed` did not increase
- `receive buffer errors` did not increase
- `send buffer errors` did not increase

### Node `ce1pod-gb82p4`

Command outputs showed:

- `ip -s link show enp1s0`
  - `errors 0`
  - `dropped 0`
  - `missed 459270`

- `netstat -su`
  - `446 packet receive errors`
  - `409 receive buffer errors`
  - `5456884 send buffer errors`

30-second recheck:

- interface traffic continued increasing
- `missed` did not increase
- `receive buffer errors` did not increase
- `send buffer errors` did not increase

## Throughput interpretation

The live comparison used `sar -n DEV` to answer whether the private NIC was actually saturated.

Across the nodes sampled in the incident, interface utilization remained low enough that the incident was not explained by raw link saturation alone.

Interpretation:

- traffic was active
- the interfaces were not proven full
- this was not a simple “NIC bandwidth maxed out” case

## How the counters were interpreted

### What `missed` means here

Non-zero `missed` counts on the private interface suggest that packets reached the interface path but were not serviced normally at receive time.

That is different from a pure CRC or hard-link failure.

### What UDP buffer errors mean here

The close relationship between:

- `packet receive errors`
- `receive buffer errors`

supports the interpretation that packets were arriving but not always being queued or processed cleanly in UDP receive buffers.

Large `send buffer errors` also matter because they align with Oracle trace symptoms such as:

- unacknowledged sends
- retransmits
- deferred deletes
- repeated negative acknowledgements

### Why the 30-second recheck matters

The short recheck is important because it separates:

- historical bad counters
- actively worsening counters

In this case:

- traffic was still flowing
- counters like `missed` and most UDP buffer error counters were not rising further during the 30-second window

That means:

- the interfaces had evidence of earlier stress
- but the current trace storm was not proven to be caused by a continuously worsening live NIC drop storm at the exact sampling moment

## Correlation with Oracle trace behavior

The trace evidence in the incident showed repeated Oracle IPC symptoms such as:

- unacknowledged sends
- retransmits
- deferred delete processing
- `Deferred NAK`
- `ERRCHK PKT`

The OS/network counter pattern supported that story because:

- private interfaces had non-zero historical `missed` counts
- UDP receive/send buffer errors existed
- traffic was still flowing
- counters were not exploding further during the short recheck

That combined pattern is consistent with:

- an earlier interconnect / IPC disturbance
- Oracle channels left in bad or stale peer state
- trace growth sustained by Oracle retry and deferred-delete loops

## Key Distinction

This case is a good example of the difference between:

### Raw NIC saturation

Typical signs would be:

- very high interface utilization
- actively growing interface drops/errors
- line-rate style exhaustion

### Oracle IPC retry storm after earlier disturbance

Typical signs would be:

- traffic still flowing
- historical `missed` or UDP buffer stress visible
- counters not rapidly worsening during short recheck
- Oracle traces still flooding with retry/deferred-delete symptoms

This live case matched the second pattern more closely.

## Reusable Investigation Pattern

For future RAC private interconnect incidents, use this order:

1. Check private NIC counters with `ip -s link`
2. Check UDP errors with `netstat -su`
3. Check whether the NIC is actually saturated with `sar -n DEV`
4. Check CPU and scheduler state with `vmstat` and `top`
5. Repeat the interface and UDP counters after 30 seconds
6. Compare all RAC nodes, not just one node
7. Correlate OS counters with Oracle trace symptoms

## Practical Takeaway

From this live use case, the evidence supports:

- not a pure bandwidth saturation problem
- yes to historical private-side packet handling / UDP buffer stress
- yes to Oracle IPC retry/deferred-delete storm behavior continuing after the initial disturbance
