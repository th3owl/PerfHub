# Thread-Level CPU Analysis

## Step 1: Inspect Threads For A Hot Process

Command:

```bash
pid=356924
ps -L -p $pid -o pid,tid,psr,stat,pcpu,comm,args --sort=-pcpu | head -30
```

Sample output:

```text
PID     TID     PSR STAT %CPU COMMAND
356924  367351    8 Ssl  10.8 ora_of02_e6t1po
356924  367747  181 Ssl  18.5 ora_of0e_e6t1po
356924  367818  152 Ssl  18.5 ora_of0q_e6t1po
356924  367238  180 Ssl   1.5 ora_of00_e6t1po
356924  367289  174 Ssl   1.5 ora_of01_e6t1po
```

## Interpretation

The process-level CPU was around:

```text
123%
```

Thread-level output shows the CPU is spread across many threads.

This is not a single runaway thread. It is multi-threaded Oracle worker activity.

## Important Columns

`PID`

Process ID.

`TID`

Thread ID. On Linux, each thread also has a task ID.

`PSR`

Processor/core the thread last ran on.

`STAT`

Thread state.

`%CPU`

CPU used by that thread.

`COMMAND`

Thread/process name.

## Key Takeaway

If a process shows high CPU, inspect its threads.

A process can show more than 100% CPU when multiple threads consume CPU. Thread-level analysis shows whether the CPU comes from one runaway thread or distributed worker activity.

Note: 
To print sorted data:
```
pid=356924
ps -L -p $pid -o pid,tid,psr,stat,pcpu,comm,args --no-headers | sort -k5 -nr | head -30
```
Sample Output:
```356924 367940 167 Ssl  18.4 ora_of12_e6t1po ora_ofsd_e6t1pod3
356924 367818 205 Ssl  18.4 ora_of0q_e6t1po ora_ofsd_e6t1pod3
356924 367747 186 Ssl  18.4 ora_of0e_e6t1po ora_ofsd_e6t1pod3
356924 367351  92 Ssl  10.8 ora_of02_e6t1po ora_ofsd_e6t1pod3
356924 367409 196 Ssl   1.6 ora_of03_e6t1po ora_ofsd_e6t1pod3
356924 367943 228 Ssl   1.5 ora_of13_e6t1po ora_ofsd_e6t1pod3
356924 367934 185 Ssl   1.5 ora_of11_e6t1po ora_ofsd_e6t1pod3
356924 367929 216 Ssl   1.5 ora_of10_e6t1po ora_ofsd_e6t1pod3
356924 367922  60 Ssl   1.5 ora_of0z_e6t1po ora_ofsd_e6t1pod3
356924 367917  36 Ssl   1.5 ora_of0y_e6t1po ora_ofsd_e6t1pod3
356924 367910 114 Ssl   1.5 ora_of0x_e6t1po ora_ofsd_e6t1pod3
356924 367905 191 Ssl   1.5 ora_of0w_e6t1po ora_ofsd_e6t1pod3
356924 367899 248 Ssl   1.5 ora_of0v_e6t1po ora_ofsd_e6t1pod3
356924 367888 205 Ssl   1.5 ora_of0u_e6t1po ora_ofsd_e6t1pod3
356924 367882 160 Ssl   1.5 ora_of0t_e6t1po ora_ofsd_e6t1pod3
```
