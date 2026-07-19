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
