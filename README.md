# Harness Chaos

Our goal is to apply chaos engineering for quality assurance on current Harness projects, such as

- [Claude Code](https://github.com/anthropics/claude-code)
- [OpenCode](https://github.com/anomalyco/opencode)
- [Codex](https://github.com/openai/codex)
- [Copilot CLI](https://github.com/github/copilot-cli)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli)
- [Agents SDK](https://github.com/openai/openai-agents-python)
- [Pi](https://github.com/badlogic/pi-mono)

## Principles of Chaos Engineering

According to the [Principles of Chaos Engineering](./doc/principles_of_chaos_eng_en.md), our goal is to uncover systemic weaknesses, such as

1. Lack of proper fallback when a service is unavailable / cascading failures from single points of failure, which may manifest as the Agent being unable to continue working, e.g. https://github.com/anomalyco/opencode/issues/4781
2. Retry storms caused by improper timeout settings, which may manifest as `SSE read timed out`, e.g. https://github.com/anomalyco/opencode/issues/17307
3. Memory leaks, e.g. https://github.com/anomalyco/opencode/issues/20695
4. Data races between databases of multiple instances, e.g. https://github.com/anomalyco/opencode/issues/14194

To uncover weaknesses, we conduct experiments as follows:

1. Define a "steady state" based on the system's output under normal behavior
2. Ensure that this steady state is maintained in both the control group and the experimental group
3. Introduce variables in the experimental group that reflect real-world events, such as server crashes, hard drive corruption, network disconnections, etc.
4. Demonstrate that the experimental group is no longer in a "steady state" by comparing the output differences between the control group and the experimental group

## Chaos Engineering in Practice

Our goal is to maintain a database that includes:

- Manifestations of weaknesses
- Scope of impact of weaknesses
- Versions of related dependencies
- Complete experiment steps and results
- GitHub Issues and fix MRs

## Repository Structure

```
database.md                       # Weakness database (all findings)
experiments/
  CHE-001-opencode-service-unavailability.md
  CHE-002-opencode-retry-storm.md
  CHE-003-opencode-memory-leak.md
  CHE-004-opencode-data-race.md
doc/
  principles_of_chaos_eng_en.md   # Principles of Chaos Engineering (English)
  principles_of_chaos_eng_zh.md   # Principles of Chaos Engineering (Chinese)
README_zh.md                      # Chinese README
```

## Documentation

| Document | Description |
|----------|-------------|
| [Principles of Chaos Engineering (EN)](./doc/principles_of_chaos_eng_en.md) | Core principles and methodology |
| [Principles of Chaos Engineering (ZH)](./doc/principles_of_chaos_eng_zh.md) | 混沌工程原则（中文） |
| [Weakness Database](./database.md) | All discovered weaknesses with scope, versions, and issue links |

## Contributing

See the [Contributing section](./database.md#contributing) in the database for instructions on adding new entries.
