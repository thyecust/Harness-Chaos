# Chaos Engineering Weakness Database

This database records systemic weaknesses discovered through chaos engineering experiments on AI coding agent projects. Each entry follows the schema defined in [Principles of Chaos Engineering](./principles_of_chaos_eng_en.md#database-schema).

---

## Entries

### CHE-001 · OpenCode — Service Unavailability / Cascading Failure

| Field | Value |
|-------|-------|
| **ID** | CHE-001 |
| **Project** | [OpenCode](https://github.com/anomalyco/opencode) |
| **Weakness class** | Service unavailability / cascading failure |
| **Manifestation** | Agent is unable to continue working when the LLM API endpoint becomes unreachable; session hangs indefinitely with no user-visible error or recovery path |
| **Impact scope** | All users on any network with intermittent or degraded connectivity to the LLM provider; complete loss of agent functionality for the duration of the outage |
| **Dependency versions** | opencode ≤ commit at time of issue filing |
| **Experiment steps** | See [experiments/CHE-001-opencode-service-unavailability.md](./experiments/CHE-001-opencode-service-unavailability.md) |
| **Experiment result** | Steady state **broken** — agent session hung without recovery |
| **GitHub issue** | https://github.com/anomalyco/opencode/issues/4781 |
| **Fix MR** | — |
| **Status** | `open` |

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
| **Status** | `open` |

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
| **Status** | `open` |

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
| **Status** | `open` |

---

## Summary

| ID | Project | Weakness Class | Status |
|----|---------|----------------|--------|
| CHE-001 | OpenCode | Service unavailability / cascading failure | open |
| CHE-002 | OpenCode | Retry storm / improper timeout | open |
| CHE-003 | OpenCode | Memory leak | open |
| CHE-004 | OpenCode | Data race (multi-instance DB) | open |

---

## Contributing

To add a new entry:

1. Copy an existing entry block.
2. Assign the next sequential `CHE-NNN` ID.
3. Fill in all fields; link to a new experiment file under `doc/experiments/`.
4. Add the entry to the Summary table.
5. Open a pull request.
