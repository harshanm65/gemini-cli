# Gemini CLI: `TypeError: fetch failed sending request` Investigation Report

**Date:** 2026-02-14 **Affected command:**
`gemini -p "explain the code architecture to me" -y` **CLI version:** 0.28.2
(globally installed at `/opt/homebrew/lib/node_modules/@google/gemini-cli`)
**Node.js:** v22.22.0 (built-in undici 6.23.0, npm undici 7.19.0/7.22.0)
**Platform:** macOS Darwin 25.2.0, arm64

---

## 1. Symptom

Running any non-trivial prompt through the Gemini CLI consistently crashes with:

```
Error: exception TypeError: fetch failed sending request
    at .../node_modules/@google/genai/dist/node/index.mjs:11550:19
    ...
An unexpected critical error occurred:[object Object]
```

Simple prompts like `"say hi"` sometimes worked, sometimes didn't. The failure
rate was ~100% for prompts that triggered tool use (e.g.,
`codebase_investigator` subagent) and variable for simple prompts.

---

## 2. Investigation Timeline

### 2.1 Initial triage: network connectivity

**Test:** Direct `curl` to the Gemini API endpoint.

```bash
curl -s -o /dev/null -w "%{http_code}" https://generativelanguage.googleapis.com
# Result: 404 (expected for root endpoint - server is reachable)
```

**Test:** Streaming API call using the genai SDK directly in Node.js.

```javascript
const { GoogleGenAI } = await import('.../genai/dist/node/index.mjs');
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
const r = await ai.models.generateContentStream({
  model: 'gemini-2.0-flash',
  contents: 'Say hello',
});
// Result: Works perfectly
```

**Finding:** The API key is valid, network is reachable, and the genai SDK works
in isolation. The issue is specific to the CLI process.

### 2.2 Identifying the request pattern

Using `NODE_DEBUG=fetch`, the CLI's request sequence was traced:

```
FETCH: connected to generativelanguage.googleapis.com using https:h1
FETCH: sending request to POST .../gemini-2.5-flash-lite:generateContent       <-- classifier
FETCH: received response to POST .../gemini-2.5-flash-lite:generateContent - HTTP 200
FETCH: trailers received from POST .../gemini-2.5-flash-lite:generateContent
FETCH: sending request to POST .../gemini-2.5-pro:streamGenerateContent?alt=sse <-- main request
FETCH: request to POST ... errored - other side closed                         <-- FAILURE
```

**Key observation:** The CLI uses a **model router (classifier)** that makes a
non-streaming request to `gemini-2.5-flash-lite` to decide whether to route to
`flash` or `pro`. This is implemented in
`packages/core/src/routing/strategies/classifierStrategy.ts`. After the
classifier response, the main streaming request fails with **"other side
closed"**.

The connections use **HTTP/1.1** (`h1`), not HTTP/2.

### 2.3 Reproducing in isolation

Multiple attempts to reproduce outside the CLI all **succeeded**:

| Test scenario                                              | Result |
| ---------------------------------------------------------- | ------ |
| Two sequential requests (same SDK instance)                | OK     |
| With large request body (50KB system prompt)               | OK     |
| With CLI-like custom headers (`User-Agent`, etc.)          | OK     |
| With concurrent telemetry request to `play.googleapis.com` | OK     |
| With 5-second delay between requests                       | OK     |
| With `gemini-3-pro-preview` model                          | OK     |
| With `gemini-2.5-pro` model                                | OK     |

**Finding:** The exact same request pattern works in standalone Node.js scripts
using the same SDK version and the same `GoogleGenAI` instance. Something
specific to the CLI process triggers the failure.

### 2.4 Bypassing the classifier

```bash
gemini -p "say hi" -y --model gemini-2.5-flash
# Result: Works (bypasses classifier, single request)

gemini -p "explain the code architecture to me" -y --model gemini-2.5-flash
# Result: Main request works, but codebase_investigator subagent fails
```

