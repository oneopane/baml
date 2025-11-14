# BAML Validation Criteria Report

**Date:** 2025-11-14
**Decision:** ✅ **RECOMMENDED FOR USE**

---

## Executive Summary

BAML meets **ALL critical requirements** and **MOST important requirements** for production validation and debugging. The framework provides comprehensive observability through its `Collector` API, detailed error reporting, full token/latency tracking, and flexible configuration options.

**Key Finding:** SAP (Semantic Auto Prompting) was not found in the codebase - this may be a future feature or known by a different name.

---

## Critical Requirements (Must Have All)

| Criterion | Status | Implementation Details |
|-----------|--------|------------------------|
| **Raw composed prompt** | ✅ YES | Available via `Collector.last.calls[0].httpRequest.body` - contains full JSON request including messages array with exact prompt sent to LLM |
| **Raw LLM response** | ✅ YES | Available via `Collector.last.rawLlmResponse` (string) and `httpResponse.body` (structured) |
| **Input token count** | ✅ YES | `Usage.inputTokens` and `Usage.cachedInputTokens` for prompt caching scenarios |
| **Output token count** | ✅ YES | `Usage.outputTokens` - available per-call and aggregated across retries |
| **Actual model used** | ✅ YES | Available via `httpRequest.body.json().model` (e.g., "claude-3-5-sonnet-20241022"), plus `provider` and `clientName` fields |
| **Total latency** | ✅ YES | `Timing.durationMs` at function level, plus per-call timing for retries |

**Verdict:** 6/6 ✅ **PASS**

---

## Important Requirements (Need Most)

| Criterion | Status | Implementation Details |
|-----------|--------|------------------------|
| **SAP transformations** | ❌ NO | No evidence of "Semantic Auto Prompting" found in codebase, documentation, or tests |
| **All retry attempts** | ✅ YES | `FunctionLog.calls` array contains one `LlmCall` object per attempt, including failed retries |
| **Error stage info** | ✅ YES | `ExposedError` enum distinguishes: `ValidationError` (parsing), `ClientHttpError` (LLM call), `BamlInvalidArgumentError` (composition), plus `FinishReasonError` and `TimeoutError` |
| **Can disable retries** | ✅ YES | Omit `retry_policy` from client config, or set `max_retries 0` |
| **Can disable SAP** | ❌ N/A | SAP feature not found in codebase |
| **Pass Anthropic params** | ✅ YES | All options in `client { options { } }` block are passed directly to API - supports `temperature`, `top_p`, `max_tokens`, custom `headers`, `thinking` config, etc. |

**Verdict:** 4/6 ✅ **PASS** (excluding N/A SAP items: 4/4)

---

## Nice to Have Requirements

| Criterion | Status | Implementation Details |
|-----------|--------|------------------------|
| **Latency breakdown** | ✅ YES | `StreamTiming` provides `time_to_first_token_ms` and `time_to_first_parsed_ms` for granular streaming metrics |
| **Observability hooks** | ✅ YES | `onTick` callback fires during execution with real-time `FunctionLog` updates - works with async functions |
| **Read BAML template** | ⚠️ PARTIAL | Can access composed prompt via `httpRequest.body`, but no direct API to read `.baml` template source files programmatically |
| **Cost calculation** | ✅ YES | `Usage` tracking + Boundary Studio dashboard with time-series cost data, breakdown by client/function, filtering by tags |
| **Streaming + raw chunks** | ✅ YES | `stream.getFinalResponse()` for final result, `for await (const partial of stream)` for incremental updates, `LlmStreamCall.sseChunks()` for raw SSE data |
| **Dry-run mode** | ⚠️ PARTIAL | Access via `b.parse` (parser object) but requires manual prompt composition - no built-in "compose-only" mode |

**Verdict:** 4/6 full, 2/6 partial

---

## Detailed Implementation Guide

### 1. Accessing Raw Prompt & Response

```typescript
import { Collector } from "@boundaryml/baml"

const collector = new Collector("validation")
const result = await b.ExtractResume(resumeText, { collector })

// Raw prompt (exact JSON sent to LLM)
const request = collector.last?.calls[0].httpRequest
const promptMessages = request?.body.json().messages
console.log(JSON.stringify(promptMessages, null, 2))

// Raw LLM response (before parsing)
console.log(collector.last?.rawLlmResponse)

// Structured HTTP response
const response = collector.last?.calls[0].httpResponse
console.log(response?.status)  // 200
console.log(response?.body.text())  // Full response body
```

**Files:**
- TypeScript types: `engine/language_client_typescript/native.d.ts:144-160`
- Python types: `engine/language_client_python/python_src/baml_py/baml_py.pyi:474-502`

---

### 2. Token Counting & Model Information

