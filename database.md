# Chaos Engineering Weakness Database

This database records systemic weaknesses discovered through chaos engineering experiments on AI coding agent projects. Each entry follows the schema defined in [Principles of Chaos Engineering](./doc/principles_of_chaos_eng_en.md#database-schema).

---

## Entries

### CHE-001 · OpenCode — Session Poisoning from Invalid Image Data

| Field | Value |
|-------|-------|
| **ID** | CHE-001 |
| **Project** | [OpenCode](https://github.com/anomalyco/opencode) |
| **Weakness class** | Session poisoning from invalid image data |
| **Manifestation** | Oversized or invalid image data (> 5 MB or > 2000 px) persisted as base64 in SQLite session history causes every subsequent API call to fail permanently; the session is unrecoverable without manual database surgery or starting a new session |
| **Impact scope** | All users who encounter oversized/invalid images via the read tool, clipboard paste, or MCP tools (e.g., Chrome screenshots); session is permanently poisoned with no built-in recovery path |
| **Dependency versions** | opencode v1.4.7 |
| **Experiment steps** | See [experiments/CHE-001-opencode-service-unavailability.md](./experiments/CHE-001-opencode-service-unavailability.md) |
| **Experiment result** | Steady state **broken** — 19 MB JPEG stored as ~25 MB base64 in session; Claude Opus 4.6 returns `SupplierResponseFailedError: image exceeds 5 MB maximum` on every subsequent request; session permanently unrecoverable |
| **GitHub issue** | https://github.com/anomalyco/opencode/issues/4781 |
| **Fix MR** | — |
| **Status** | `open` — **reproduced** |

---

### CHE-002 · OpenCode — Retry Storm / Improper Timeout

| Field | Value |
|-------|-------|
| **ID** | CHE-002 |
| **Project** | [OpenCode](https://github.com/anomalyco/opencode) |
| **Weakness class** | Retry storms caused by improper timeout settings |
| **Manifestation** | `SSE read timed out` errors trigger rapid, unthrottled retries; under a slow or intermittent connection the agent amplifies load on the LLM provider and exhausts local resources |
| **Impact scope** | Users on high-latency connections; LLM provider endpoints during partial outages |
| **Dependency versions** | opencode ≤ commit at time of issue filing |
| **Experiment steps** | See [experiments/CHE-002-opencode-retry-storm.md](./experiments/CHE-002-opencode-retry-storm.md) |
| **Experiment result** | Steady state **broken** — repeated `SSE read timed out` with no backoff |
| **GitHub issue** | https://github.com/anomalyco/opencode/issues/17307 |
| **Fix MR** | — |
| **Status** | `open` — **not reproduced** |

---

### CHE-003 · OpenCode — Memory Leak

| Field | Value |
|-------|-------|
| **ID** | CHE-003 |
| **Project** | [OpenCode](https://github.com/anomalyco/opencode) |
| **Weakness class** | Memory leak |
| **Manifestation** | Resident set size (RSS) grows monotonically over a long-running session; eventually causes OOM termination of the agent process |
| **Impact scope** | Users running extended sessions (e.g., large refactoring tasks); severity increases proportionally with conversation length |
| **Dependency versions** | opencode ≤ commit at time of issue filing |
| **Experiment steps** | See [experiments/CHE-003-opencode-memory-leak.md](./experiments/CHE-003-opencode-memory-leak.md) |
| **Experiment result** | Steady state **broken** — memory usage did not plateau; agent crashed after ~2 hours |
| **GitHub issue** | https://github.com/anomalyco/opencode/issues/20695 |
| **Fix MR** | — |
| **Status** | `open` — **not reproduced** |

---

### CHE-004 · OpenCode — Data Race (Multi-Instance Database)

| Field | Value |
|-------|-------|
| **ID** | CHE-004 |
| **Project** | [OpenCode](https://github.com/anomalyco/opencode) |
| **Weakness class** | Data race between databases of multiple instances |
| **Manifestation** | When two OpenCode instances share the same SQLite database (e.g., in a mono-repo with multiple terminal windows), concurrent writes cause duplicate entries, lost updates, or UNIQUE constraint violations |
| **Impact scope** | Power users running multiple agent sessions concurrently in the same project; data integrity of the session history database |
| **Dependency versions** | opencode ≤ commit at time of issue filing |
| **Experiment steps** | See [experiments/CHE-004-opencode-data-race.md](./experiments/CHE-004-opencode-data-race.md) |
| **Experiment result** | Steady state **broken** — constraint violation errors observed within seconds of concurrent startup |
| **GitHub issue** | https://github.com/anomalyco/opencode/issues/14194 |
| **Fix MR** | — |
| **Status** | `open` — **not reproduced** |

---

## Summary

| ID | Project | Weakness Class | Reproduction |
|----|---------|----------------|--------------|
| CHE-001 | OpenCode | Session poisoning from invalid image data | **reproduced** |
| CHE-002 | OpenCode | Retry storm / improper timeout | not reproduced |
| CHE-003 | OpenCode | Memory leak | not reproduced |
| CHE-004 | OpenCode | Data race (multi-instance DB) | not reproduced |

---

## Contributing

To add a new entry:

1. Copy an existing entry block.
2. Assign the next sequential `CHE-NNN` ID.
3. Fill in all fields; link to a new experiment file under `experiments/`.
4. Add the entry to the Summary table.
5. Open a pull request.
