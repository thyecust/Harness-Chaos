# CHE-002 · OpenCode — Retry Storm / Improper Timeout

**Database entry**: [CHE-002](../database.md#che-002--opencode--retry-storm--improper-timeout)  
**GitHub issue**: https://github.com/anomalyco/opencode/issues/17307

---

## Hypothesis

If the LLM provider's SSE (Server-Sent Events) stream is slow or drops mid-response, OpenCode will apply exponential backoff before retrying, limiting the request rate to the provider and keeping the agent responsive.

## Steady State Definition

- SSE stream delivers tokens within 5 seconds of the first byte.
- On a transient stream error, the agent retries at most 3 times with at least 1 s between attempts.
- The agent's outbound request rate to the LLM provider remains ≤ 2 requests per minute during degraded conditions.

## Variables Introduced

| Variable | Method |
|----------|--------|
| Slow / dropped SSE stream | `tc netem` to inject 3 s latency + 20% packet loss on the loopback interface of a local proxy |
| Duration | 5 minutes |

## Pre-requisites

- Linux host with `tc` (iproute2) and `iproute2-tc` available.
- A local HTTP proxy (e.g., `mitmproxy`) forwarding to the real LLM provider, so traffic can be shaped at the loopback interface.
- OpenCode configured to use the local proxy (`HTTPS_PROXY=http://127.0.0.1:8080`).
- Packet-capture tool (`tcpdump` or `ss`) to count outbound requests.

## Experiment Steps

### 1. Establish baseline (control group)

```bash
# Start mitmproxy in transparent mode
mitmproxy --mode regular --listen-port 8080 &

# Start OpenCode through the proxy
HTTPS_PROXY=http://127.0.0.1:8080 opencode

# Issue a task that requires a long LLM response:
#   "Explain the Rust borrow checker in detail."

# Record: number of SSE stream connections made (via mitmproxy flow count).
# Expected: exactly 1 connection, response completes normally.
```

### 2. Inject fault — high latency and packet loss

```bash
# Add 3 s delay + 20% packet loss on the loopback interface
sudo tc qdisc add dev lo root netem delay 3000ms loss 20%
echo "Fault injected: 3 s latency + 20% loss on loopback"
```

### 3. Exercise the system under fault

```bash
# In the OpenCode session, issue the same task again.
# In a separate terminal, monitor outbound connection count:
watch -n 1 'ss -tn dst :443 | wc -l'

# Also watch mitmproxy's flow list for repeated requests to the LLM endpoint.
# Record:
# - Number of retry attempts within the first 60 s
# - Time between retries
# - Any error messages surfaced to the user
```

### 4. Collect metrics

```bash
# Count SSE connection attempts logged by mitmproxy
# (mitmproxy writes to ~/.mitmproxy/mitmproxy.log)
grep -c "SSE\|stream\|completion" ~/.mitmproxy/mitmproxy.log

# Or capture with tcpdump
sudo tcpdump -i lo -n 'dst port 8080' -w /tmp/che-002-capture.pcap &
# ... wait 60 s ...
kill %2
tcpdump -r /tmp/che-002-capture.pcap | grep -c "SYN"
```

### 5. Remove fault

```bash
sudo tc qdisc del dev lo root
echo "Fault removed"
```

### 6. Verify recovery

```bash
# Issue the task again; it should complete with a single SSE connection.
```

## Results

| Metric | Control Group | Experimental Group |
|--------|--------------|-------------------|
| SSE connections per task | 1 | 47 in 60 s (observed) |
| Min time between retries | N/A | < 100 ms (no backoff) |
| User-visible error | None (success) | `SSE read timed out` (repeated) |
| Outbound request rate | ~0.02 req/s | ~0.8 req/s sustained |

## Conclusion

Steady state was **broken**. When the SSE stream timed out, OpenCode immediately retried without any backoff. Within one minute, 47 connection attempts were made to the LLM provider. This is a retry storm: under a partial outage, the agent amplifies load rather than backing off, which can worsen the outage for all users and exhaust rate-limit quotas.

## Recommended Fix

- Replace the immediate retry with an exponential backoff strategy (e.g., 1 s, 2 s, 4 s, … up to a maximum of 60 s).
- Cap the total number of retries per request (e.g., 5 attempts).
- Expose `SSE_READ_TIMEOUT` and `MAX_RETRIES` as user-configurable settings.
- Surface a single consolidated error message to the user rather than silently retrying.
