---
name: hpc-socket-pinning
description: Pin heavy jobs to one CPU socket on the 2-socket Xeon HPC node (taskset + numactl --localalloc). Use whenever launching any heavy, long-running or multi-threaded job on this node (alignment, R/Python pipelines, MATseq runs), and for verifying CPU/NUMA placement of a running process.
license: MIT
metadata:
    skill-author: TessAfanasyeva
---

# HPC socket pinning

Any time you launch a heavy job on this node, pin it to one socket and allocate memory locally.

## Hardware

- CPUs: 2 x Intel Xeon Platinum 8268
- Physical cores: 48 (24 per socket); logical CPUs: 96; NUMA nodes: 2
- RAM: ~376 GiB

## Verified topology

| Set | CPU list |
|---|---|
| Socket 0 | `0-46:2` |
| Socket 1 | `1-47:2` |
| Both sockets | `0-47` |

Hyperthread siblings: CPU N and N+48.

## Default launch

```bash
nohup nice -n 5 taskset -c "$CPU_SET" \
  numactl --localalloc \
  <program> <arguments> > job.log 2>&1 &
```

## Rules

Use socket 0 by default. Use `1-47:2` if socket 0 is busy, and `0-47` only for jobs that need both sockets.
Set thread-count variables (e.g. `OMP_NUM_THREADS`, `--threads`, `mc.cores`) to match the number of CPUs in the chosen set (24 per socket).

## Monitoring

Confirm the bind worked and the job is running on the intended cores.

- `numastat -p` takes one PID or one name pattern. Do not pass a comma- or space-separated PID list; it is ignored and node-wide counters are printed instead.
- For multi-process programs `taskset -apc <PID>` and `numastat -p <PID>` cover only that PID. Check workers via the name pattern, or loop over `pgrep -P <PID>`.
- `pgrep -f <program>` also matches the launching shell, which is outside the pinned set. Capture `P=$!` on the line right after the launch and use it for all checks. Wrappers such as `poetry run` exec in place, so `$!` is the program's own PID and `pgrep -P $!` finds no child. Run `kill -0 $P` first to confirm the job is still alive.

```bash
taskset -apc <PID>                  # affinity of all threads
numastat -p <PID>                   # memory should sit on the matching NUMA node
numastat -p <script-or-program>     # multi-process jobs: pattern matches the command line; one row per process + Total
ps -L -o tid,psr,pcpu,ni -p <PID>   # per-thread CPU (psr) must be in the set; ni = nice value
pidstat -u -t -p <PID> 5            # per-thread %CPU and CPU column over time
```
