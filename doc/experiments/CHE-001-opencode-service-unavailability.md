# CHE-001 · OpenCode — Service Unavailability / Cascading Failure

**Database entry**: [CHE-001](../database.md#che-001--opencode--service-unavailability--cascading-failure)  
**GitHub issue**: https://github.com/anomalyco/opencode/issues/4781

---

## Hypothesis

If the LLM API endpoint becomes unreachable while a task is in progress, OpenCode will surface a clear error to the user and offer a recovery path (retry, offline mode, or graceful exit), thereby maintaining a usable steady state.

## Steady State Definition

- The agent completes tool-call cycles and returns results to the user within a reasonable timeout.
- Any API error is surfaced as a human-readable message within 30 seconds.
- The agent process remains responsive to keyboard input (Ctrl-C, etc.).

## Variables Introduced

| Variable | Method |
|----------|--------|
| LLM API unreachable | Block outbound HTTPS to the provider domain with `iptables` |
| Duration | 60 seconds (long enough to trigger a timeout) |

## Pre-requisites

- Linux host with `iptables` available and `sudo` access.
- OpenCode installed and configured with a valid API key.
- A test project directory.

## Experiment Steps

### 1. Establish baseline (control group)

```bash
# Start OpenCode on a test project
cd /tmp/test-project && opencode

# In the agent UI, issue a simple task:
#   "List the files in this directory."

# Observe: agent completes the task and returns output.
# Record: wall-clock time from prompt to response (expected < 10 s).
```

### 2. Inject fault (experimental group)

```bash
# In a second terminal, identify the LLM provider hostname, e.g. api.anthropic.com
PROVIDER_HOST="api.anthropic.com"
PROVIDER_IP=$(dig +short "$PROVIDER_HOST" | head -1)

# Block outbound traffic to the provider
sudo iptables -I OUTPUT -d "$PROVIDER_IP" -j DROP

echo "Fault injected: outbound traffic to $PROVIDER_IP blocked"
```

### 3. Exercise the system under fault

```bash
# Back in the OpenCode session, issue the same task:
#   "List the files in this directory."

# Observe and record:
# - Does the agent return an error message? Within how many seconds?
# - Does the agent process remain responsive?
# - Are there any retry loops visible in logs?
```

### 4. Collect metrics

```bash
# Watch agent logs (if available) for error/retry patterns
tail -f ~/.local/share/opencode/logs/opencode.log

# Record:
# - Time until first error visible to user
# - Whether the process can be cleanly interrupted (Ctrl-C)
# - Any output to stderr
```

### 5. Remove fault

```bash
sudo iptables -D OUTPUT -d "$PROVIDER_IP" -j DROP
echo "Fault removed"
```

### 6. Verify recovery

```bash
# Issue the same task again; the agent should succeed within the baseline time.
```

## Results

| Metric | Control Group | Experimental Group |
|--------|--------------|-------------------|
| Time to first response | < 10 s | Hung indefinitely (> 5 min observed) |
| Error surfaced to user | N/A | None — no error message displayed |
| Process responsive to Ctrl-C | Yes | No — required SIGKILL |
| Recovery after fault removal | N/A | Not tested (process had to be killed) |

## Conclusion

Steady state was **broken**. OpenCode had no timeout on the LLM API call; when the endpoint became unreachable the agent session hung with no user-visible error and no way to recover without killing the process. This represents a cascading failure from a single point of dependency (the LLM provider).

## Recommended Fix

- Add a configurable connect/read timeout to all LLM API calls.
- Surface timeout errors immediately as a human-readable message.
- Implement exponential backoff with a maximum retry count and a final "service unavailable" message.
- Allow the user to cancel an in-flight request via Ctrl-C.
