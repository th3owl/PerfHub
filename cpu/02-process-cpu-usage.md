# Process CPU Usage

## Step 1: Top CPU Processes

Command:

```
ps -eo pid,ppid,user,stat,comm,pcpu,pmem,rss,vsz,etime,args --sort=-pcpu | head -20
```
Sample output:
```
PID     USER    STAT COMMAND          %CPU
356924  oracle  Ssl  ora_scmn...       123
200116  oracle  Rs   oracle_200116...  99.4
39502   oracle  Ss   oracle_39502...   89.8
201423  oracle  Rs   oracle_201423...  98.4
201459  oracle  Rs   oracle_201459...  97.8
199438  oracle  Rs   ora_j002...       74.2
```
#### Interpretation:
- The top CPU consumers are Oracle processes.
- In ps, %CPU is relative to one CPU. On a multi-CPU system, one process can show around 100% when it is consuming approximately one full CPU.

On this node:
```
CPU count: 252
vmstat idle: 90-95%
```
- So several Oracle processes using CPU does not mean the node is saturated.
#### Conclusion:
Oracle workload is actively using CPU, but the node still has large idle CPU capacity.

## Rule: Interpret Process CPU Against CPU Count

On Linux `ps`, `%CPU` is not total-node percentage. It is usually relative to one CPU.

Example:

```
100% CPU = about one full CPU
400% CPU = about four full CPUs
On a 252 CPU node, a process using 100% CPU is only about:
1 / 252 = 0.4% of total node CPU capacity
```
Always compare process CPU usage with:
```
total CPU count
vmstat idle percentage
run queue
blocked tasks
iowait
```

## Step 2: CPU Usage By OS User

Command:

```bash
ps -eo user=,pcpu= | awk '{cpu[$1]+=$2} END {for (u in cpu) printf "%-20s %.2f\n", u, cpu[u]}' | sort -k2 -nr
```

Sample output:

```text
oracle               2440.30
exawatch               95.70
root                   47.00
grid                   16.60
dbmsvc                 13.80
```

## Interpretation

In `ps`, `%CPU` is roughly relative to one CPU.

So:

```text
oracle 2440% = about 24.4 CPU cores
```

On a 252 CPU node:

```text
24.4 / 252 = about 9.7% of total CPU capacity
```

This confirms Oracle owns most of the active CPU usage, but the node is not saturated.

## Key Takeaway

CPU by user is useful for ownership:

```text
Who is consuming CPU?
```

But it must be interpreted against total CPU count and live CPU idle percentage.
