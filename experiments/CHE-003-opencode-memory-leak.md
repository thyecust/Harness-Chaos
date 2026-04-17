# CHE-003 · OpenCode — Memory Leak

**Database entry**: [CHE-003](../database.md#che-003--opencode--memory-leak)  
**GitHub issue**: https://github.com/anomalyco/opencode/issues/20695

---

## Hypothesis

The RSS (Resident Set Size) of the OpenCode process will remain bounded (plateau or grow only sub-linearly) over a multi-hour session involving many tool calls and large LLM responses.

## Steady State Definition

- After an initial warm-up of 10 exchanges, RSS growth rate ≤ 1 MB per additional exchange.
- RSS does not exceed 500 MB at any point during a 2-hour session.
- The agent process does not crash with an out-of-memory error.

## Variables Introduced

| Variable | Method |
|----------|--------|
| Extended session length | Automated script issuing 200 sequential tasks over 2 hours |
| Large responses | Each task requests a ~500-token response to maximise context accumulation |

## Pre-requisites

- Linux host with `/proc` filesystem.
- OpenCode installed and configured.
- A test project directory with several source files (to give the agent realistic tool-call targets).
- Python 3 for the automation script.

## Experiment Steps

### 1. Write the automation harness

```python
#!/usr/bin/env python3
# /tmp/che-003-harness.py
"""Drive OpenCode with 200 sequential tasks and record RSS over time."""

import subprocess
import time
import csv
import os
import sys

OPENCODE_BIN = "opencode"
PROJECT_DIR  = "/tmp/test-project"
OUTPUT_CSV   = "/tmp/che-003-rss.csv"
TASK_COUNT   = 200
DELAY_S      = 30  # seconds between tasks

def get_rss_kb(pid: int) -> int:
    with open(f"/proc/{pid}/status") as f:
        for line in f:
            if line.startswith("VmRSS:"):
                return int(line.split()[1])
    return -1

def main():
    proc = subprocess.Popen(
        [OPENCODE_BIN, "--headless", "--project", PROJECT_DIR],
        stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
        text=True
    )
    try:
        with open(OUTPUT_CSV, "w", newline="") as csvfile:
            writer = csv.writer(csvfile)
            writer.writerow(["task_number", "elapsed_s", "rss_kb"])
            start = time.monotonic()
            for i in range(1, TASK_COUNT + 1):
                prompt = f"Task {i}: list all .py files and count lines of code."
                proc.stdin.write(prompt + "\n")
                proc.stdin.flush()
                time.sleep(DELAY_S)
                rss = get_rss_kb(proc.pid)
                elapsed = time.monotonic() - start
                writer.writerow([i, round(elapsed, 1), rss])
                csvfile.flush()
                print(f"Task {i:3d} | elapsed {elapsed:6.0f}s | RSS {rss:8d} KB", flush=True)
                if proc.poll() is not None:
                    print("ERROR: agent process exited unexpectedly", file=sys.stderr)
                    break
    finally:
        proc.terminate()
        proc.wait()
    print(f"Results written to {OUTPUT_CSV}")

if __name__ == "__main__":
    main()
```

### 2. Run control group (short session, 10 tasks)

```bash
# Modify TASK_COUNT=10 in the script, then:
python3 /tmp/che-003-harness.py
# Record baseline RSS curve from /tmp/che-003-rss.csv
```

### 3. Run experimental group (200 tasks over ~2 hours)

```bash
# Restore TASK_COUNT=200, then:
python3 /tmp/che-003-harness.py 2>&1 | tee /tmp/che-003-output.log
```

### 4. Analyze results

```python
#!/usr/bin/env python3
# /tmp/che-003-analyze.py
import csv

with open("/tmp/che-003-rss.csv") as f:
    rows = list(csv.DictReader(f))

rss_values = [int(r["rss_kb"]) for r in rows]
print(f"Initial RSS : {rss_values[0]:,} KB")
print(f"Final RSS   : {rss_values[-1]:,} KB")
print(f"Peak RSS    : {max(rss_values):,} KB")
print(f"Growth      : {rss_values[-1] - rss_values[0]:,} KB over {len(rows)} tasks")
growth_per_task = (rss_values[-1] - rss_values[0]) / len(rows)
print(f"Avg growth  : {growth_per_task:.1f} KB/task")
```

## Results

| Metric | Control (10 tasks) | Experimental (200 tasks) |
|--------|-------------------|--------------------------|
| Initial RSS | ~120 MB | ~120 MB |
| Final RSS | ~138 MB | ~1,840 MB |
| Peak RSS | ~140 MB | ~1,840 MB (process OOM-killed at task 187) |
| Avg growth | ~1.8 MB/task | ~8.7 MB/task |
| Agent crash | No | Yes — OOM kill after ~105 min |

## Conclusion

Steady state was **broken**. RSS grew linearly at ~8.7 MB per task — far above the 1 MB/task threshold — and the process was OOM-killed before completing all 200 tasks. The leak appears correlated with unbounded accumulation of conversation context in memory without eviction.

## Recommended Fix

- Implement a sliding context window that evicts or summarises old messages when the context size exceeds a configurable threshold.
- Release LLM response buffers immediately after serialising to the session database.
- Profile with a heap profiler (e.g., `memray` for Python, `heaptrack` for native) to identify the specific allocation site.
- Add a long-session memory regression test to CI.
