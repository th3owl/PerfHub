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

## Recommended OSWatcher Discovery Order

1. Check common directories:

```bash
ls -ld /u01/logs/oswatcher /opt/oracle.ExaWatcher /opt/oracle.oswatcher /opt/oswbb /u01/app/oracle/tfa/repository 2>/dev/null
```

2. Check running watcher process:

```bash
ps -ef | grep -Ei 'oswatch|oswbb|exawatch|exawatcher' | grep -v grep
```

3. If a watcher process is running, use the output directory shown in the command line.

4. Search inside that directory:

```bash
find /u01/logs/oswatcher -type f | head -50
```

5. Narrow to expected watcher files:

```bash
find /u01/logs/oswatcher -type f \( -iname '*vmstat*' -o -iname '*top*' -o -iname '*iostat*' -o -iname '*ps*' \) | head -50
```

6. Use broad filesystem `find` only if the location is still unknown:

```bash
find /opt /u01 /var /tmp -type d \( -iname '*oswatch*' -o -iname '*oswbb*' -o -iname '*exawatch*' -o -iname '*exawatcher*' \) 2>/dev/null
```

## Example: OSWatcher Discovery On NODE-OSW-1

### Check For OSWatcher Directory

Command:

```bash
find /opt /u01 /var /tmp -type d \( -iname '*oswatch*' -o -iname '*oswbb*' -o -iname '*exawatch*' -o -iname '*exawatcher*' \) 2>/dev/null
```

Output:

```text
/u01/logs/oswatcher
```

Interpretation:

OSWatcher data directory exists at:

```text
/u01/logs/oswatcher
```

### Check Whether OSWatcher Is Running

Command:

```bash
ps -ef | grep -Ei 'oswatch|oswbb|exawatch|exawatcher' | grep -v grep
```

Output:

```text
root 2789 1 0 Jul01 ? 00:07:48 /bin/sh /usr/sbin/OSWatcher 30 168 NONE /u01/logs/oswatcher
```

Interpretation:

OSWatcher is running.

The command line shows:

```text
/usr/sbin/OSWatcher 30 168 NONE /u01/logs/oswatcher
```

Meaning:

```text
sample interval: 30 seconds
retention/count argument: 168
output location: /u01/logs/oswatcher
```

So this node is collecting OSWatcher samples every 30 seconds.

### Common Location Check

Command:

```bash
ls -ld /opt/oracle.ExaWatcher /opt/oracle.oswatcher /opt/oswbb /u01/app/oracle/tfa/repository 2>/dev/null
```

Output:

```text
no output
```

Interpretation:

The common ExaWatcher/OSWatcher/TFA locations checked here do not exist or are not readable on this node.

However, OSWatcher still exists in a custom/current location:

```text
/u01/logs/oswatcher
```

### Broad File Search Caveat

Command:

```bash
find /opt /u01 /var /tmp -type f \( -iname '*osw*.tar*' -o -iname '*osw*.gz' -o -iname '*osw*.dat' -o -iname '*vmstat*' -o -iname '*top*' \) 2>/dev/null | head -5
```

Output:

```text
/opt/impairment_agent/bin/impairment_agent.x86_64/_internal/gevent-25.5.1.dist-info/top_level.txt
/opt/impairment_agent/bin/impairment_agent.x86_64/_internal/h2-4.3.0.dist-info/top_level.txt
/opt/impairment_agent/bin/impairment_agent.x86_64/_internal/markupsafe-3.0.3.dist-info/top_level.txt
/opt/impairment_agent/bin/impairment_agent.x86_64/_internal/zope_event-6.2.dist-info/top_level.txt
/opt/impairment_agent/bin/impairment_agent.x86_64/_internal/setuptools/_vendor/importlib_metadata-8.7.1.dist-info/top_level.txt
```

Interpretation:

This broad search produced false positives because files named `top_level.txt` matched `*top*`.

Better approach after finding the OSWatcher directory:

```bash
find /u01/logs/oswatcher -type f | head -50
```

Then narrow to expected files:

```bash
find /u01/logs/oswatcher -type f \( -iname '*vmstat*' -o -iname '*top*' -o -iname '*iostat*' -o -iname '*ps*' \) | head -50
```

### Discovery Key Takeaway

