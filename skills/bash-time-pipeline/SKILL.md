---
name: bash_time_pipeline
description: How to measure execution time of a pipeline in bash using the time keyword, TIMEFORMAT, capturing timing output, and EPOCHREALTIME; includes keyword-vs-/usr/bin/time gotcha
tags:
- bash
- shell
- time
- pipeline
- profiling
- stderr
created_at: '2026-09-30T13:43:27.034760626+02:00'
---

# Timing a pipeline in bash

## The `time` keyword

In bash, `time` is a **reserved word** that times the entire pipeline that follows it:

```bash
time cmd1 | cmd2          # times the WHOLE pipeline
```

Output (real/user/sys) goes to **stderr**.

## Gotchas

1. **Keyword vs external binary.** In POSIX `sh` (dash) there is no `time` keyword; `/usr/bin/time cmd1 | cmd2` would time only `cmd1`. To use `/usr/bin/time` (extra stats like max RSS via `-v`) on a pipe, wrap it:
   ```bash
   /usr/bin/time -v bash -c 'cmd1 | cmd2'
   ```
2. **Custom format** with `TIMEFORMAT` (bash-only):
   ```bash
   TIMEFORMAT='%3R'        # just real seconds, e.g. 2.004
   TIMEFORMAT='real %3R user %3U sys %3S'
   ```
   `time -p` gives POSIX format.
3. **Capture only the timing** (time writes to stderr, mixed with pipeline stderr):
   ```bash
   TIMEFORMAT='%3R'
   t=$( { time cmd1 | cmd2 >/dev/null 2>&1; } 2>&1 )
   ```
4. **Time only one stage** of the pipe by grouping:
   ```bash
   cmd1 | { time cmd2; } 2> stage2.time
   ```

## Programmatic alternative

`$EPOCHREALTIME` (bash >= 5, microsecond precision, no subprocess needed):

```bash
start=$EPOCHREALTIME
cmd1 | cmd2
end=$EPOCHREALTIME
awk -v s="$start" -v e="$end" 'BEGIN{printf "%.3fs\n", e-s}'
```

(`$SECONDS` is integer-seconds only; bash has no float arithmetic, hence awk/bc.)