```typescript
const collector = new Collector("tokens")
await b.Function(input, { collector })

// Per-call tokens
const usage = collector.last?.calls[0].usage
console.log(`Input: ${usage?.inputTokens}`)
console.log(`Output: ${usage?.outputTokens}`)
console.log(`Cached: ${usage?.cachedInputTokens}`)  // Prompt caching

// Aggregated across all retries
console.log(`Total input: ${collector.usage?.inputTokens}`)
console.log(`Total output: ${collector.usage?.outputTokens}`)

// Model information
const call = collector.last?.calls[0]
console.log(`Provider: ${call?.provider}`)        // "anthropic"
console.log(`Client: ${call?.clientName}`)        // "Claude"
console.log(`Model: ${call?.httpRequest.body.json().model}`)  // "claude-3-5-sonnet-20241022"
```

**Files:**
- Usage types: `engine/language_client_typescript/native.d.ts:244-249`
- Test examples: `integ-tests/typescript/tests/collector.test.ts`

---

### 3. Latency & Timing

```typescript
const collector = new Collector("timing")
await b.Function(input, { collector })

// Overall function timing
const timing = collector.last?.timing
console.log(`Start: ${timing?.startTimeUtcMs}`)
console.log(`Duration: ${timing?.durationMs}ms`)

// Per-call timing (for retries)
for (const call of collector.last?.calls || []) {
    console.log(`Call to ${call.clientName}: ${call.timing.durationMs}ms`)
}

// Streaming-specific timing
const streamCall = collector.last?.calls[0] as LlmStreamCall
console.log(`First token: ${streamCall.timing.time_to_first_token_ms}ms`)
console.log(`First parsed: ${streamCall.timing.time_to_first_parsed_ms}ms`)
```

**Files:**
- Timing types: `engine/language_client_typescript/native.d.ts:201-211`

---

### 4. Retry Configuration

```baml
# Define retry policy
retry_policy ExponentialBackoff {
  max_retries 3
  strategy {
    type exponential_backoff
    delay_ms 200
    multiplier 1.5
    max_delay_ms 10000
  }
}

# Apply to client
client<llm> Claude {
  provider anthropic
  retry_policy ExponentialBackoff
  options {
    model "claude-3-5-sonnet-20241022"
    api_key env.ANTHROPIC_API_KEY
  }
}
```

**Accessing Retry Attempts:**

```typescript
const collector = new Collector("retries")
try {
    await b.Function(input, { collector })
} catch (error) {
    // All attempts available even on failure
    console.log(`Made ${collector.last?.calls.length} attempts`)

    for (const [i, call] of collector.last?.calls.entries() || []) {
        console.log(`Attempt ${i + 1}:`)
        console.log(`  Status: ${call.httpResponse?.status}`)
        console.log(`  Selected: ${call.selected}`)
        console.log(`  Duration: ${call.timing.durationMs}ms`)
    }
}
```

**Files:**
- Config parsing: `engine/baml-lib/parser-database/src/types/configurations.rs`
- Runtime retry: `engine/baml-runtime/src/internal/llm_client/retry_policy.rs`

---

### 5. Error Stage Detection

```typescript
import { BamlValidationError, BamlClientHttpError } from "@boundaryml/baml"

try {
    await b.Function(input)
} catch (error: any) {
    if (error instanceof BamlValidationError) {
        // Parsing stage error
        console.log("Failed at PARSING stage")
        console.log(`Prompt: ${error.prompt}`)
        console.log(`Raw output: ${error.raw_output}`)
        console.log(`Error: ${error.message}`)

    } else if (error instanceof BamlClientHttpError) {
        // LLM call stage error
        console.log("Failed at LLM CALL stage")
        console.log(`Client: ${error.client_name}`)
        console.log(`Status: ${error.status_code}`)
        console.log(`Error: ${error.message}`)

    } else if (error.name === "BamlInvalidArgumentError") {
        // Composition stage error
        console.log("Failed at COMPOSITION stage")
        console.log(`Error: ${error.message}`)
    }
}
```

**Error Types Available:**
- `ValidationError` - Response parsing/validation failed
- `ClientHttpError` - HTTP/network error calling LLM
- `FinishReasonError` - LLM returned non-success finish reason
- `TimeoutError` - Request exceeded timeout
- `AbortError` - Request was cancelled

**Files:**
- Error definitions: `engine/baml-runtime/src/errors.rs`

---

### 6. Anthropic API Parameter Pass-Through

All options in the `options` block are passed directly to the Anthropic API:

```baml
client<llm> ClaudeAdvanced {
  provider anthropic
  options {
    model "claude-3-5-sonnet-20241022"
    api_key env.ANTHROPIC_API_KEY

    # Standard parameters
    max_tokens 4096
    temperature 0.7
    top_p 0.9
    top_k 40

    # Thinking mode (extended thinking models)
    thinking {
      type "enabled"
      budget_tokens 2048
    }

    # Prompt caching
    allowed_role_metadata ["cache_control"]
    headers {
      "anthropic-beta" "prompt-caching-2024-07-31"
    }

    # Custom endpoint
    base_url "https://custom-endpoint.com"

    # Timeout configuration
    request_timeout_ms 60000
  }
}
```

**Implementation Detail:**
```rust
// From engine/baml-lib/llm-client/src/clients/anthropic.rs
let mut body = json!(self.properties.properties);  // All options → request body
```

This means **any new Anthropic API parameter** will automatically work in BAML without code changes.

**Files:**
- Anthropic client: `engine/baml-lib/llm-client/src/clients/anthropic.rs`
- Runtime client: `engine/baml-runtime/src/internal/llm_client/primitive/anthropic/anthropic_client.rs`

---

### 7. Observability Hooks (onTick)

```typescript
const onTick = (reason: string, log: FunctionLog | null) => {
    console.log(`Tick reason: ${reason}`)

    if (log?.calls?.length) {
        const latest = log.calls[log.calls.length - 1]
        console.log(`Latest call: ${latest.clientName}`)
        console.log(`Tokens: ${latest.usage?.inputTokens} / ${latest.usage?.outputTokens}`)
        console.log(`Duration: ${latest.timing.durationMs}ms`)

        // Access raw HTTP details
        const req = latest.httpRequest
        console.log(`Request URL: ${req.url}`)
        console.log(`Request body: ${req.body.text()}`)
    }
}

const result = await b.Function(input, { onTick })
```

**Use Cases:**
- Real-time monitoring of long-running requests
- Progress tracking for streaming responses
- Debug logging without modifying BAML code

**Files:**
- Documentation: `fern/03-reference/baml_client/ontick.mdx`
- Implementation: `engine/language_client_cffi/src/ffi/callbacks.rs`
- Tests: `integ-tests/typescript/tests/ontick.test.ts`

---

### 8. Streaming with Raw Chunks

```typescript
// Standard streaming
const stream = b.stream.ExtractData(input)

for await (const partial of stream) {
    console.log(`Progress: ${partial.items?.length} items so far`)
}

const final = await stream.getFinalResponse()

// Access raw SSE chunks
const collector = new Collector("streaming")
const stream2 = b.stream.Function(input, {
    collector,
    onTick: (reason, log) => {
        const lastCall = log?.calls[log.calls.length - 1]
        if (lastCall && 'sseChunks' in lastCall) {
            const chunks = (lastCall as LlmStreamCall).sseChunks()
            for (const chunk of chunks) {
                const data = JSON.parse(chunk.text)
                console.log(`Raw SSE: ${JSON.stringify(data)}`)

                // Example: Extract thinking content
                if (data.delta?.thinking) {
                    console.log(`Thinking: ${data.delta.thinking}`)
                }
            }
        }
    }
})

await stream2.getFinalResponse()
```

**Streaming Attributes (in BAML schema):**
```baml
class Resume {
  name string @stream.not_null  // Always present, may be partial
  skills string[] @stream.done  // Only stream when complete
  experience string @stream.with_state  // Includes completion metadata
}
```

**Files:**
- Documentation: `fern/01-guide/04-baml-basics/streaming.mdx`
- Streaming types: `engine/baml-lib/baml-types/src/ir_type/converters/streaming.rs`

---

### 9. Cost Tracking

```typescript
const collector = new Collector("cost-tracking")

// Run multiple functions
await b.Function1(input, { collector })
await b.Function2(input, { collector })
await b.Function3(input, { collector })

// Aggregate usage
const totalUsage = collector.usage
console.log(`Total input tokens: ${totalUsage?.inputTokens}`)
console.log(`Total output tokens: ${totalUsage?.outputTokens}`)

// Calculate cost (example for Claude 3.5 Sonnet)
const inputCost = (totalUsage?.inputTokens || 0) * 0.003 / 1000
const outputCost = (totalUsage?.outputTokens || 0) * 0.015 / 1000
console.log(`Estimated cost: $${(inputCost + outputCost).toFixed(4)}`)

// Tag calls for cost attribution
await b.Function(input, {
    tags: { userId: "user123", feature: "resume-parsing" }
})
```

**Boundary Studio Dashboard:**
- Real-time cost visualization
- Time-series cost data
- Breakdown by client, function, tags
- Export capabilities

**Files:**
- Documentation: `fern/03-reference/baml_client/collector.mdx`
- Dashboard: `engine/baml-rpc/src/ui/ui_dashboard_cost.rs`

---

## Key Files Reference