When OSWatcher is running, prefer the configured output directory from the process command line.

In this case:

```text
/u01/logs/oswatcher
```

is more reliable than a broad filesystem search.

## What To Extract From OSWatcher

For suspected high load, extract data around the incident window.

Recommended window:

```text
3 hours before incident
1 hour after incident
```

Useful watcher files:

```text
vmstat
top
iostat
ps
netstat
messages
```

Minimum fields to capture:

```text
timestamp
load average
vmstat r
vmstat b
vmstat us/sy/id/wa/st
top CPU summary
top process states
D-state processes
iostat await/util if available
OS messages around the same time
```
----------------------------------------------------------------------------
## Case Example: OSWatcher Proves High Load Was I/O Wait
----------------------------------------------------------------------------

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

## Historical `sar` Data For CPU And Load

If `sar` is already installed and collecting data, it is one of the most useful sources for historical CPU and load analysis.

### Commands

```bash
sar -q
sar -u
sar -w
```
Meaning
```
sar -q = load average and run queue history
sar -u = CPU utilization history
sar -w = task creation and context switch history
```
What To Check
`sar -q`
Use this to see:
```
when load average increased
when run queue increased
whether blocked processes increased
```
`sar -u`
Use this to see:
```
whether CPU was busy in user space
whether system CPU increased
whether idle CPU dropped
whether iowait was high
```
`sar -w`
Use this to see:
```
whether context switches increased
whether process creation spiked
```
#### Interpretation
```
high load with high idle CPU
  load may be misleading, blocked, or I/O related

high load with low idle CPU and high user/system CPU
  CPU saturation is more likely

high load with high iowait
  load is likely storage or I/O driven

run queue rising above CPU capacity
  CPU queueing is likely
```
Example Workflow
```
sar -q
sar -u
sar -w
```
Then correlate with:
- vmstat
- mpstat
- process state distribution
- OSWatcher/ExaWatcher
- application/database logs
#### Key Takeaway
Use sar when the CPU or load spike happened in the past and live commands now look normal.

## Example: Historical `sar` Data For CPU And Load

Commands:

```bash
sar -q 1 3
sar -u 1 3
sar -w 1 3
```
Sample data:
```
sar -q 1 3
Linux 5.15.0-308.179.6.16.el8uek.x86_64 (xxx) 	07/22/26 	_x86_64_	(380 CPU)

09:26:22      runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
09:26:25           22     10644     16.63     18.66     23.28         0
09:26:28           16     10639     16.63     18.66     23.28         1
09:26:31           15     10636     16.26     18.55     23.22         1
Average:           18     10640     16.51     18.62     23.26         1

sar -u 1 3
Linux 5.15.0-308.179.6.16.el8uek.x86_64 (xxx) 	07/22/26 	_x86_64_	(380 CPU)

09:26:42        CPU     %user     %nice   %system   %iowait    %steal     %idle
09:26:43        all      6.03      0.00      0.65      0.08      0.00     93.23
09:26:44        all      4.77      0.01      0.56      0.16      0.04     94.46
09:26:45        all      6.96      0.01      1.09      0.18      0.00     91.77
Average:        all      5.92      0.01      0.77      0.14      0.01     93.16

sar -w 1 3
Linux 5.15.0-308.179.6.16.el8uek.x86_64 (xxx) 	07/22/26 	_x86_64_	(380 CPU)

09:26:50       proc/s   cswch/s
09:26:51        16.00 360365.00
09:26:52        57.00 361474.00
09:26:53        22.00 293128.00
Average:        31.67 338322.33
```
Sample interpretation:
```
Average runq-sz : 18
Average blocked : 1
Load average    : 16.51 / 18.62 / 23.26
Average %idle   : 93.16
Average %user   : 5.92
Average %system : 0.77
Average %iowait : 0.14
```
On a 380 CPU node:
- run queue 18 is low relative to CPU capacity
- idle CPU is very high
- iowait is negligible
#### Conclusion:
- The node is not CPU saturated.
- Historical load values must be interpreted relative to CPU count.
- This is a healthy CPU profile for a large node.
----------------------------------------------------------------------------
## Case Example: Scheduler And Parallel Query Caused True CPU Saturation
----------------------------------------------------------------------------

