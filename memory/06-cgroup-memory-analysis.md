### Step 12: Process cgroup Membership

Command:

```
for p in 388791 359597; do echo "### PID $p"; cat /proc/$p/cgroup; done
```
Sample output summary:
```
PID 388791 java/logstash:
memory:/system.slice/logstash.service
cpu,cpuacct:/system.slice/logstash.service
name=systemd:/system.slice/logstash.service

PID 359597 ora_ipc0:
memory:/user.slice/user-1001.slice/session-302219.scope
name=systemd:/system.slice/oracle-ohasd.service
```
#### Interpretation:
The node is using cgroup v1 because /proc/<pid>/cgroup shows numbered controllers such as:
```
11:memory:
5:cpu,cpuacct:
1:name=systemd:
```
For Logstash, memory and systemd service accounting are aligned under:
```
/system.slice/logstash.service
```
For the Oracle process, systemd service identity and memory accounting differ:
```
memory controller: /user.slice/user-1001.slice/session-302219.scope
systemd identity : /system.slice/oracle-ohasd.service
```
#### Expert note:
When checking cgroup memory usage or limits, use the path shown for the memory controller, not necessarily the name=systemd path.

### Step 13: cgroup Memory Usage For A Service

Command:

```
cg=/sys/fs/cgroup/memory/system.slice/logstash.service
for f in memory.usage_in_bytes memory.max_usage_in_bytes memory.limit_in_bytes memory.failcnt memory.oom_control; do echo "### $f"; cat "$cg/$f"; done
```
Sample output:
```
memory.usage_in_bytes
10168766464

memory.max_usage_in_bytes
10183327744

memory.limit_in_bytes
9223372036854771712

memory.failcnt
0

memory.oom_control
oom_kill_disable 0
under_oom 0
oom_kill 0
```
#### Interpretation:
The Logstash cgroup is currently using about 9.47 GiB of memory, with a peak around 9.49 GiB.
The memory limit value is extremely large:
```
9223372036854771712
```
This effectively means no practical cgroup memory limit.
`memory.failcnt = 0` means the cgroup has not hit its memory limit.
`under_oom = 0` means the cgroup is not currently under OOM.
`oom_kill = 0` means no cgroup OOM kill is recorded for this cgroup.
#### Conclusion:
Logstash is not currently memory-limited by cgroups. Historical Logstash kills were global OOM victim selections, not cgroup-limit kills.

### Step 14: cgroup Memory Usage For Oracle Session Scope

Command:

```
cg=/sys/fs/cgroup/memory/user.slice/user-1001.slice/session-302219.scope
for f in memory.usage_in_bytes memory.max_usage_in_bytes memory.limit_in_bytes memory.failcnt memory.oom_control; do echo "### $f"; cat "$cg/$f"; done
```
Sample output:
```
memory.usage_in_bytes
300013641728

memory.max_usage_in_bytes
797888757760

memory.limit_in_bytes
9223372036854771712

memory.failcnt
0

memory.oom_control
oom_kill_disable 0
under_oom 0
oom_kill 0
```
#### Interpretation:
- The Oracle/session memory cgroup currently accounts for about 279 GiB.
- Its recorded peak is about 743 GiB.
- There is no practical cgroup memory limit, and no cgroup OOM is currently recorded.
#### Expert note:
`memory.max_usage_in_bytes` is useful because it preserves the peak usage since the cgroup was created. However, it does not include the timestamp of that peak.
#### Conclusion:
The Oracle/session cgroup had a much higher historical memory footprint than it has now. This may relate to the historical global OOM, but timing must be proven with time-series logs or other timestamped evidence.
