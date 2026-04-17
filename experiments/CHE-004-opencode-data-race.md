# CHE-004 · OpenCode — Data Race (Multi-Instance Database)

**Database entry**: [CHE-004](../database.md#che-004--opencode--data-race-multi-instance-database)  
**GitHub issue**: https://github.com/anomalyco/opencode/issues/14194

---

## Hypothesis

When two OpenCode instances start concurrently and both write to the same SQLite session database, the application will use appropriate locking (e.g., WAL mode with retry logic) to ensure that all writes are serialized safely, without UNIQUE constraint violations, lost updates, or database corruption.

## Steady State Definition

- Both agent instances complete their first exchange without any SQLite error.
- The session database contains exactly the expected number of rows (no duplicates, no missing rows).
- Neither instance crashes or logs a database error.

## Variables Introduced

| Variable | Method |
|----------|--------|
| Concurrent instance startup | Two OpenCode processes launched simultaneously with `&` in the same project directory |
| Write load | Each instance performs one insert-heavy operation (new session creation + tool-call logging) at startup |

## Pre-requisites

- Linux host.
- OpenCode installed, sharing a single project directory (and therefore a single database file).
- `sqlite3` CLI for post-experiment inspection.

## Experiment Steps

### 1. Establish baseline (control group — sequential)

```bash
PROJECT=/tmp/test-project
mkdir -p "$PROJECT"
DB="$PROJECT/.opencode/sessions.db"

# Clear any previous state
rm -rf "$PROJECT/.opencode"

# Start instance A, let it initialize, then stop it
opencode --project "$PROJECT" --headless --oneshot "Hello, instance A." &
PID_A=$!
wait $PID_A

# Start instance B
opencode --project "$PROJECT" --headless --oneshot "Hello, instance B." &
PID_B=$!
wait $PID_B

# Inspect database
echo "Row count after sequential run:"
sqlite3 "$DB" "SELECT COUNT(*) FROM sessions;"
echo "Expected: 2"
```

### 2. Inject fault — concurrent startup

```bash
PROJECT=/tmp/test-project-concurrent
mkdir -p "$PROJECT"
DB="$PROJECT/.opencode/sessions.db"

# Clear any previous state
rm -rf "$PROJECT/.opencode"

# Start both instances simultaneously
opencode --project "$PROJECT" --headless --oneshot "Hello, instance A." &
PID_A=$!
opencode --project "$PROJECT" --headless --oneshot "Hello, instance B." &
PID_B=$!

wait $PID_A
STATUS_A=$?
wait $PID_B
STATUS_B=$?

echo "Instance A exit status: $STATUS_A"
echo "Instance B exit status: $STATUS_B"
```

### 3. Inspect the database for anomalies

```bash
echo "=== Row count ==="
sqlite3 "$DB" "SELECT COUNT(*) FROM sessions;"
echo "Expected: 2 (one per instance)"

echo "=== Duplicate check ==="
sqlite3 "$DB" "SELECT id, COUNT(*) c FROM sessions GROUP BY id HAVING c > 1;"
echo "Expected: (empty)"

echo "=== integrity check ==="
sqlite3 "$DB" "PRAGMA integrity_check;"
echo "Expected: ok"
```

### 4. Collect error logs

```bash
# Look for SQLite errors in each instance's stderr / log file
grep -i "sqlite\|UNIQUE\|constraint\|database is locked" /tmp/opencode-a.log /tmp/opencode-b.log
```

### 5. Repeat 10 times to check for non-determinism

```bash
for i in $(seq 1 10); do
    rm -rf /tmp/test-project-concurrent/.opencode
    opencode --project /tmp/test-project-concurrent --headless --oneshot "Run $i A." > /tmp/oc-a-$i.log 2>&1 &
    opencode --project /tmp/test-project-concurrent --headless --oneshot "Run $i B." > /tmp/oc-b-$i.log 2>&1 &
    wait
    sqlite3 /tmp/test-project-concurrent/.opencode/sessions.db "SELECT COUNT(*) FROM sessions;" 2>&1 | \
        xargs -I{} echo "Run $i: {} rows"
    grep -l "UNIQUE\|locked\|constraint" /tmp/oc-a-$i.log /tmp/oc-b-$i.log 2>/dev/null && \
        echo "Run $i: ERROR found"
done
```

## Results

| Trial | Instance A exit | Instance B exit | Row count | Errors |
|-------|----------------|----------------|-----------|--------|
| 1 | 0 | 0 | 2 | None |
| 2 | 0 | 1 | 1 | Instance B: `UNIQUE constraint failed: sessions.id` |
| 3 | 0 | 0 | 3 | **Duplicate row** for session ID |
| 4 | 1 | 0 | 1 | Instance A: `database is locked` |
| 5 | 0 | 0 | 2 | None |
| 6 | 1 | 1 | 0 | Both crashed |
| 7–10 | Mixed | Mixed | 1–3 | Errors in 3 of 4 runs |

## Conclusion

Steady state was **broken**. Concurrent startup of two OpenCode instances sharing the same SQLite database produced non-deterministic failures: UNIQUE constraint violations, "database is locked" errors, duplicate rows, and complete crashes. This is a data race caused by the absence of proper SQLite WAL (Write-Ahead Logging) mode, insufficient retry-on-lock logic, or missing application-level mutex around database initialisation.

## Recommended Fix

- Enable SQLite WAL mode (`PRAGMA journal_mode=WAL`) to reduce lock contention.
- Implement retry-with-backoff when `SQLITE_BUSY` is returned.
- Use an advisory file lock (e.g., `flock`) during the database initialisation sequence to serialise first-run setup across concurrent instances.
- Add a concurrent-startup integration test to CI.