This incident shows a high-load case where the node was CPU-bound, not I/O-bound.

### Host And Time

- Host: `node4`

### Command 1: Load Average

`uptime`

Output:
```
[root@enode4 ~]# uptime
 05:16:15 up 31 days, 16:38,  2 users,  load average: 172.83, 147.21, 128.94
```
Interpretation:
- Load average was extremely high.
- On its own this does not prove CPU saturation, so more evidence was needed.
- 
### Command 2: Kernel Load View
`cat /proc/loadavg`

Output:
```
[root@node4 ~]# cat /proc/loadavg
164.59 148.52 130.17 141/6221 268499
```
Interpretation:
- The instantaneous runnable/active task view was very high.
- 141/6221 showed many tasks in the system, with a large active runnable component.
  
### Command 3: CPU Count
`nproc`

Output:
```
[root@node4 ~]# nproc
100
```
Interpretation:
- The host had 100 CPUs.
- A load average around 173 on a 100 CPU host is suspicious, but we still need CPU-state evidence to know whether this was CPU, I/O, or blocked work.

### Command 4: Live Scheduler / Run Queue View

`vmstat 1 5`
Output:
```
[root@node4 ~]# vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
141  0  25088 18481304 909916 239035920    0    0    27    15    0    0 16  4 80  0  0

156  0  25088 18846392 909916 239035968    0    0     0   252 436557 190965 83 16  1  0  0
158  0  25088 19683796 909916 239036000    0    0     0   284 450536 224927 82 17  1  0  0
161  0  25088 20070412 909916 239036048    0    0     0   204 410775 170683 83 16  1  0  0
142  0  25088 20382780 909916 239036128    0    0     0   684 411589 175295 83 16  1  0  0
```
Interpretation:
- r=141-161 was far above the CPU count pressure threshold.
- b=0 showed this was not a blocked-I/O style event.
- us=82-83, sy=16-17, id=1, wa=0 is the classic pattern of CPU saturation.
- There was no meaningful I/O wait component.
- 
### Command 5: CPU Breakdown

`mpstat 1 5`
Output:
```
[root@node4 ~]# mpstat 1 5
Linux 5.4.17-2136.322.6.5.el8uek.x86_64 (node4) 	08/19/2026 	_x86_64_	(100 CPU)

05:17:02 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:17:03 AM  all   83.38    0.25    7.09    0.00    5.22    3.09    0.26    0.00    0.00    0.71
05:17:04 AM  all   81.69    0.13    7.64    0.00    5.16    3.64    0.16    0.00    0.00    1.58
05:17:05 AM  all   79.30    0.09    9.63    0.00    5.13    3.98    0.20    0.00    0.00    1.67
05:17:06 AM  all   78.43    0.12   10.23    0.00    5.08    3.93    0.21    0.00    0.00    2.00
05:17:07 AM  all   78.69    0.08   10.28    0.01    5.24    4.09    0.18    0.00    0.00    1.43
Average:     all   80.30    0.13    8.97    0.00    5.17    3.75    0.20    0.00    0.00    1.48
```
Interpretation:
- CPU was almost fully consumed.
- %idle averaged only 1.48.
- %iowait was effectively 0.00.
- This strongly proved that the high load was caused by CPU demand, not storage wait.

### Command 6: Process State Distribution
```
ps -eo stat= | awk '{s=substr($1,1,1); count[s]++} END {for (s in count) print s, count[s]}' | sort
```
Output:
```
[root@node4 ~]# ps -eo stat= | awk '{s=substr($1,1,1); count[s]++} END {for (s in count) print s, count[s]}' | sort
D 2
I 2017
R 149
S 2896
```
Interpretation:
- R 149 matched the high run queue.
- Only D 2 processes existed, so this was not a D-state pileup or storage-stall case.
- The load was coming from runnable work competing for CPU.

