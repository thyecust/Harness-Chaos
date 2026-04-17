# Harness-Chaos

Chaos engineering for quality assurance on AI coding agent projects.

## Goal

Apply chaos engineering principles to uncover systemic weaknesses in projects such as:

- [Claude Code](https://github.com/anthropics/claude-code)
- [OpenCode](https://github.com/anomalyco/opencode)
- [Codex](https://github.com/openai/codex)
- [Copilot CLI](https://github.com/github/copilot-cli)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli)
- [Agents SDK](https://github.com/openai/openai-agents-python)
- [Pi](https://github.com/badlogic/pi-mono)

## Repository Structure

```
doc/
  principles_of_chaos_eng_en.md   # Principles, methodology, and database schema
  database.md                     # Weakness database (all findings)
  experiments/
    CHE-001-opencode-service-unavailability.md
    CHE-002-opencode-retry-storm.md
    CHE-003-opencode-memory-leak.md
    CHE-004-opencode-data-race.md
```

## Documentation

| Document | Description |
|----------|-------------|
| [Principles of Chaos Engineering](./doc/principles_of_chaos_eng_en.md) | Core concepts, experiment methodology, and weakness taxonomy |
| [Weakness Database](./doc/database.md) | All discovered weaknesses with scope, versions, and issue links |

## Weakness Classes

| Class | Description |
|-------|-------------|
| Service unavailability / cascading failure | Agent unable to continue when a dependency is unreachable |
| Retry storm | Unthrottled retries under degraded conditions amplify load |
| Memory leak | Unbounded RSS growth over long-running sessions |
| Data race | Concurrent instance writes corrupt shared state |

## Contributing

See the [Contributing section](./doc/database.md#contributing) in the database for instructions on adding new entries.