**Finding:** The issue is not limited to the classifier -> main request pattern.
**Any second API call** from the CLI process fails. The subagent makes its own
streaming request, which also hits the connection reuse bug.

### 2.5 The HTTP/1.1 vs HTTP/2 discovery

The `curl` response headers revealed the server supports HTTP/2:

```
HTTP/2 200
```

But the CLI connects via HTTP/1.1 (`h1`). Node.js's built-in `fetch()` (undici)
defaults to HTTP/1.1 unless `allowH2: true` is set on the dispatcher.

**Test:** Setting a global undici dispatcher with HTTP/2 enabled:

```javascript
import { Agent, setGlobalDispatcher } from 'undici';
setGlobalDispatcher(new Agent({ allowH2: true }));
// Then run the same two-request pattern...
// Result: WORKS (both requests succeed over h2)
```

**Finding:** HTTP/2 fixes the "other side closed" error because HTTP/2 handles
connection multiplexing properly and doesn't suffer from HTTP/1.1's sequential
connection reuse issues.

### 2.6 Patching the genai SDK for error details

The genai SDK wraps fetch errors, losing the original cause:

```javascript
// In @google/genai/dist/node/index.mjs:11549-11551
async apiCall(url, requestInit) {
    return fetch(url, requestInit).catch((e) => {
        throw new Error(`exception ${e} sending request`);  // Loses e.cause, e.code
    });
}
```

Patching the SDK to log the full error chain revealed:

```javascript
{
  message: 'fetch failed',
  code: undefined,
  cause: '...SSL routines:ssl3_read_bytes:ssl/tls alert bad record mac...',
  causeCode: 'ERR_SSL_SSL/TLS_ALERT_BAD_RECORD_MAC',
  causeCause: undefined
}
```

**Finding:** With HTTP/2 enabled, the error changes from "other side closed" to
an intermittent SSL bad record MAC error. The error code is
`ERR_SSL_SSL/TLS_ALERT_BAD_RECORD_MAC`, which was **NOT** in the CLI's retryable
error codes list (which had `ERR_SSL_SSLV3_ALERT_BAD_RECORD_MAC` instead).

### 2.7 Why retries didn't work

The CLI has a `retryFetchErrors` mechanism, but three issues prevented it from
working:

**Issue 1: Default is `false`**

In `packages/core/src/config/config.ts`:

```typescript
this.retryFetchErrors = params.retryFetchErrors ?? false;
```

**Issue 2: Schema defaults override config defaults**

The CLI's settings loading (`packages/cli/src/config/settings.ts`) uses
`mergeSettings()` which applies schema defaults first:

```typescript
const schemaDefaults = getDefaultsFromSchema();
// Schema has: retryFetchErrors: { default: false }
```

This means `settings.general?.retryFetchErrors` resolves to `false` (from schema
default), not `undefined`. So `false ?? true` evaluates to `false` (since `??`
only applies for `null`/`undefined`, not `false`).

**Issue 3: Connection pool stays broken**

Even with `retryFetchErrors: true`, retries failed because the underlying
HTTP/1.1 connection pool kept reusing the same dead connection. Every retry
attempt hit the same closed connection.

---

## 3. Root Cause Analysis

### Primary cause: HTTP/1.1 connection reuse failure

Node.js's undici-based `fetch()` defaults to HTTP/1.1 connections. The Gemini
API server closes the HTTP/1.1 keep-alive connection after responding to the
classifier request. When undici attempts to reuse this connection for the
subsequent streaming request, the server has already sent a FIN/RST, resulting
in "other side closed".

The reason this is not reproducible in standalone scripts is likely due to
timing: the CLI process has significant processing between the two requests
(model selection, tool configuration, hook firing, system prompt construction),
during which the server's keep-alive timeout may expire.

### Secondary cause: SSL errors with HTTP/2