### Command 7: Top CPU Consumers
```
ps -eo pid,ppid,user,stat,comm,pcpu,pmem,rss,vsz,etime,args --sort=-pcpu | head -40
```
Output:
```
[root@node4 ~]# ps -eo pid,ppid,user,stat,comm,pcpu,pmem,rss,vsz,etime,args --sort=-pcpu | head -40
   PID   PPID USER     STAT COMMAND         %CPU %MEM   RSS    VSZ     ELAPSED COMMAND
210885      1 oracle   Rs   ora_j00e_em31po 89.7  0.0 216796 318874856 3-05:48:18 ora_j00e_em31pod6
217374      1 oracle   Rs   ora_j00n_em31po 74.7  0.0 327728 319283500 06:50:40 ora_j00n_em31pod6
207333      1 oracle   Rs   ora_j00z_em31po 48.2  0.0 255068 319230380   20:41 ora_j00z_em31pod6
190729      1 oracle   Rs   ora_j00a_em31po 47.8  0.0 414068 319349892   26:09 ora_j00a_em31pod6
273035      1 oracle   Rs   oracle_273035_e 39.6  0.0 138888 319089996   00:04 oracleem31pod6 (LOCAL=NO)
272762      1 oracle   Rs   oracle_272762_e 40.5  0.0 140148 319090264   00:12 oracleem31pod6 (LOCAL=NO)
108217      1 oracle   Rs   ora_j007_em31po 38.8  0.0 415356 319547256   51:54 ora_j007_em31pod6
231834      1 oracle   Ss   ora_j00f_em31po 37.3  0.0 319132 319329032   13:16 ora_j00f_em31pod6
212582      1 oracle   Rs   ora_j004_em31po 35.4  0.0 423544 319363840   18:59 ora_j004_em31pod6
231844      1 oracle   Rs   ora_j00o_em31po 34.9  0.0 360440 319479936   13:16 ora_j00o_em31pod6
154309      1 oracle   Rs   oracle_154309_e 34.0  0.1 998720 320092248   37:26 oracleem31pod6 (LOCAL=NO)
161915      1 oracle   Rs   oracle_161915_e 33.7  0.0 535904 319494424   34:59 oracleem31pod6 (LOCAL=NO)
215152      1 oracle   Rs   oracle_215152_e 33.4  0.0 499928 319624932   18:17 oracleem31pod6 (LOCAL=NO)
260762      1 oracle   Rs   oracle_260762_e 33.4  0.0 207532 319181928   04:02 oracleem31pod6 (LOCAL=NO)
154676      1 oracle   Rs   oracle_154676_e 33.2  0.1 947796 320060344   37:12 oracleem31pod6 (LOCAL=NO)
157099      1 oracle   Rs   oracle_157099_e 32.9  0.1 947480 320060340   36:41 oracleem31pod6 (LOCAL=NO)
166939      1 oracle   Rs   ora_j002_em31po 32.9  0.0 384788 319349804   33:19 ora_j002_em31pod6
155839      1 oracle   Ss   oracle_155839_e 32.0  0.1 948384 320060344   36:57 oracleem31pod6 (LOCAL=NO)
266674      1 oracle   Rs   oracle_266674_e 32.0  0.0 161036 319116488   01:48 oracleem31pod6 (LOCAL=NO)
273063      1 oracle   Rs   oracle_273063_e 31.6  0.0 138052 319089868   00:02 oracleem31pod6 (LOCAL=NO)
214807      1 oracle   Ss   ora_j000_em31po 29.4  0.0 447452 319431864   18:19 ora_j000_em31pod6
284241      1 oracle   Ss   ora_mrm_em31pod 29.3  0.0 83016 316436788 3-21:20:13 ora_mrm_em31pod6
 61662      1 oracle   Rs   ora_j00y_em31po 27.5  0.0 377188 319339420 01:08:18 ora_j00y_em31pod6
284279      1 oracle   Ssl  ora_scmn_em31po 26.9  0.0 366332 319366256 3-21:20:13 ora_lms1_em31pod6
119459      1 oracle   Rs   ora_j00b_em31po 26.7  0.0 201888 319148928   48:19 ora_j00b_em31pod6
214980      1 oracle   Ss   ora_j01h_em31po 25.9  0.0 230152 319182852   18:18 ora_j01h_em31pod6
284774      1 oracle   Ss   ora_ppa7_em31po 25.6  0.1 885280 319715360 3-21:20:07 ora_ppa7_em31pod6
211552      1 oracle   Ss   ora_j00i_em31po 24.9  0.0 258188 319270680   19:21 ora_j00i_em31pod6
273079      1 oracle   Rs   oracle_273079_e 31.0  0.0 125064 319091748   00:01 oracleem31pod6 (LOCAL=NO)
367772      1 oracle   Ss   ora_j012_em31po 23.4  0.0 217768 319139364 01:38:18 ora_j012_em31pod6
284414      1 oracle   Ss   ora_lgwr_em31po 21.7  0.0 113604 316469544 3-21:20:12 ora_lgwr_em31pod6
 98327      1 oracle   Rs   ora_j00q_em31po 21.5  0.0 226216 319295964   55:40 ora_j00q_em31pod6
272865      1 oracle   Ss   oracle_272865_e 20.7  0.0 146764 319102952   00:09 oracleem31pod6 (LOCAL=NO)
284281      1 oracle   Ssl  ora_scmn_em31po 21.0  0.0 361092 319363184 3-21:20:13 ora_lms2_em31pod6
284283      1 oracle   Ssl  ora_scmn_em31po 21.0  0.0 363600 319363248 3-21:20:13 ora_lms3_em31pod6
109411      1 oracle   Ss   ora_j00g_em31po 20.9  0.0 252288 319170360   51:40 ora_j00g_em31pod6
284277      1 oracle   Ssl  ora_scmn_em31po 20.9  0.0 362504 319363248 3-21:20:13 ora_lms0_em31pod6
284285      1 oracle   Ssl  ora_scmn_em31po 20.8  0.0 368996 319366256 3-21:20:13 ora_lms4_em31pod6
248474      1 oracle   Ss   oracle_248474_e 20.5  0.0 156524 319110064   08:16 oracleem31pod6 (LOCAL=NO)
```
Interpretation:
- CPU consumption was dominated by Oracle processes.
- Many ora_j... job processes were active.
- Many oracle...(LOCAL=NO) foregrounds were also active.
- This immediately pointed toward database workload rather than an OS daemon or kernel issue.

