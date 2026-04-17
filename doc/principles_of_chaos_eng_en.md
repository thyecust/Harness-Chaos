# Principles of Chaos Engineering

> Adapted from [principlesofchaos.org](https://principlesofchaos.org/)

Chaos Engineering is the discipline of experimenting on a system in order to build confidence in the system's capability to withstand turbulent conditions in production.

## Core Concepts

### Steady State

A *steady state* is some measurable output of a system that indicates normal behavior. The throughput, error rates, and latency percentiles could all be metrics of interest representing steady state. By focusing on systemic behavior patterns during experiments, Chaos validates that the system *does* work, rather than trying to validate *how* it works.

### Principles

**1. Build a Hypothesis Around Steady State Behavior**

Focus on the measurable output of a system, rather than internal attributes of the system. Measurements of that output over a short period of time constitute a proxy for the system's steady state. The overall system's throughput, error rates, latency percentiles, etc. could all be metrics of interest representing steady state behavior. By focusing on systemic behavior patterns during experiments, Chaos Engineering validates that the system works, rather than trying to validate how it works.

**2. Vary Real-World Events**

Chaos variables reflect real-world events. Prioritize events either by potential impact or estimated frequency. Consider events that correspond to hardware failures like servers dying, software failures like malformed responses, and non-failure events like a spike in traffic or a scaling event. Any event capable of disrupting steady state is a potential variable in a Chaos experiment.

**3. Run Experiments in Production**

Systems behave differently depending on environment and traffic patterns. Since the behavior of utilization can only be reliably sampled from actual traffic, Chaos strongly prefers to experiment directly on production traffic. However, for AI coding agents, experiments may be run in staging environments that closely mirror production configurations.

**4. Automate Experiments to Run Continuously**

Running experiments manually is labor-intensive and ultimately unsustainable. Automate experiments and run them continuously. Chaos Engineering builds automation into the system to drive both orchestration and analysis.

**5. Minimize Blast Radius**

Experimenting in production has the potential to cause unnecessary customer pain. While there must be an allowance for some short-term negative impact, it is the responsibility and obligation of the Chaos Engineer to ensure the fallout from experiments is minimized and contained.

## Weaknesses to Uncover

The following classes of systemic weaknesses are of primary interest for AI coding agent projects:

| # | Weakness | Description | Example Manifestation |
|---|----------|-------------|----------------------|
| 1 | **Service unavailability / cascading failure** | Lack of proper fallback when a dependent service (e.g., LLM API) is unavailable; single points of failure causing the agent to be unable to continue working | Agent hangs or crashes when API endpoint is unreachable |
| 2 | **Retry storms** | Improper timeout settings causing repeated, rapid retries that amplify load during degraded conditions | `SSE read timed out` errors; exponential request surge |
| 3 | **Memory leaks** | Long-running agent sessions accumulating memory without releasing it, eventually causing OOM crashes | Resident set size grows unbounded over a multi-hour session |
| 4 | **Data races** | Concurrent access to shared state (e.g., SQLite databases) by multiple agent instances causing corruption or unexpected behavior | Duplicate entries, lost updates, or constraint violations |

## Experiment Methodology

### Steps

1. **Define steady state** — identify measurable outputs (response latency, error rate, task completion rate, memory usage) that characterize normal operation.
2. **Form a hypothesis** — state what you expect to remain true under a specific turbulent condition (e.g., "the agent will gracefully retry after a transient 503").
3. **Design the experiment** — choose one real-world variable to inject (network partition, slow disk, API rate-limit response, process kill, etc.).
4. **Run control and experimental groups** — establish a baseline with no fault injection, then re-run with the fault injected.
5. **Measure and compare** — collect metrics from both groups and compare them against the steady-state definition.
6. **Confirm or refute the hypothesis** — document whether the system maintained steady state; if not, record the failure mode.
7. **Fix and retest** — file a GitHub issue, develop a fix, and re-run the experiment to confirm remediation.

### Tooling

- **Fault injection**: `tc netem` (network latency/loss), `iptables` (connectivity drops), `stress-ng` (CPU/memory pressure), mock servers returning error codes.
- **Metrics collection**: process memory (`/proc/<pid>/status`), HTTP response codes and timings, log analysis.
- **Automation**: shell scripts or Python harnesses driving the agent under test, with assertions against expected steady-state metrics.

## Database Schema

Each weakness entry in the database records:

| Field | Description |
|-------|-------------|
| `id` | Unique identifier |
| `project` | Affected project and repository URL |
| `weakness_class` | One of the four weakness classes above |
| `manifestation` | Observable symptom |
| `impact_scope` | Who is affected and how severely |
| `dependency_versions` | Versions of relevant dependencies at time of discovery |
| `experiment_steps` | Step-by-step reproduction procedure |
| `experiment_result` | Outcome (steady state broken / maintained) |
| `github_issue` | Link to the issue |
| `fix_mr` | Link to the fix PR/MR (if available) |
| `status` | `open` / `fixed` / `wont_fix` |
