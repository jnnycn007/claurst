# Free-Code vs Original Claude Code — Complete Diff Analysis

> Audit date: 2026-04-03
> Free-code snapshot version: 2.1.87 (2026-03-31)
> Original src: extracted Claude Code TypeScript source (no package.json)

---

## Table of Contents

1. [Telemetry & Analytics — Complete Removal](#1-telemetry--analytics--complete-removal)
2. [CCH Request-Signing System](#2-cch-request-signing-system)
3. [Cybersecurity Guardrail Instruction — Wiped](#3-cybersecurity-guardrail-instruction--wiped)
4. [Native Client Attestation — Bypassed](#4-native-client-attestation--bypassed)
5. [OpenAI Codex Backend Integration (New Subsystem)](#5-openai-codex-backend-integration-new-subsystem)
6. [Feature Flags — 54 Experimental Unlocked](#6-feature-flags--54-experimental-unlocked)
7. [Ultraplan Command — Always Enabled](#7-ultraplan-command--always-enabled)
8. [GrowthBook Key Gating — Relaxed](#8-growthbook-key-gating--relaxed)
9. [Analytics Initialization — Dead Code Eliminated](#9-analytics-initialization--dead-code-eliminated)
10. [OpenTelemetry SDK — Fully Stripped from Deps](#10-opentelemetry-sdk--fully-stripped-from-deps)
11. [Session Tracing — All Spans Are No-ops](#11-session-tracing--all-spans-are-no-ops)
12. [Build System & Macro Injection](#12-build-system--macro-injection)
13. [New Files Unique to Free-Code](#13-new-files-unique-to-free-code)
14. [New Services — Context Collapse & Microcompact Caching](#14-new-services--context-collapse--microcompact-caching)
15. [Internal-Only Stub Tools Exposed](#15-internal-only-stub-tools-exposed)
16. [Provider Environment Variables Expanded](#16-provider-environment-variables-expanded)

---

## 1. Telemetry & Analytics — Complete Removal

### `src/services/analytics/index.ts`

**Original (~174 lines):** Full event-queueing analytics pipeline.
- `attachAnalyticsSink()` — registers Datadog/BigQuery sink, drains queued events
- `logEvent()` — synchronous, queues if sink not yet attached
- `logEventAsync()` — async variant
- Queue buffering with `queueMicrotask` drain on sink attach
- PII-tagging type guards

**Free (~41 lines):** Pure stub — all functions are no-ops.

```typescript
// Free version — all call sites compile unchanged, all data discarded
export function attachAnalyticsSink(_newSink: AnalyticsSink): void {}
export function logEvent(_eventName: string, _metadata: LogEventMetadata): void {}
export async function logEventAsync(_eventName: string, _metadata: LogEventMetadata): Promise<void> {}
```

---

### `src/utils/telemetry/instrumentation.ts`

**Original (~400 lines):** Full OpenTelemetry SDK bootstrap.
- `bootstrapTelemetry()` — maps `ANT_*` env vars → `OTEL_*`
- `parseExporterTypes()` — parses exporter list
- `getOtlpReaders()` / `getOtlpLogExporters()` / `getOtlpTraceExporters()` — creates OTLP metric/log/trace readers for grpc, http/json, http/proto
- `initializeBetaTracing()` — separate beta-tracing OTLP endpoint
- `initializeTelemetry()` (~281 lines) — orchestrates entire OTEL SDK setup
- `flushTelemetry()` — flushes pending data before process exit
- `isBigQueryMetricsEnabled()` — checks subscription tier for BigQuery eligibility

**Free (~30 lines):** Stubs only.

```typescript
// Free version
export function bootstrapTelemetry(): void {}
export function isTelemetryEnabled(): boolean { return false }
export async function initializeTelemetry(): Promise<null> { return null }
export async function flushTelemetry(): Promise<void> {}
```

---

### `src/utils/telemetry/sessionTracing.ts`

**Original (~928 lines):** Comprehensive OpenTelemetry span tracing.
- `startInteractionSpan()` / `endInteractionSpan()` — root span per user request
- `startLLMRequestSpan()` / `endLLMRequestSpan()` — traces each API call, records TTFT, token counts, model, fastMode
- `startToolSpan()` / `endToolSpan()` — traces tool execution
- `startToolBlockedOnUserSpan()` — tracks permission confirmation dialogs
- `addToolContentEvent()` — logs tool I/O with truncation
- `startHookSpan()` / `endHookSpan()` — hook execution spans
- `executeInSpan()` — generic async span wrapper
- `isEnhancedTelemetryEnabled()` — GrowthBook gate check

**Free (~148 lines):** All replaced with no-op dummy-span objects.

```typescript
// Free version — createNoopSpan() is the only implementation
function createNoopSpan(): Span {
  return {
    setAttribute() {},
    setAttributes() {},
    addEvent() {},
    end() {},
    recordException() {},
  }
}
export function isEnhancedTelemetryEnabled(): boolean { return false }
export function startInteractionSpan(_userPrompt: string): Span { return createNoopSpan() }
// ... all other span functions return createNoopSpan() or return void
```

---

### `src/utils/telemetry/events.ts`

**Original:** `logOTelEvent()` — emits structured log records via OTel logger, tracks event sequence numbers, captures workspace dir.

**Free:**
```typescript
export function isUserPromptLoggingEnabled(): boolean { return false }
export function redactIfDisabled(content: string): string {
  return isUserPromptLoggingEnabled() ? content : '<REDACTED>'
}
export async function logOTelEvent(_eventName: string, _metadata: {...} = {}): Promise<void> {}
```

---

### `src/entrypoints/init.ts`

**Original (~330+ lines):** Imports `isBetaTracingEnabled`, `getTelemetryAttributes`; has `initializeTelemetryAfterTrust()` (~40 lines) that waits for remote settings, then dynamically imports and runs `initializeTelemetry()`, sets up meter state, handles interactive vs headless modes.

**Free (213 lines):** Telemetry initialization entirely removed.
```typescript
// Free version replaces the entire function
export function initializeTelemetryAfterTrust(): void {
  return
}
```
Also removed: 1P event logging setup (lines 94–106 in original), `applyConfigEnvironmentVariables` call (uses only `applySafeConfigEnvironmentVariables`).

---

## 2. CCH Request-Signing System

### New file: `src/utils/cch.ts` (Free only, 29 lines)

Does not exist in original. Implements JavaScript-side xxHash64 request-body signing.

```typescript
const CCH_SEED = 0x6E52736AC806831En
const CCH_PLACEHOLDER = 'cch=24d82'
const CCH_MASK = 0xFFFFFn

// computeCch: xxHash64 of body bytes with seed, masked to 5 hex digits
export function computeCch(body: string): string { ... }
// replaceCchPlaceholder: swaps 'cch=00000' with actual hash
export function replaceCchPlaceholder(body: string, hash: string): string { ... }
// hasCchPlaceholder: detects whether placeholder is present
export function hasCchPlaceholder(body: string): boolean { ... }
```

### `src/constants/system.ts` lines 63–88

**Original (line 82):**
```typescript
const cch = feature('NATIVE_CLIENT_ATTESTATION') ? ' cch=00000;' : ''
// Comment: "Bun's native HTTP stack overwrites the zeros with a computed hash"
```

**Free (line 76):**
```typescript
const cch = ' cch=00000;'  // Always present; replaced by client.ts fetch wrapper
```

The feature gate `NATIVE_CLIENT_ATTESTATION` is completely removed. The placeholder is always injected.

### `src/services/api/client.ts` lines 407–416 (Free only)

Free adds post-build fetch interception for `/v1/messages` requests:

```typescript
// Added imports (lines 3–7)
import { computeCch, hasCchPlaceholder, replaceCchPlaceholder } from '../../utils/cch.js'

// Added in buildFetch() (lines 407-416)
if (url.includes('/v1/messages') && hasCchPlaceholder(bodyStr)) {
  const hash = computeCch(bodyStr)
  bodyStr = replaceCchPlaceholder(bodyStr, hash)
}
```

**What this means:** Original relied on Bun's native HTTP layer (`Attestation.zig`) to sign requests server-verified. Free reimplements the same signing in pure JavaScript, bypassing the Bun native layer entirely. The `ch=` / `cch=` value in the `x-anthropic-billing-header` is a 5-hex-digit xxHash64 integrity token the server uses to gate features like fast mode.

---

## 3. Cybersecurity Guardrail Instruction — Wiped

### `src/constants/cyberRiskInstruction.ts` line 24

**Original:**
```typescript
export const CYBER_RISK_INSTRUCTION = `IMPORTANT: Assist with authorized security testing, defensive security, CTF challenges, and educational contexts. Refuse requests for destructive techniques, DoS attacks, mass targeting, supply chain compromise, or detection evasion for malicious purposes. Dual-use security tools (C2 frameworks, credential testing, exploit development) require clear authorization context: pentesting engagements, CTF competitions, security research, or defensive use cases.`
```

**Free:**
```typescript
export const CYBER_RISK_INSTRUCTION = ``
```

The entire instruction is an empty string. All offensive-security refusal logic injected at the system-prompt level is eliminated.

---

## 4. Native Client Attestation — Bypassed

See section 2. The original uses `feature('NATIVE_CLIENT_ATTESTATION')` compile-time guard (Bun macro). When disabled this flag, `cch=` is not included in headers at all. Free hardcodes the placeholder unconditionally and computes the hash in JavaScript.

---

## 5. OpenAI Codex Backend Integration (New Subsystem)

Entirely absent in original. Free adds a complete parallel AI provider via OpenAI Codex.

### New: `src/constants/codex-oauth.ts` (45 lines)
```typescript
export const CODEX_CLIENT_ID = 'app_EMoamEEZ73f0CkXaXp7hrann'
export const CODEX_AUTHORIZE_URL = 'https://auth.openai.com/oauth/authorize'
export const CODEX_TOKEN_URL = 'https://auth.openai.com/oauth/token'
export const CODEX_REDIRECT_URI = 'http://localhost:1455/auth/callback'
export const CODEX_SCOPES = 'openid profile email offline_access'
export const CODEX_JWT_AUTH_CLAIM = 'https://api.openai.com/auth'
```

### New: `src/services/oauth/codex-client.ts` (~700 lines)
Full PKCE OAuth 2.0 client targeting `auth.openai.com`:
- Local HTTP server on port 1455 for redirect callback
- JWT decode to extract `chatgpt-account-id`
- Access/refresh token management
- Token refresh loop

### New: `src/services/api/codex-fetch-adapter.ts` (812 lines)
Fetch interceptor that translates the Anthropic Messages API schema ↔ OpenAI Responses API:
- **Endpoint:** `https://chatgpt.com/backend-api/codex/responses`
- **Vision:** `base64` image blocks → `input_image` payloads
- **Tool use:** `tool_use` → `function_call`, `tool_result` → `function_call_output`
- **Thinking:** intercepts `response.reasoning.delta` SSE frames, wraps in `<thinking>` events
- **Token tracking:** binds `usage.input_tokens` / `output_tokens` to CLI metrics display
- **Cache stripping:** removes Anthropic-only `cache_control` annotations

**Codex models available:**
```
gpt-5.2-codex  (default — "Frontier agentic coding")
gpt-5.1-codex
gpt-5.1-codex-mini
gpt-5.1-codex-max
gpt-5.4
gpt-5.2
```

### Modified: `src/services/api/client.ts` lines 308–320
```typescript
if (isCodexSubscriber()) {
  const codexTokens = getCodexOAuthTokens()
  if (codexTokens?.accessToken) {
    const codexFetch = createCodexFetch(codexTokens.accessToken)
    return new Anthropic({
      apiKey: 'codex-placeholder',
      ...ARGS,
      fetch: codexFetch as unknown as typeof globalThis.fetch,
    })
  }
}
```

### Modified: `src/utils/auth.ts`
New Codex auth helpers (lines 1314–1362, 1629–1638):
```typescript
export function saveCodexOAuthTokens(tokens: CodexTokens): void { ... }
export function getCodexOAuthTokens(): CodexTokens | null { ... }
export function clearCodexOAuthTokens(): void { ... }
export function isCodexSubscriber(): boolean {
  if (getAPIProvider() !== 'openai') return false
  return !!getCodexOAuthTokens()?.accessToken
}
```

### New: `src/services/oauth/client.ts` lines 46–72
```typescript
export function buildOpenAIAuthUrl({ codeChallenge, state, port, isManual }): string { ... }
```
Original only has `buildAuthUrl()` for Anthropic OAuth.

---

## 6. Feature Flags — 54 Experimental Unlocked

### New: `scripts/build.ts`
Full feature-flag bundler. Defines 35 experimental flags (plus `VOICE_MODE` as default):

```
AGENT_MEMORY_SNAPSHOT, AGENT_TRIGGERS, AGENT_TRIGGERS_REMOTE,
AWAY_SUMMARY, BASH_CLASSIFIER, BRIDGE_MODE,
BUILTIN_EXPLORE_PLAN_AGENTS, CACHED_MICROCOMPACT,
CCR_AUTO_CONNECT, CCR_MIRROR, CCR_REMOTE_SETUP,
COMPACTION_REMINDERS, CONNECTOR_TEXT, EXTRACT_MEMORIES,
HISTORY_PICKER, HOOK_PROMPTS, KAIROS_BRIEF, KAIROS_CHANNELS,
LODESTONE, MCP_RICH_OUTPUT, MESSAGE_ACTIONS,
NATIVE_CLIPBOARD_IMAGE, NEW_INIT, POWERSHELL_AUTO_MODE,
PROMPT_CACHE_BREAK_DETECTION, QUICK_SEARCH, SHOT_STATS,
TEAMMEM, TOKEN_BUDGET, TREE_SITTER_BASH,
TREE_SITTER_BASH_SHADOW, ULTRAPLAN, ULTRATHINK,
UNATTENDED_RETRY, VERIFICATION_AGENT, VOICE_MODE (default)
```

**Build variants:**

| Command | Output | Flags |
|---------|--------|-------|
| `bun run build` | `./cli` | VOICE_MODE only |
| `bun run build:dev` | `./cli-dev` | VOICE_MODE + dev stamp |
| `bun run build:dev:full` | `./cli-dev` | All 54 working flags |

**Build macros injected:**
```typescript
process.env.USER_TYPE = 'external'
process.env.CLAUDE_CODE_FORCE_FULL_LOGO = 'true'
process.env.CLAUDE_CODE_VERIFY_PLAN = 'false'
process.env.CCR_FORCE_BUNDLE = 'true'
MACRO.PACKAGE_URL = 'claude-code-source-snapshot'
MACRO.FEEDBACK_CHANNEL = 'github'
MACRO.ISSUES_EXPLAINER = "This reconstructed source snapshot does not include Anthropic internal issue routing."
```

---

## 7. Ultraplan Command — Always Enabled

### `src/commands/ultraplan.tsx` line 466

**Original:**
```typescript
isEnabled: () => "external" === 'ant',  // Always false for external builds
```

**Free:**
```typescript
isEnabled: () => true,
```

Ultraplan was strictly internal (ANT-only). Free unconditionally enables it.

**New UI components (Free only):**
- `src/components/UltraplanChoiceDialog.tsx`
- `src/components/UltraplanLaunchDialog.tsx`
- `src/utils/ultraplan/prompt.txt` (actual ultraplan system prompt text)

---

## 8. GrowthBook Key Gating — Relaxed

### `src/constants/keys.ts` lines 6–17

**Original:**
```typescript
const useExperimentalClientKey =
  isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_BUILD) ||
  (process.env.USER_TYPE === 'ant' && isEnvTruthy(process.env.ENABLE_GROWTHBOOK_DEV))

if (useExperimentalClientKey) {
  return 'sdk-yZQvlplybuXjYh6L'  // Internal experimental key
}
return process.env.GROWTHBOOK_CLIENT_KEY ?? (process.env.IS_DEV
  ? 'sdk-xRVcrliHIlrg4og4'
  : 'sdk-...')
```

**Free:**
```typescript
return process.env.GROWTHBOOK_CLIENT_KEY ?? (process.env.IS_DEV
  ? isEnvTruthy(process.env.ENABLE_GROWTHBOOK_DEV)
    ? 'sdk-yZQvlplybuXjYh6L'
    : 'sdk-xRVcrliHIlrg4og4'
  : 'sdk-...')
```

`USER_TYPE === 'ant'` check removed. Any dev environment can use the experimental key with `ENABLE_GROWTHBOOK_DEV=1`.

---

## 9. Analytics Initialization — Dead Code Eliminated

### `src/entrypoints/init.ts`

Removed from free:
- Import of `isBetaTracingEnabled` (betaSessionTracing)
- Import of `getTelemetryAttributes`
- 1P event logging initialization block (~12 lines)
- `applyConfigEnvironmentVariables` (replaced with `applySafeConfigEnvironmentVariables` only)
- `doInitializeTelemetry()` with flag to prevent double-init
- `setMeterState()` that imported and initialized OTEL meter

---

## 10. OpenTelemetry SDK — Fully Stripped from Deps

### `package.json` — Packages removed in Free

**Removed entirely:**
```
@opentelemetry/api
@opentelemetry/api-logs
@opentelemetry/core
@opentelemetry/exporter-logs-otlp-grpc
@opentelemetry/exporter-logs-otlp-http
@opentelemetry/exporter-logs-otlp-proto
@opentelemetry/exporter-metrics-otlp-grpc
@opentelemetry/exporter-metrics-otlp-http
@opentelemetry/exporter-metrics-otlp-proto
@opentelemetry/exporter-prometheus
@opentelemetry/exporter-trace-otlp-grpc
@opentelemetry/exporter-trace-otlp-http
@opentelemetry/exporter-trace-otlp-proto
@opentelemetry/resources
@opentelemetry/sdk-logs
@opentelemetry/sdk-metrics
@opentelemetry/sdk-trace-base
@opentelemetry/semantic-conventions
@growthbook/growthbook
```

**Added in Free (not in original):**
```
xxhash-wasm   (for CCH signing)
```

---

## 11. Session Tracing — All Spans Are No-ops

See section 1. Summary of what was removed from `sessionTracing.ts`:

| Original capability | Lines | Free status |
|---------------------|-------|-------------|
| Interaction root span (startInteractionSpan) | ~59 | No-op |
| LLM request span with TTFT tracking | ~177 | No-op |
| Tool execution span | ~105 | No-op |
| Permission dialog span | ~50 | No-op |
| Hook execution span | ~40 | No-op |
| Tool I/O content events | ~30 | No-op |
| AsyncLocalStorage span context | ~20 | Removed |
| Span cleanup interval | ~15 | Removed |
| isEnhancedTelemetryEnabled() GrowthBook gate | ~17 | Returns false |
| **Total gutted** | **~780** | |

---

## 12. Build System & Macro Injection

### `src/entrypoints/cli.tsx` lines 3–10 (Free only)

Original relies on Bun build-time `MACRO` injection. Free adds a runtime fallback guard:

```typescript
if (typeof MACRO === 'undefined') {
  (globalThis as any).MACRO = {
    VERSION: '2.1.87-dev',
    BUILD_TIME: new Date().toISOString(),
    PACKAGE_URL: 'claude-code-source-snapshot',
    FEEDBACK_CHANNEL: 'github',
  };
}
```

---

## 13. New Files Unique to Free-Code

### Core infrastructure
| File | Purpose |
|------|---------|
| `src/utils/cch.ts` | JavaScript xxHash64 CCH signing |
| `src/services/api/codex-fetch-adapter.ts` | 812-line Anthropic→OpenAI schema adapter |
| `src/services/oauth/codex-client.ts` | OpenAI PKCE OAuth 2.0 client |
| `src/constants/codex-oauth.ts` | OpenAI OAuth constants |
| `scripts/build.ts` | Feature-flag bundler |
| `env.d.ts` | MACRO global type declaration |
| `install.sh` | One-liner installer |

### New UI & commands
| File | Purpose |
|------|---------|
| `src/components/UltraplanChoiceDialog.tsx` | Ultraplan selection dialog |
| `src/components/UltraplanLaunchDialog.tsx` | Ultraplan launch dialog |
| `src/components/agents/SnapshotUpdateDialog.tsx` | Agent memory snapshot UI |
| `src/commands/assistant/assistant.tsx` | Assistant command |
| `src/assistant/AssistantSessionChooser.tsx` | Session chooser |
| `src/utils/ultraplan/prompt.txt` | Ultraplan system prompt |

### New services
| File | Purpose |
|------|---------|
| `src/services/compact/cachedMCConfig.ts` | Microcompact config caching |
| `src/services/compact/cachedMicrocompact.ts` | Microcompact state caching |
| `src/services/compact/snipCompact.ts` | Snippet-based compaction |
| `src/services/compact/snipProjection.ts` | Snippet projection |
| `src/services/contextCollapse/index.ts` | Context collapse orchestration |
| `src/services/contextCollapse/operations.ts` | Collapse operations |
| `src/services/contextCollapse/persist.ts` | Collapse persistence |

### SDK type definitions
| File | Purpose |
|------|---------|
| `src/entrypoints/sdk/coreTypes.generated.ts` | Internal core types |
| `src/entrypoints/sdk/runtimeTypes.ts` | Runtime types |
| `src/entrypoints/sdk/toolTypes.ts` | Tool types |

### Dev tooling
| File | Purpose |
|------|---------|
| `src/ink/devtools.ts` | Ink debugging utilities |
| `src/ink/global.d.ts` | Ink global type declarations |

---

## 14. New Services — Context Collapse & Microcompact Caching

**Context Collapse** (`src/services/contextCollapse/`) — Handles automatic reduction of large conversations to stay within context limits. Three layers: orchestration, operations (actual truncation logic), and persistence.

**Microcompact Caching** (`src/services/compact/`) — Caches compaction state through query/API flows so repeated compaction doesn't re-process. Backed by `CACHED_MICROCOMPACT` feature flag.

---

## 15. Internal-Only Stub Tools Exposed

Free includes stub implementations of Anthropic-internal tools with `isEnabled: () => false`:

### `src/tools/TungstenTool/TungstenTool.ts`
```typescript
isEnabled: () => false
// "Tungsten is only available in Anthropic internal builds"
```
Live monitoring/tracing tool (internal observability).

### `src/tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.ts`
```typescript
isEnabled: () => false
// "Plan execution verification is unavailable in this reconstructed build"
```

### `src/tools/WorkflowTool/constants.ts`
Constants only, no implementation.

---

## 16. Provider Environment Variables Expanded

Free documents and wires up 5 API providers (original: 1+):

| Provider | Activation | Notes |
|----------|-----------|-------|
| Anthropic (direct) | Default | `ANTHROPIC_API_KEY` |
| OpenAI Codex | `CLAUDE_CODE_USE_OPENAI=1` | Full Codex adapter + OAuth |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK=1` | + `AWS_REGION` |
| Google Vertex AI | `CLAUDE_CODE_USE_VERTEX=1` | |
| Anthropic Foundry | `CLAUDE_CODE_USE_FOUNDRY=1` | + `ANTHROPIC_FOUNDRY_API_KEY` |

**Model override env vars (Free documents explicitly):**
```
ANTHROPIC_MODEL
ANTHROPIC_BASE_URL
ANTHROPIC_DEFAULT_OPUS_MODEL
ANTHROPIC_DEFAULT_SONNET_MODEL
ANTHROPIC_DEFAULT_HAIKU_MODEL
```

---

## Summary Matrix

| Area | Original | Free |
|------|----------|------|
| Telemetry (OTEL) | Full Datadog/BigQuery/OTLP pipeline | All stubs → no-ops |
| Analytics events | Queued → Datadog sink | Dead code |
| CCH signing | Bun native `Attestation.zig` | JavaScript xxHash64 |
| NATIVE_CLIENT_ATTESTATION flag | Feature-gated | Removed, always on |
| Cyber risk guardrail | Full instruction string | Empty string |
| Ultraplan | ANT internal only | Always enabled |
| GrowthBook experimental key | Requires `USER_TYPE=ant` | Removed requirement |
| OpenAI Codex provider | None | Full (OAuth + 812-line adapter) |
| Feature flags | ~88 undocumented | 54 documented & unlockable |
| Build system | External | `scripts/build.ts` with feature presets |
| OTEL npm packages | 19 packages | All removed |
| New service subsystems | None | Context collapse, microcompact cache |
| Internal stubs | Hidden | Exposed (disabled) |