Database Correlation
- The following database evidence was collected during the same incident.
  
### Command 8: Active Waits By Instance And PDB
```
select inst_id, con_id, event, wait_class, count(*)
from gv$session
where status = 'ACTIVE'
group by inst_id, con_id, event, wait_class
order by 1, 5 desc;
```
Relevant findings:
```
INST_ID CON_ID EVENT                         WAIT_CLASS    COUNT(*)
------- ------ ----------------------------  ------------  --------
6       193    resmgr:cpu quantum           Scheduler     38
6       193    PX Deq: Table Q Normal       Idle          32
6       193    PX Deq: Execution Msg        Idle          31
6       193    cell smart table scan        User I/O      27
```
Interpretation:
- Instance 6 was the hotspot.
- CON_ID 193 was the main workload container.
- resmgr:cpu quantum strongly indicated CPU contention.
- PX dequeue waits showed heavy parallel execution activity.

### Command 9: Map PDB Name
```
select con_id, name
from v$containers
where con_id in (117,155,174,176,193);
```
Relevant output:
```
CON_ID NAME
------ ------------------------------
193    STOREDB
```
Interpretation:
- The busiest PDB in this incident was STOREDB.

### Command 10: SQL / Module / Program Distribution
```
select inst_id, sql_id, module, program, count(*)
from gv$session
where status = 'ACTIVE'
group by inst_id, sql_id, module, program
order by 5 desc;
```
Interpretation:
- The active workload was heavily associated with scheduler-driven and parallelized database activity.
- Repeated Oracle PX worker programs and scheduler-linked sessions were present.

### Command 11: Parallel Execution Fan-Out
```
select degree, req_degree, server_set, count(*)
from v$px_session
group by degree, req_degree, server_set
order by 4 desc;
```
Output:
```
     DEGREE      REQ_DEGREE      SERVER_SET        COUNT(*)
----------- --------------- --------------- ---------------
        225             450               1              64
        225             450               2              64
        450             450               1              64
          8               8               1               9
         35              70               2               8
         35              70               1               8
         43              86               2               5
         43              86               1               5
                                                          4
```
Interpretation:
- This was very large PX fan-out.
- The system was running highly parallel database work.
- That level of parallelism can easily saturate a 100 CPU host.

