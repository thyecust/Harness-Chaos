# CHE-001 · OpenCode — Session Poisoning from Invalid Image Data

**Database entry**: [CHE-001](../database.md#che-001--opencode--session-poisoning-from-invalid-image-data)
**GitHub issue**: https://github.com/anomalyco/opencode/issues/4781

---

## Hypothesis

If the LLM API rejects a request containing an oversized image attachment, OpenCode will surface a clear error for that request and allow the session to continue normally on subsequent messages.

## Steady State Definition

- The agent completes tool-call cycles and returns results to the user.
- A transient API error (e.g., oversized image) does not affect subsequent requests in the same session.
- The user can continue issuing prompts after an error without manual intervention.

## Variables Introduced

| Variable | Method |
|----------|--------|
| Oversized image in session | Attach an image file exceeding provider limits (> 5 MB) via the `--file` flag, causing OpenCode's read tool to embed it as base64 in the session history |
| Provider switch | Continue the same session with a model whose provider enforces strict image size limits (Anthropic Claude, 5 MB maximum) |
| Follow-up messages | Issue unrelated text-only prompts after the provider rejection occurs |

## Pre-requisites

- OpenCode v1.4.7 installed and configured with an API gateway that routes to multiple providers.
- A test project directory (git-initialized).
- An image file exceeding 5 MB. This experiment uses a 19 MB NASA JPEG: https://www.nasa.gov/wp-content/uploads/2026/04/nhq202604060006.jpg
- Two providers configured: one tolerant of large images (e.g., GPT-5 via `@ai-sdk/openai-compatible`) and one with strict limits (e.g., Claude Opus 4.6 via `@ai-sdk/anthropic`).

## Experiment Steps

### 1. Establish baseline (control group)

```bash
# Configure OpenCode with both providers in .opencode/opencode.json
# Provider A: OpenAI-compatible (tolerant of large images)
# Provider B: Anthropic-compatible (enforces 5 MB image limit)

# Start OpenCode and verify basic functionality
cd /tmp/che-001-test && opencode run -m provider-a/gpt-5 "List the files in this directory."

# Observe: agent completes the task and returns output.
# Record: wall-clock time from prompt to response (expected < 10 s).
# Result: Agent ran `ls`, listed files, responded in ~5 s. ✓
```

### 2. Inject fault (experimental group)

```bash
# Download a large image (19 MB JPEG)
curl -o /tmp/che-001-test/nhq202604060006.jpg \
  https://www.nasa.gov/wp-content/uploads/2026/04/nhq202604060006.jpg

# Attach it to a new session via --file (this triggers the read tool,
# which embeds the full image as base64 in the session history)
cd /tmp/che-001-test && opencode run -m provider-a/gpt-5 \
  --dangerously-skip-permissions \
  --file nhq202604060006.jpg \
  -- "Describe this image in one sentence."

# Observe: the tolerant provider does not reject the image,
# but cannot view it. The 19 MB JPEG is now stored as ~25 MB base64 in the
# session's SQLite history (two copies: file attachment + read tool result).
```

### 3. Verify poison payload is stored

```bash
# Get the session ID
SID=$(sqlite3 ~/.local/share/opencode/opencode.db \
  "SELECT id FROM session ORDER BY time_created DESC LIMIT 1;")

# Check the size of base64 image data in the session
sqlite3 ~/.local/share/opencode/opencode.db \
  "SELECT length(data) FROM part WHERE session_id = '$SID' AND data LIKE '%base64%';"

# Result: Two entries of ~25 MB each (25410385 and 25410869 bytes).
# Total poisoned data in session: ~50 MB.
```

### 4. Trigger the provider rejection

```bash
# Continue the SAME session but switch to Claude Opus 4.6,
# whose provider (Anthropic) enforces a 5 MB image size limit
cd /tmp/che-001-test && opencode run \
  -m provider-b/claude-opus-4-6 \
  --dangerously-skip-permissions \
  -s "$SID" \
  "Just say hello."

# Observe: the API gateway forwards the session history (including the 25 MB
# base64 image) to Claude, which rejects it with a 400 error:
#
#   Error: Bad Request: {"error":{"message":"supplier response failed,
#     original body is [{\"message\":\"messages.2.content.0.tool_result.content.1
#     .image.source.base64: image exceeds 5 MB maximum: 25410268 bytes > 5242880
#     bytes\"}]", "type":"SupplierResponseFailedError"}}
```

### 5. Verify session is permanently poisoned

```bash
# Attempt 1: explicitly ask to ignore images
cd /tmp/che-001-test && opencode run \
  -m provider-b/claude-opus-4-6 \
  --dangerously-skip-permissions \
  -s "$SID" \
  "Ignore all images. Just say hello."

# Result: Same 400 error. Image data is baked into history, not the prompt.

# Attempt 2: minimal prompt
cd /tmp/che-001-test && opencode run \
  -m provider-b/claude-opus-4-6 \
  --dangerously-skip-permissions \
  -s "$SID" \
  "hi"

# Result: Same 400 error. Session is permanently dead.
```

### 6. Verify new session is unaffected

```bash
# Start a fresh session with the same provider
cd /tmp/che-001-test && opencode run \
  -m provider-b/claude-opus-4-6 \
  --dangerously-skip-permissions \
  "Say hello in one sentence."

# Result: "Hello, welcome — how can I help you today?" ✓
# New sessions are unaffected; only the poisoned session is dead.
```

## Results

| Metric | Control Group | Experimental Group |
|--------|--------------|-------------------|
| Time to first response | ~5 s | ~3 s (400 error returned quickly) |
| Error surfaced to user | N/A | Yes — `SupplierResponseFailedError: image exceeds 5 MB maximum: 25410268 bytes > 5242880 bytes` |
| Subsequent text-only prompts succeed | Yes | **No** — same 400 error on every subsequent message |
| Session recoverable without manual intervention | N/A | **No** — only manual SQLite surgery or starting a new session restores functionality |
| Explicitly asking agent to ignore images | N/A | **No effect** — base64 image data is embedded in stored history, not in the user's prompt |
| Base64 data stored in SQLite | N/A | **~50 MB** across two `part` records (file attachment + read tool result) |

## Reproduction Environment

| Component | Version / Value |
|-----------|----------------|
| OpenCode | 1.4.7 |
| Tolerant provider | GPT-5 via `@ai-sdk/openai-compatible` |
| Strict provider | Claude Opus 4.6 via `@ai-sdk/anthropic` |
| API gateway | OpenAI-compatible gateway routing to multiple LLM providers |
| Test image | [nhq202604060006.jpg](https://www.nasa.gov/wp-content/uploads/2026/04/nhq202604060006.jpg) (19 MB, NASA public domain) |

## Conclusion

Steady state was **broken**. When an oversized image is embedded in a session via the read tool, OpenCode persists the full base64-encoded image data (~25 MB per copy) in its local SQLite database (`~/.local/share/opencode/opencode.db`) as part of the session's message history. Every subsequent API call replays the full stored history — including the oversized image. When the target provider enforces image size limits (Anthropic's 5 MB maximum), the provider rejects every request with the same 400 error. The session is permanently poisoned with no built-in recovery path.

The poisoning mechanism has two stages:
1. **Storage**: The image is stored as base64 in the `part` table, inside `tool_result` content blocks (path: `messages.N.content.0.tool_result.content.1.image.source.base64`).
2. **Replay**: On every subsequent API call, OpenCode replays the full session history including the poisoned `part` records. The provider's image size validation fails before processing the new user message.

This affects users who encounter oversized images through the read tool, clipboard paste, `--file` flag, or MCP tools. The issue has multiple duplicate reports (#14562, #12068, #13865, #21668) and confirmed production occurrences with Claude Opus 4.6.

## Recommended Fix

1. **Auto-revert on 4xx** (root cause fix): When the API gateway returns a 4xx error, automatically revert or mark-as-skipped the message that triggered the error, so subsequent API calls replay a clean history.
2. **Pre-send validation via `messages.transform`** (defense in depth): Before each LLM call, scan the message history for base64 images exceeding provider limits and replace them with placeholder text (e.g., `[Image removed: exceeded 5 MB limit]`).
3. **Session recovery command**: Provide a built-in `opencode session repair` command that strips oversized attachments without requiring manual SQLite surgery.
4. **Image resizing**: Consider resizing oversized images automatically before storing them (see PR #12069).