Even with HTTP/2 enabled, intermittent SSL `bad record mac` errors occur. These
are transient TLS-level errors, likely related to connection state management in
Node.js's OpenSSL 3.5.4 or network conditions. Unlike the HTTP/1.1 issue, these
are genuinely transient and resolve on retry.

### Contributing factors

1. **genai SDK error wrapping** (`index.mjs:11549-11551`): Creates
   `new Error('exception ... sending request')` without preserving `cause`,
   making it impossible for the CLI's retry logic to inspect the original error
   code.

2. **Missing error code variant**: The retryable network codes list had
   `ERR_SSL_SSLV3_ALERT_BAD_RECORD_MAC` but not
   `ERR_SSL_SSL/TLS_ALERT_BAD_RECORD_MAC` (the actual code produced by Node.js
   22 + OpenSSL 3.5.4).

3. **Retry defaults**: `retryFetchErrors` defaulting to `false` meant the
   generic "fetch failed" message-based detection was disabled.

---

## 4. Architecture Context

### Request flow in the CLI

```
User Input
    |
    v
GeminiClient.sendMessageStream()
    |
    v
ModelRouterService (classifierStrategy.ts)
    |-- BaseLlmClient.generateJson()           <-- Non-streaming to flash-lite
    |   |-- retryWithBackoff()
    |   |-- contentGenerator.generateContent()  <-- 1st API call
    |
    v
GeminiChat.streamWithRetries()
    |-- makeApiCallAndProcessStream()
    |   |-- retryWithBackoff()
    |   |-- contentGenerator.generateContentStream()  <-- 2nd API call (FAILS)
    |
    v
Stream chunks / Tool calls
    |
    v
If tool call → SubagentToolWrapper → LocalAgentExecutor
    |-- GeminiChat (new instance, same contentGenerator)
    |-- streamWithRetries()                    <-- 3rd API call (also FAILS)
```

### Shared resources

All API calls share the same `GoogleGenAI` client instance (via
`contentGenerator`), which means they share the same HTTP connection pool. The
connection pool is the Node.js process-global undici dispatcher.

### Key files

| File                                                         | Role                                              |
| ------------------------------------------------------------ | ------------------------------------------------- |
| `packages/core/src/routing/strategies/classifierStrategy.ts` | Model router that makes the 1st API call          |
| `packages/core/src/core/geminiChat.ts`                       | Main chat loop with `streamWithRetries()`         |
| `packages/core/src/core/baseLlmClient.ts`                    | Utility LLM client used by classifier             |
| `packages/core/src/core/loggingContentGenerator.ts`          | Wraps `googleGenAI.models` with logging           |
| `packages/core/src/core/contentGenerator.ts`                 | Creates the `GoogleGenAI` instance                |
| `packages/core/src/utils/retry.ts`                           | Retry logic with `isRetryableError()`             |
| `packages/core/src/utils/fetch.ts`                           | Global proxy/dispatcher setup                     |
| `packages/core/src/config/config.ts`                         | Config constructor, initialization                |
| `packages/core/src/agents/local-executor.ts`                 | Subagent execution (reuses same contentGenerator) |
| `packages/cli/src/config/settingsSchema.ts`                  | Settings schema with defaults                     |
| `@google/genai/dist/node/index.mjs:11549`                    | SDK `apiCall()` that wraps fetch errors           |

---

## 5. Fix

### Changes made

#### A. Enable HTTP/2 by default (`packages/core/src/utils/fetch.ts`)

```typescript
import { Agent, ProxyAgent, setGlobalDispatcher } from 'undici';

export function enableHttp2() {
  setGlobalDispatcher(new Agent({ allowH2: true }));
}

export function setGlobalProxy(proxy: string) {
  setGlobalDispatcher(new ProxyAgent({ uri: proxy, allowH2: true }));
}
```

HTTP/2 eliminates the "other side closed" error by properly multiplexing
requests on a single connection. The `setGlobalProxy` function was also updated
to enable HTTP/2 when a proxy is configured.