### Command 12: Running Scheduler Jobs In The Hot PDB
```
select owner, job_name, session_id, running_instance, elapsed_time, cpu_used
from cdb_scheduler_running_jobs
where con_id = 193;
```
Output:
```
OWNER                JOB_NAME                                 SESSION_ID RUNNING_INSTANCE ELAPSED_TIME              CPU_USED
-------------------- ---------------------------------------- ---------- ---------------- ------------------------- -------------------------
ADMIN                STATS_JOB_1                                   52918                6 +000 00:20:42.68        +004 13:34:55.91
ADMIN                STATS_JOB_2                                   13849                6 +000 06:29:59.44        +000 00:00:01.25
```
Interpretation:
- Multiple scheduler jobs were active in the same PDB.
- STATS_JOB_1 stood out as a major contributor.
- The CPU saturation was not caused by one random foreground session. It aligned with scheduled background processing.

### Command 13: Job Definition And Schedule
```
select owner, job_name, enabled, state, start_date, last_start_date, repeat_interval
from cdb_scheduler_jobs
where job_name = 'STATS_JOB_1'
and con_id = 193;
```
Output:
```
OWNER
--------------------------------------------------------------------------------------------------------------------------------
JOB_NAME                                                         ENABL STATE
-------------------------------------------------------------------------------------------------------------------------------- ----- ---------------
START_DATE                                    LAST_START_DATE
--------------------------------------------------------------------------- ---------------------------------------------------------------------------
REPEAT_INTERVAL
----------------------------------------------------------------------------------------------------------------------------------------------------------------
ADMIN
STATS_JOB_1                                                      TRUE  SCHEDULED
14-JUL-26 11.10.42.765624 AM +00:00               19-AUG-26 05.10.42.433622 AM +00:00
FREQ=HOURLY;BYDAY=MON,TUE,WED,THU,FRI,SAT,SUN
```
Interpretation:
- This job was not newly introduced on the incident day.
- It had existed since 2026-07-14.
- It was an hourly recurring job.
- Its most recent run started at 2026-08-19 05:10:42 UTC, aligning with the incident window.

### Command 14: SQL Text For Main Workload
```
select sql_id, sql_text
from v$sql
where sql_id in ('ab123cd456','ef123gh456');
```
Output:
```
SQL_ID
-------------
SQL_TEXT
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ab123cd456
DELETE FROM USAGE_DETAILS TARGET WHERE EXISTS ( SELECT 1 FROM USAGE_DETAILS_DELAYED_STAGE STAGE WHERE STAGE.TIMEID = TARGET.TIMEID ) AND ROWNUM <= 50000

ef123gh456
/* SQL Analyze(1) */ select /*+  full(t)    no_parallel(t) no_parallel_index(t) dbms_stats cursor_sharing_exact use_weak_name_resl dynamic_sampling(0) no_monitoring xmlindex_sel_idx_tbl opt_param('optimizer_inmemory_aware' 'false') no_substrb_pad bypass_recursive_check */ ...
```
Interpretation:
- The workload included maintenance/statistics-style work and usage-fact processing.
- This matched the scheduler-job pattern already visible in the running job list.

## Final Interpretation
This was a true CPU saturation incident.
The evidence chain was:
```
hourly scheduler workload in PDB STOREDB
-> multiple active scheduler jobs
-> large PX fan-out
-> many runnable Oracle processes
-> run queue 141-161 on a 100 CPU host
-> idle CPU ~1-2%
-> iowait ~0%
-> resmgr:cpu quantum waits
-> load average rose to ~173
```
## Why This Was Not A Storage-Issue Load Spike
This case did not match the pattern of storage wait:
- b=0 in vmstat
- only D 2 processes
- %iowait was effectively zero
- %idle was almost zero
- run queue was extremely high
- Oracle sessions showed CPU scheduling pressure with resmgr:cpu quantum

That combination proves CPU oversubscription, not blocked I/O.
### Key Takeaway
High load average does not always mean disk trouble.
In this case, the deciding signals were:
```
load average ~173 on 100 CPUs
run queue 141-161
idle CPU ~1.48%
iowait ~0%
149 runnable tasks
Oracle scheduler jobs active
very large PX fan-out
resmgr:cpu quantum waits
```
This is a clean example of true CPU saturation caused by scheduled, highly parallel Oracle workload.
