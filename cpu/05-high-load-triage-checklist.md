# High Load Triage Checklist

This checklist classifies high load into CPU run queue, I/O wait, memory pressure, swap pressure, or invalid load accounting.

High load is not automatically high CPU.

## Step 1: Count Runnable Tasks

Command:

```bash
ps -eo pid,user,pri,nice,vsize:10,rss:10,%mem:5,%cpu:5,state,wchan:14,stime,cputime:10,command --sort=-%cpu | awk '$9=="R"{count++} END{print count+0}'
```

Interpretation:

`R` means running or runnable.

Compare the count with CPU count.

```text
R count below CPU count  = usually okay
R count near CPU count   = CPU busy
R count above CPU count  = CPU queueing
```

## Step 2: Count D-State Tasks

Command:

```bash
ps -eo pid,user,pri,nice,vsize:10,rss:10,%mem:5,%cpu:5,state,wchan:14,stime,cputime:10,command --sort=-%cpu | awk '$9=="D"{count++} END{print count+0}'
```

Interpretation:

`D` means uninterruptible sleep. These tasks are often waiting on disk, storage, filesystem, or kernel I/O.

High load with many `D` tasks is usually not pure CPU saturation.

## Step 3: Group D-State Tasks By Wait Channel

Command:

```bash
ps -eo pid,user,pri,nice,vsize:10,rss:10,%mem:5,%cpu:5,state,wchan:14,stime,cputime:10,command --sort=-%cpu | awk '$9=="D"{print $10}' | sort | uniq -c | sort -nr
```

Interpretation:

`wchan` shows the kernel function where a task is waiting.

This helps classify blocked load as:

```text
storage wait
filesystem wait
network wait
kernel lock wait
```

## Step 4: Show Top CPU Consumers

Command:

```bash
ps -eo pid,user,pri,nice,vsize:10,rss:10,%mem:5,%cpu:5,state,wchan:14,stime,cputime:10,command --sort=-%cpu | head -30
```

Use this to identify the highest CPU consumers.

## Step 5: Show Top Memory Consumers

Command:

```bash
ps -eo pid,tid,ppid,user,pri,nice,vsize:10,rss:10,%mem:5,%cpu:5,state,wchan:14,stime,cputime:10,command --sort=-rss | head -30
```

Use this when high load may be related to memory pressure or swap.

## Step 6: Check Live CPU, I/O Wait, And Swap

Command:

```bash
vmstat 1 5
```

Important fields:

```text
r  = run queue
b  = blocked tasks
si = swap in
so = swap out
us = user CPU
sy = system CPU
id = idle CPU
wa = I/O wait
st = steal
```

## Step 7: Use Historical `sar` If Already Available

Commands:

```bash
sar -q
sar -u
sar -r
sar -S
sar -W
```

Meaning:

```text
sar -q = load/run queue history
sar -u = CPU history
sar -r = memory history
sar -S = swap usage history
sar -W = swap page-in/page-out history
```

Do not install tools during emergency triage just for this runbook. Use `sar` only if it is already available and collecting data.

## Step 8: Oracle-Specific Follow-Up

If the top CPU process is Oracle, map the OS process to DB session, PDB, and workload.

Check active sessions by instance:

```sql
select inst_id,status,count(*) session_count
from gv$session
where type <> 'BACKGROUND'
group by inst_id,status
order by inst_id,session_count,status;
```

Check active sessions by PDB:

```sql
select p.name, count(*) active_sessions
from v$session s, v$pdbs p
where s.type <> 'BACKGROUND'
and s.status = 'ACTIVE'
and s.con_id = p.con_id
group by p.name
order by 2 desc;
```

Check PDB CPU allocation distribution:

```sql
select p.inst_id, count(p.name) open_pdbs, sum(s.value) total_cpu_count
from gv$system_parameter s, gv$pdbs p
where s.inst_id = p.inst_id
and s.con_id = p.con_id
and p.open_mode = 'READ WRITE'
and s.name = 'cpu_count'
group by p.inst_id
order by 1;
```

## Load Classification

```text
High R tasks + low idle CPU
  CPU saturation

High D tasks + high iowait or storage wchan
  I/O or kernel wait pressure

Swap si/so non-zero
  Active memory pressure or reclaim issue

High memory usage + low MemAvailable
  Memory pressure may be contributing to load

Impossible load value
  Cross-check with vmstat, mpstat, process state, and /proc/loadavg
```

## Current Case Classification

For the current case:

```text
/proc/loadavg: invalid value near 2^32
CPU count: 252
R tasks: about 22
D tasks: 0 in state distribution
mpstat idle: about 92%
iowait: 0%
```

Conclusion:
The node is not under real high-load pressure. The load average is invalid for diagnosis on this host.

## Historical Evidence

For historical load incidents, prefer timestamped sources:

```text
OSWatcher
ExaWatcher
sar
vmstat logs
top snapshots
system messages
```
These can prove whether the spike was CPU saturation or I/O wait.