#### B. Call `enableHttp2()` during initialization (`packages/core/src/config/config.ts`)

```typescript
import { enableHttp2, setGlobalProxy } from '../utils/fetch.js';

// In constructor:
const proxy = this.getProxy();
if (proxy) {
  try {
    setGlobalProxy(proxy);
  } catch (error) {
    // ... error handling
  }
} else {
  enableHttp2();
}
```

When no proxy is configured, enable HTTP/2 globally. When a proxy is configured,
`setGlobalProxy` already enables HTTP/2.

#### C. Default `retryFetchErrors` to `true` (`packages/core/src/config/config.ts`)

```typescript
this.retryFetchErrors = params.retryFetchErrors ?? true;
```

As a safety net, enable retry on "fetch failed" errors by default.

#### D. Update schema default (`packages/cli/src/config/settingsSchema.ts`)

```typescript
retryFetchErrors: {
  type: 'boolean',
  default: true,  // Changed from false
  // ...
},
```

The schema default must match because `mergeSettings()` applies schema defaults
before user settings, and the `??` operator in the config constructor doesn't
override `false` (only `null`/`undefined`).

#### E. Add missing SSL error code (`packages/core/src/utils/retry.ts`)

```typescript
const RETRYABLE_NETWORK_CODES = [
  // ...existing codes...
  'ERR_SSL_SSLV3_ALERT_BAD_RECORD_MAC',
  'ERR_SSL_SSL/TLS_ALERT_BAD_RECORD_MAC', // Added: actual code from Node.js 22 + OpenSSL 3.5.4
  // ...
];
```

### Verification

After applying all fixes:

```bash
# Before fix: 0% success rate
gemini -p "explain the code architecture to me" -y
# Error: exception TypeError: fetch failed sending request

# After fix: 100% success rate (5/5 attempts)
# Some attempts show "Attempt 1 failed. Retrying with backoff..." but all recover
node packages/cli/dist/index.js -p "explain the code architecture to me" -y
# Success: Full architecture explanation returned
```

---

## 6. Upstream Issues

### genai SDK (`@google/genai`)

The SDK's `apiCall` method at `index.mjs:11549-11551` wraps fetch errors without
preserving the cause chain:

```javascript
async apiCall(url, requestInit) {
    return fetch(url, requestInit).catch((e) => {
        throw new Error(`exception ${e} sending request`);
        // Should be: throw new Error(`exception ${e} sending request`, { cause: e });
    });
}
```

This makes it impossible for downstream code to inspect the original error code
(e.g., `ECONNRESET`, `ERR_SSL_*`). The CLI must fall back to string matching on
the error message ("fetch failed") instead of checking error codes, which is
fragile.

### Node.js undici

The default HTTP/1.1 behavior in Node.js's built-in `fetch()` is prone to "other
side closed" errors when servers close keep-alive connections. This is a known
class of issues with undici's connection pooling. Enabling HTTP/2 via
`allowH2: true` is the recommended mitigation.

---

## 7. Testing Notes

### How to reproduce the original bug

1. Install gemini-cli globally: `npm install -g @google/gemini-cli@0.28.2`
2. Set `GEMINI_API_KEY` environment variable
3. Run: `gemini -p "explain the code architecture to me" -y`
4. Observe: `TypeError: fetch failed sending request`

### How to verify the fix

1. Build the local dev version: `npm run build -w packages/core -w packages/cli`
2. Run:
   `node packages/cli/dist/index.js -p "explain the code architecture to me" -y`
3. Observe: Response is returned (may show retry messages but completes
   successfully)

### Workarounds (without the fix)

- Use `--model gemini-2.5-flash` to bypass the classifier (only works for
  prompts that don't trigger subagent tool calls)
- Set `retryFetchErrors: true` in `~/.gemini/settings.json` under `general`
  (doesn't help with HTTP/1.1 since the connection pool stays broken, but helps
  with intermittent HTTP/2 SSL errors)