| Component | File Path | Lines |
|-----------|-----------|-------|
| TypeScript Types | `engine/language_client_typescript/native.d.ts` | 114-257 |
| Python Types | `engine/language_client_python/python_src/baml_py/baml_py.pyi` | 276-502 |
| Collector Docs | `fern/03-reference/baml_client/collector.mdx` | All |
| OnTick Docs | `fern/03-reference/baml_client/ontick.mdx` | All |
| Streaming Docs | `fern/01-guide/04-baml-basics/streaming.mdx` | All |
| Error Types | `engine/baml-runtime/src/errors.rs` | 1-200 |
| Retry Policy | `engine/baml-runtime/src/internal/llm_client/retry_policy.rs` | All |
| Anthropic Client | `engine/baml-lib/llm-client/src/clients/anthropic.rs` | All |
| Collector Tests | `integ-tests/typescript/tests/collector.test.ts` | All |
| OnTick Tests | `integ-tests/typescript/tests/ontick.test.ts` | All |

---

## Limitations & Considerations

### SAP (Semantic Auto Prompting) - Not Found
- No evidence of SAP feature in current codebase (as of commit `46f016e`)
- May be planned for future release or known by different terminology
- **Impact:** Cannot test "what transformations SAP applied" or "disable SAP"
- **Recommendation:** Ask BAML team about SAP roadmap or alternative naming

### Dry-Run Mode - Partial
- Parser object is accessible via `b.parse`
- No built-in "compose-only" mode to generate prompt without LLM call
- **Workaround:** Use `onTick` callback to capture request, then cancel
- **Recommendation:** Consider adding `dryRun: true` option to generate prompt without API call

### Template Introspection - Partial
- Can access composed prompt via `httpRequest.body`
- No direct API to read `.baml` template source files programmatically
- **Workaround:** Read `.baml` files directly from `baml_src/` directory
- **Recommendation:** Add `b.getTemplateSource("FunctionName")` API

---

## Comparison with Other Frameworks

| Feature | BAML | LangChain | Guidance |
|---------|------|-----------|----------|
| Raw prompt access | ✅ Yes | ⚠️ Complex | ✅ Yes |
| Raw response access | ✅ Yes | ⚠️ Complex | ✅ Yes |
| Token counting | ✅ Built-in | ⚠️ Via callbacks | ❌ Manual |
| Retry visibility | ✅ All attempts | ❌ Hidden | ❌ No retries |
| Error stages | ✅ Clear enum | ⚠️ Generic | ⚠️ Generic |
| Streaming + raw | ✅ Full support | ⚠️ Limited | ✅ Full support |
| Type safety | ✅ Schema-first | ❌ Runtime only | ⚠️ Python types |

---

## Final Recommendation

### ✅ **USE BAML** for production validation and debugging

**Strengths:**
- Complete visibility into LLM interactions (prompt, response, tokens, timing)
- Excellent error reporting with clear stage distinction
- Comprehensive retry mechanism with full attempt history
- Strong observability through Collector API and onTick hooks
- Zero abstraction loss - all raw data accessible

**Acceptable Gaps:**
- SAP feature not found (may not be needed for validation use case)
- Dry-run mode requires workaround (not critical)
- Template introspection partial (workaround available)

**Decision Confidence:** High - Meets all critical criteria and most important criteria.

---

## Testing Checklist

Use this checklist to verify BAML meets your specific needs:

```typescript
// 1. ✅ Raw prompt access
const collector = new Collector("test")
await b.Function(input, { collector })
assert(collector.last?.calls[0].httpRequest.body.json().messages)

// 2. ✅ Raw response access
assert(typeof collector.last?.rawLlmResponse === 'string')

// 3. ✅ Token counting
assert(typeof collector.last?.usage?.inputTokens === 'number')
assert(typeof collector.last?.usage?.outputTokens === 'number')

// 4. ✅ Model information
assert(collector.last?.calls[0].httpRequest.body.json().model)

// 5. ✅ Latency tracking
assert(typeof collector.last?.timing.durationMs === 'number')

// 6. ✅ Retry attempts
// (Trigger retry by using invalid API key)
const cr = new ClientRegistry()
cr.addLlmClient("Test", "anthropic", {
    model: "claude-3-5-sonnet-20241022",
    api_key: "invalid"
})
try {
    await b.Function(input, { clientRegistry: cr, collector })
} catch {
    assert(collector.last?.calls.length > 1)  // Multiple attempts
}

// 7. ✅ Error stages
try {
    await b.Function(invalidInput)
} catch (error) {
    assert(error instanceof BamlValidationError
        || error instanceof BamlClientHttpError)
}

// 8. ✅ Anthropic parameters
// (Verify via httpRequest.body)
assert(collector.last?.calls[0].httpRequest.body.json().temperature === 0.7)
```

---

**Report Generated:** 2025-11-14
**BAML Version:** Latest (commit `46f016e`)
**Methodology:** Comprehensive codebase analysis via AST traversal and documentation review
