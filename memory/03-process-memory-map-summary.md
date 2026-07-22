### Step 7: Process Memory Map Summary

Command:

```
cat /proc/359597/smaps_rollup
```
#### Important output:
```
Rss:             6400008 kB
Pss:             4577238 kB
Pss_Anon:          30956 kB
Pss_File:            241 kB
Pss_Shmem:       4546041 kB
Shared_Hugetlb: 565311488 kB
Private_Hugetlb: 44216320 kB
Swap:                  0 kB
```
#### Interpretation:
- smaps_rollup gives a summarized memory-map view for one process.
#### Important terms:
- `RSS`
Resident memory mapped by the process.
- `PSS`
Proportional memory attributed to the process. If memory is shared, each process gets only its fair share.
- `Pss_Anon`
Proportional anonymous/private memory.
- `Pss_File`
Proportional file-backed memory.
- `Pss_Shmem`
Proportional shared memory.
- `Shared_Hugetlb`
HugePages-backed memory shared with other processes.
- `Private_Hugetlb`
HugePages-backed memory private to this process mapping.
- `Swap`
Memory from this process currently swapped out.
#### Conclusion:
This Oracle process is not a private memory-heavy process. It is attached to a very large HugePages-backed database memory area. Swap is zero for this process.

# Using IPCS - Another example

## Step: Confirm SYSV Shared Memory With `ipcs`

When a process shows high `RssShmem` or large `SYSV` mappings in `pmap` or `/proc/<pid>/maps`, use `ipcs` to confirm which shared memory segments it is attached to.

### Commands

```bash
pid=<PID>
pmap -x $pid | grep -E 'anon|zero|SYSV'
grep -E 'SYSV|zero' /proc/$pid/maps | head -50
ipcs -m
ipcs -m -p
ipcs -m -t
```

### What To Look For

From `/proc/<pid>/maps` or `pmap`, identify large `SYSV` segments and note their sizes.

Example:

```text
SYSV00000000   32768 KB
SYSV00000000   613089280 KB
SYSV00000000   1114112 KB
SYSV01918328      2048 KB
```

Then compare with `ipcs -m` shared memory segment sizes.

Example:

```text
65579  oracle   33554432
65580  oracle   627803422720
65581  oracle   1140850688
65582  oracle   2097152
```

Convert `ipcs` bytes to KB and match them to the `pmap` or `/proc/<pid>/maps` segment sizes.

Example:

```text
33554432 bytes      = 32768 KB
627803422720 bytes  = 613089280 KB
1140850688 bytes    = 1114112 KB
2097152 bytes       = 2048 KB
```

This confirms that the process is attached to those shared memory segments.

### How To Interpret `ipcs`

`ipcs -m`

Shows shared memory segments, owner, size, and attach count.

Important fields:

```text
bytes
  size of the shared memory segment

owner
  user that owns the segment

nattch
  number of attached processes
```

A very large segment with high `nattch` usually indicates shared database memory, not private process usage.

Example:

```text
shmid 65580
owner oracle
bytes 627803422720
nattch 4253
```

Interpretation:

```text
This is a huge Oracle shared memory segment.
It is attached by thousands of processes.
It should not be treated as private memory of one process.
```

`ipcs -m -p`

Shows creator PID and last-operation PID.

Useful for identifying which process created or recently used the segment.

`ipcs -m -t`

Shows attach, detach, and change times.

Useful for relating shared memory segment activity to instance startup or incident windows.

### Important Note About `(deleted)`

A mapping like this is normal:

```text
/SYSV00000000 (deleted)
```

For SYSV shared memory, `(deleted)` does not mean the memory is gone.

It means the segment was marked for removal from the namespace, but it remains alive while processes are still attached.

### Key Takeaway

Use `ipcs` when high process RSS is suspected to be shared memory.

If `RssShmem` is high and `ipcs` shows large Oracle SYSV segments with high `nattch`, the process is mapping shared database memory rather than privately consuming that full amount.

## Live Case Example: Oracle RSS Explained By SYSV Shared Memory

A process was flagged because its RSS crossed the large-process threshold.

### Process View

```text
PID        : 163636
Process    : ora_ipc0_e2v1pod4
VmRSS      : 57.44 GiB
RssAnon    : 0.26 GiB
RssFile    : 0.17 GiB
RssShmem   : 57.00 GiB
VmSwap     : 0.00 GiB
```

Interpretation:

```text
Almost all resident memory mapped by the process is shared memory.
Private anonymous memory is very small.
This is not a private process heap growth pattern.
```

### `smaps_rollup` View

```text
Rss              57.44 GiB
Pss              34.50 GiB
Pss_Anon          0.26 GiB
Pss_Shmem        34.24 GiB
Shared_Hugetlb  578.68 GiB
Private_Hugetlb   7.09 GiB
Swap              0.00 GiB
```

Interpretation:

```text
RSS shows what the process maps in RAM.
PSS shows the fair-share attribution after dividing shared memory.
The gap between RSS and PSS confirms that much of the memory is shared.
```

### SYSV Mapping Evidence

From `/proc/<pid>/maps` and `pmap`:

```text
SYSV00000000   32768 KB
SYSV00000000   613089280 KB
SYSV00000000   1114112 KB
SYSV01918328      2048 KB
```

### `ipcs` Correlation

```text
shmid 65579  oracle   33554432 bytes
shmid 65580  oracle   627803422720 bytes
shmid 65581  oracle   1140850688 bytes
shmid 65582  oracle   2097152 bytes
```

Converted sizes:

```text
33554432 bytes      = 32768 KB
627803422720 bytes  = 613089280 KB
1140850688 bytes    = 1114112 KB
2097152 bytes       = 2048 KB
```

These sizes match the SYSV mappings exactly.

### Shared Memory Strength Of Evidence

`ipcs -m` also showed:

```text
shmid 65580
owner  oracle
nattch 4253
```

Interpretation:

```text
The large shared memory segment is attached by thousands of processes.
This is shared Oracle database memory, not private memory owned by one process.
```

### Node Memory Context

```text
MemAvailable   474.60 GiB
SwapFree        16.56 GiB out of 16.68 GiB
HugePages_Used 585.78 GiB
Hugetlb        586.01 GiB
```

Interpretation:

```text
The node still has plenty of available memory.
Swap pressure is negligible.
HugePages usage closely matches the large SYSV shared segment.
```

### Final Conclusion

- This process crossed the high-RSS threshold because it maps large Oracle SYSV shared memory backed by HugePages.
- It is not privately consuming 57.44 GiB of process heap memory.
- This is a shared-memory database footprint, not a private memory leak.
