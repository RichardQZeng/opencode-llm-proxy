# Cross-Agent Compatibility Review

Review target: `dev` at `8091790`, compared with upstream `main` at `8c262c4`.

## Status

The current changes work for the verified MaxKB and Cube MCP flow, but are not yet safe for all agents and protocol clients.

## Findings

### High: Gemini keepalives corrupt NDJSON streams

`createSseQueue()` emits SSE comments (`: keep-alive\n\n`) for every buffered stream. Gemini streaming responses use `application/x-ndjson`, where every non-empty line must be JSON. A model taking more than five seconds to emit output causes standard Gemini clients to fail parsing before the response arrives.

References: `index.js:1556-1574`, `index.js:2746-2754`

Recommended fix: make heartbeat framing protocol-specific, or disable SSE comments for Gemini NDJSON responses.

### High: Prompt failures and early cancellation can hang

`promptAsync()` rejections are swallowed while the event iterator is consumed. If OpenCode rejects without emitting `session.error` or `session.idle`, the request remains active until the proxy timeout. An abort signal that is already aborted before listener registration can also leave the native event listener open indefinitely.

References: `index.js:24-52`, `index.js:1275-1288`, `index.js:1350-1351`

Recommended fix: race prompt failure, event completion, and cancellation explicitly; close listeners immediately when `signal.aborted` is already true.

### High: Plugin reload leaves stale runtime state

The plugin starts a Bun server and stores a global `started` flag, but returns no `dispose` hook. OpenCode reloads or multiple project instances can therefore retain the old server and client while the new plugin instance returns no event hooks. Tool requests then wait for events that cannot reach the stale listener set.

References: `index.js:2831-2906`

The installed official OpenCode `1.18.22` plugin types define `Hooks.dispose?: () => Promise<void>`.

Recommended fix: return a `dispose` hook that stops the server, closes event listeners, and clears the global state owned by that plugin instance.

### High: Continuation turns can lose text before tool calls

Multi-message requests enable assistant-envelope cleanup and suppress live text deltas. Buffered text is replayed only when the result has no tool calls. An assistant response containing text followed by one or more tool calls therefore returns only the tool calls.

References: `index.js:420`, `index.js:767-768`

Recommended fix: retain and emit legitimate text alongside tool calls after envelope processing.

### Medium: Envelope cleanup can alter legitimate content

`unwrapAssistantMessage()` searches for assistant-shaped JSON at the end of a response and replaces the entire response with its text blocks. A legitimate answer explaining or demonstrating that JSON shape can lose its preceding prose and requested JSON.

References: `index.js:627-641`

Recommended fix: unwrap only when the complete trimmed response is the exact envelope, rather than accepting preceding content.

### Medium: Structured-output instructions conflict

JSON-schema requests still receive the system instruction `Do not emit JSON or a message envelope`, even though the caller explicitly requested structured JSON. Mock-based structured-output tests do not exercise model behavior under these contradictory instructions.

References: `index.js:410-415`, `index.js:675-681`

Recommended fix: omit the non-JSON instruction when a structured output format is active.

### Medium: Bun idle timeout is lower than allowed request timeout

The proxy permits application request timeouts up to 3,600 seconds, but Bun's idle timeout is capped at 255 seconds. Slow non-streaming requests produce no bytes and can be terminated by Bun before the configured application timeout.

References: `index.js:148`, `index.js:2860-2865`, `README.md:187`

Recommended fix: use `idleTimeout: 0` or disable the timeout per request when the configured request timeout exceeds Bun's cap.

## Verification

- Full test suite: 255 passed
- Protocol conformance suite: 4 passed
- ESLint: passed
- `git diff --check`: passed

The existing suites do not cover delayed Gemini output, prompt rejection without terminal events, pre-aborted signals, plugin reload, text-plus-tool continuation turns, or real-model structured-output instruction conflicts.
