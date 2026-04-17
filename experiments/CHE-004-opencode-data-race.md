# CHE-004 · OpenCode — Data Race (Multi-Instance Database)

**Database entry**: [CHE-004](../database.md#che-004--opencode--data-race-multi-instance-database)
**GitHub issue**: https://github.com/anomalyco/opencode/issues/14194

---

## Hypothesis

When two or more OpenCode instances concurrently access the same SQLite session database — particularly when one runs inside a Docker container with the database shared via a volume mount — the application will use appropriate locking to ensure all writes are serialized safely, without corruption, constraint violations, or crashes.

## Steady State Definition

- All agent instances complete their exchanges without any SQLite error.
- The session database passes `PRAGMA integrity_check` after concurrent access.
- Neither instance crashes or logs a database error.

## Variables Introduced

| Variable | Method |
|----------|--------|
| Concurrent local-only instances | Multiple OpenCode processes launched simultaneously on the host, sharing the global SQLite database |
| Concurrent local + Docker instances | OpenCode processes on the host AND inside Docker containers, with the database directory shared via `-v` volume mount |
| Write load | Each instance creates a new session and performs tool calls (file creation + read-back) |

## Pre-requisites

- macOS host with Docker Desktop running.
- OpenCode v1.4.7 installed on the host.
- A Docker image with OpenCode installed (e.g., `opencode-test:latest` built from a Node 22 base image).
- `sqlite3` CLI for post-experiment inspection.
- A backup of `~/.local/share/opencode/opencode.db` (the experiment may permanently corrupt it).

## Experiment Steps

### 1. Establish baseline (control group — local-only concurrency)

```bash
PROJECT=/tmp/che-004-test
mkdir -p "$PROJECT" && cd "$PROJECT" && git init && echo test > test.txt && git add . && git commit -m init

# Configure OpenCode providers in $PROJECT/.opencode/opencode.json

# Record baseline
sqlite3 ~/.local/share/opencode/opencode.db "PRAGMA integrity_check;"
sqlite3 ~/.local/share/opencode/opencode.db "SELECT COUNT(*) FROM session;"

# Launch 10 concurrent local instances
for i in $(seq 1 10); do
  opencode run -m <provider>/<model> --dangerously-skip-permissions \
    "Hello from instance $i" > /tmp/oc-$i.log 2>&1 &
done
wait

# Check for errors
for i in $(seq 1 10); do
  grep -qi "sqlite\|corrupt\|malformed\|locked\|UNIQUE\|constraint" /tmp/oc-$i.log && \
    echo "Instance $i: ERROR"
done

# Verify DB integrity
sqlite3 ~/.local/share/opencode/opencode.db "PRAGMA integrity_check;"
# Result: ok — no errors with local-only concurrency.
```

### 2. Prepare Docker image

```bash
# Build a Docker image with OpenCode pre-installed
docker run --name oc-prebuilt --user root --entrypoint /bin/bash <base-image> \
  -c "npm i -g opencode-ai@latest"
docker commit oc-prebuilt opencode-test:latest
docker rm oc-prebuilt
```

### 3. Inject fault — concurrent local + Docker instances

```bash
# Back up the database first!
cp ~/.local/share/opencode/opencode.db ~/.local/share/opencode/opencode.db.bak

# Launch 5 Docker + 5 local instances simultaneously
for i in $(seq 1 5); do
  docker run --rm --user root --entrypoint /bin/bash \
    -v "$HOME/.local/share/opencode:/root/.local/share/opencode" \
    -v "$HOME/.config/opencode:/root/.config/opencode" \
    -v "/tmp/che-004-test:/workspace" \
    -w /workspace \
    opencode-test:latest \
    -c "opencode run -m <provider>/<model> --dangerously-skip-permissions \
        'Docker instance $i: create file docker_$i.txt with hello'" \
    > /tmp/oc-stress-docker-$i.log 2>&1 &

  opencode run -m <provider>/<model> --dangerously-skip-permissions \
    "Local instance $i: create file local_$i.txt with hello" \
    > /tmp/oc-stress-local-$i.log 2>&1 &
done
wait

# Check for errors
for f in /tmp/oc-stress-docker-*.log /tmp/oc-stress-local-*.log; do
  grep -qi "sqlite\|corrupt\|malformed\|NOTADB" "$f" && echo "$(basename $f): ERROR"
done

# Check DB integrity
sqlite3 ~/.local/share/opencode/opencode.db "PRAGMA integrity_check;"
```

### 4. Verify persistent corruption

```bash
# If integrity_check fails, try running any OpenCode command
opencode run -m <provider>/<model> --dangerously-skip-permissions "Hello"
# Expected: SQLiteError: database disk image is malformed

# The only recovery is restoring from backup:
cp ~/.local/share/opencode/opencode.db.bak ~/.local/share/opencode/opencode.db
rm -f ~/.local/share/opencode/opencode.db-shm ~/.local/share/opencode/opencode.db-wal
```

## Results

### Control group: local-only concurrency

| Trial | Instances | Errors | DB integrity |
|-------|-----------|--------|--------------|
| 10 concurrent local instances | 10 | 0 | ok |
| 10 concurrent local instances (write-heavy) | 10 | 0 | ok |

Local-only concurrency is safe — SQLite WAL mode handles it correctly on a native filesystem.

### Experimental group: local + Docker concurrency

| Trial | Docker instances | Local instances | Docker errors | Local errors | DB integrity |
|-------|-----------------|-----------------|---------------|--------------|--------------|
| 1 (5+5) | 5 | 5 | 0 | 5 (`SQLITE_NOTADB`) | ok (after Docker finished) |
| 2 (3+3, round 1) | 3 | 3 | 3 | 1 | **`SQLITE_CORRUPT: database disk image is malformed`** |
| 2 (3+3, round 2) | 3 | 3 | 3 | 3 | **permanently corrupted** |
| 2 (3+3, round 3) | 3 | 3 | 3 | 3 | **permanently corrupted** |

### Observed errors

1. **`SQLITE_NOTADB: file is not a database`** — Local processes fail to read the DB while Docker processes hold incompatible locks via the volume mount.
2. **`SQLITE_CORRUPT: database disk image is malformed`** — Permanent corruption of B-tree pages and index entries:
   ```
   *** in database main ***
   Tree 29 page 29: btreeInitPage() returns error code 11
   Tree 21 page 21: btreeInitPage() returns error code 11
   Tree 20 page 20: btreeInitPage() returns error code 11
   Tree 12 page 12: btreeInitPage() returns error code 11
   wrong # of entries in index session_workspace_idx
   wrong # of entries in index session_parent_idx
   wrong # of entries in index session_project_idx
   wrong # of entries in index sqlite_autoindex_session_1
   ```
3. Once corrupted, **all subsequent OpenCode operations fail** — the database is permanently unusable without restoring from a backup.

## Reproduction Environment

| Component | Version / Value |
|-----------|----------------|
| OpenCode | 1.4.7 |
| Host OS | macOS Darwin 25.4.0 (arm64) |
| Docker | 29.2.1 |
| Container base image | Node 22 (linux/arm64) |
| SQLite journal mode | WAL (default) |
| Database path | `~/.local/share/opencode/opencode.db` |

## Conclusion

Steady state was **broken**. Concurrent access to the SQLite database from local processes and Docker containers sharing the database via volume mounts causes permanent database corruption. SQLite's WAL mode file-locking protocol does not work reliably across the Docker host/container filesystem boundary — the volume mount layer does not properly mediate `fcntl` lock semantics, leading to concurrent writers corrupting B-tree pages and index structures.

Key findings:
1. **Local-only concurrency is safe**: 10+ concurrent local instances produced zero errors — SQLite WAL mode works correctly on a native filesystem.
2. **Local + Docker concurrency is destructive**: Docker volume mounts break SQLite's file-locking guarantees, causing `SQLITE_NOTADB` and `SQLITE_CORRUPT` errors.
3. **Corruption is permanent**: Once corrupted, the database cannot be used — all subsequent operations fail. The only recovery is restoring from a backup.
4. **Data loss is likely**: Session history, tool call logs, and other state stored in the database may be lost.

This matches the exact scenario reported in issue #14194: running OpenCode locally and in Docker while sharing `~/.local/share/opencode/` corrupts the database.

## Recommended Fix

1. **Document the limitation**: Warn users not to share the OpenCode data directory between host and Docker containers via volume mounts.
2. **Use a client-server architecture**: Instead of sharing the SQLite file directly, run OpenCode's database as a server process that handles all writes, with Docker containers connecting as clients (e.g., via `opencode serve` + `opencode attach`).
3. **Detect concurrent cross-boundary access**: On startup, check for an advisory lock file and warn if another instance (potentially in a container) is already accessing the database.
4. **Automatic backups**: Periodically back up the database so users can recover from corruption.
5. **Consider a more robust storage backend**: For multi-instance use cases, consider PostgreSQL or another database that properly handles concurrent access across filesystem boundaries.
