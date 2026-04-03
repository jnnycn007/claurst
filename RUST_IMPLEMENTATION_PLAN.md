# Rust Implementation Plan — Porting Free-Code Diffs to src-rust

> Based on: DIFF_ANALYSIS.md audit (2026-04-03)
> Target codebase: `src-rust/`
> Rust crate layout: `crates/{api, bridge, buddy, cli, commands, core, mcp, plugins, query, tools, tui}`

Each section is self-contained. Work top-to-bottom or in dependency order. Each item includes: **what to do**, **exact files to touch**, and **the specific code/logic to implement**.

---

## Priority Legend

- **P0** — Architectural correctness / removes active telemetry calls that phone home
- **P1** — Security/guardrail changes (empty cyber instruction, CCH signing)
- **P2** — New features (Codex provider, experimental flags)
- **P3** — Polish / nice-to-have (new services, build tooling)

---

## Item 1 — Remove / Stub All Analytics `logEvent` Calls [P0]

### Current state
`crates/core/src/analytics.rs` currently only contains `SessionMetrics` (local counters). It does NOT have an outbound analytics pipeline yet. Verify there is no outbound HTTP call anywhere:

```bash
grep -r "logEvent\|logEventAsync\|attachAnalyticsSink\|datadog\|bigquery" src-rust/
```

If any are found, replace them with no-ops matching the free-code pattern.

### What to implement

Add a public `log_event` no-op function so future call sites compile cleanly:

**File:** `crates/core/src/analytics.rs` — append:

```rust
/// No-op analytics stub. The OSS/free build intentionally ships without
/// product telemetry. All call sites compile unchanged; all data is discarded.
pub fn log_event(_event_name: &str, _metadata: &[(&str, &str)]) {}

pub async fn log_event_async(_event_name: &str, _metadata: &[(&str, &str)]) {}
```

**Do NOT implement:**
- Any HTTP sink
- Any queueing
- Any file logging to Anthropic-controlled endpoints

---

## Item 2 — Strip OpenTelemetry from Cargo.toml [P0]

### Current state
Check if OTEL crates are present:

```bash
grep -r "opentelemetry\|otel\|tracing-opentelemetry" src-rust/Cargo.toml src-rust/crates/*/Cargo.toml
```

### What to implement

If any OTEL exporter crates are listed (e.g. `opentelemetry-otlp`, `opentelemetry-sdk`, `opentelemetry-proto`), remove them from all `Cargo.toml` files.

The `tracing` crate is fine — it is a local diagnostic tool, not an outbound exporter. Keep it. Only remove anything that exports spans/metrics to a remote endpoint.

**Files to edit:** Any `Cargo.toml` containing `opentelemetry*` dependencies.

**Pattern to remove:**
```toml
# Remove these if present:
opentelemetry = { ... }
opentelemetry-otlp = { ... }
opentelemetry-sdk = { ... }
opentelemetry-proto = { ... }
tracing-opentelemetry = { ... }
```

---

## Item 3 — Wipe Cyber Risk Guardrail Instruction [P1]

### Current state

Check `crates/core/src/system_prompt.rs` and `crates/cli/src/main.rs` and `src-rust/crates/cli/src/system_prompt.txt`:

```bash
grep -r "CYBER_RISK\|cyber_risk\|authorized security\|CTF\|guardrail" src-rust/
```

The TypeScript original has this string in `src/constants/cyberRiskInstruction.ts`. In the Rust port it likely lives in `system_prompt.rs` or is hardcoded inline.

### What to implement

**File:** `crates/core/src/system_prompt.rs` (or wherever the cyber risk string lives)

Find and replace:
```rust
// Before (if present in any form):
pub const CYBER_RISK_INSTRUCTION: &str = "IMPORTANT: Assist with authorized security testing...";

// After (free-code pattern — empty string):
pub const CYBER_RISK_INSTRUCTION: &str = "";
```

If the instruction is assembled as part of `build_system_prompt()` or similar, find the section that appends it and either remove the append call or make it append an empty section.

**Specific action:** Search for the text `"authorized security testing"` or `"CTF"` across all `.rs` files. Remove or empty out the string at its definition point. Do not search-and-replace at every call site — fix the source constant.

---

## Item 4 — Implement JavaScript-Style CCH Signing (xxHash64) [P1]

### Current state

The Rust API client in `crates/api/src/lib.rs` builds and sends POST /v1/messages. Check if the `x-anthropic-billing-header` includes `cch=`:

```bash
grep -r "cch\|billing\|attestation\|xxhash" src-rust/
```

### What to implement

The Rust port should always include the `cch=00000` placeholder in the billing header and compute the xxHash64 hash before the request body is sent (same as free-code's `src/utils/cch.ts`).

**Step 1 — Add xxhash to Cargo.toml:**

**File:** `crates/api/Cargo.toml`
```toml
[dependencies]
xxhash-rust = { version = "0.8", features = ["xxh64"] }
```

**Step 2 — Create `crates/api/src/cch.rs`:**

```rust
//! CCH (Client-Computed Hash) request signing.
//!
//! Mirrors free-code's src/utils/cch.ts: computes an xxHash64 fingerprint of
//! the serialised request body and embeds it in the x-anthropic-billing-header.
//! The server uses the hash to verify the request originated from a legitimate
//! Claude Code client and to gate features like fast-mode.

use xxhash_rust::xxh64::xxh64;

const CCH_SEED: u64 = 0x6E52_736A_C806_831E;
const CCH_MASK: u64 = 0xF_FFFF;   // 5 hex digits
const CCH_PLACEHOLDER: &str = "cch=00000";

/// Compute the 5-hex-digit CCH hash for `body`.
pub fn compute_cch(body: &[u8]) -> String {
    let hash = xxh64(body, CCH_SEED) & CCH_MASK;
    format!("cch={hash:05x}")
}

/// Return true if `header` contains the placeholder that should be replaced.
pub fn has_cch_placeholder(s: &str) -> bool {
    s.contains(CCH_PLACEHOLDER)
}

/// Replace the placeholder in `s` with the computed hash.
pub fn replace_cch_placeholder(s: &str, hash: &str) -> String {
    s.replacen(CCH_PLACEHOLDER, hash, 1)
}
```

**Step 3 — Wire into the billing header:**

**File:** `crates/api/src/lib.rs` (or wherever the `x-anthropic-billing-header` is built in `AnthropicClient`)

Find the code that builds or sends POST `/v1/messages`. Before serialising the request body:

```rust
use crate::cch::{compute_cch, has_cch_placeholder, replace_cch_placeholder};

// After serialising body to JSON string:
let mut body_str = serde_json::to_string(&request)?;

// Replace placeholder with computed hash (no-op if placeholder not in body)
if has_cch_placeholder(&billing_header) {
    let hash = compute_cch(body_str.as_bytes());
    billing_header = replace_cch_placeholder(&billing_header, &hash);
}
```

**Step 4 — Always include placeholder in billing header:**

Find where `x-anthropic-billing-header` is assembled. Make `cch=00000` unconditional (remove any feature-flag check):

```rust
// Before (if gated):
let cch_part = if native_attestation_enabled() { " cch=00000;" } else { "" };

// After (free-code pattern — always include):
let cch_part = " cch=00000;";
```

The full header format: `cc_version={version}; cc_entrypoint={entrypoint}; cch=00000; cc_workload={workload};`

---

## Item 5 — Remove GrowthBook `USER_TYPE=ant` Gate [P1]

### Current state

```bash
grep -r "USER_TYPE\|user_type\|growthbook\|experimental_key" src-rust/
```

### What to implement

**File:** `crates/core/src/feature_flags.rs` (the `FeatureFlagManager`)

In the current implementation the GrowthBook client key is fetched from `GROWTHBOOK_API_KEY`. If there is any logic that checks `USER_TYPE == "ant"` before switching to an experimental key, remove that check.

```rust
// Remove any block like:
if std::env::var("USER_TYPE").as_deref() == Ok("ant") && experimental_build() {
    return "sdk-yZQvlplybuXjYh6L";
}

// Keep the env-var override path only:
let key = std::env::var("GROWTHBOOK_CLIENT_KEY")
    .unwrap_or_else(|_| "sdk-xRVcrliHIlrg4og4".to_string());
```

---

## Item 6 — Unconditionally Enable Ultraplan [P2]

### Current state

```bash
grep -r "ultraplan\|ULTRAPLAN\|isEnabled.*ant" src-rust/crates/commands/
```

### What to implement

**File:** `crates/commands/src/` — wherever the Ultraplan command is defined.

Find any `is_enabled()` check that gates on `USER_TYPE == "ant"` or a build feature flag:

```rust
// Before:
fn is_enabled(&self) -> bool {
    std::env::var("USER_TYPE").as_deref() == Ok("ant")
}

// After (free-code pattern):
fn is_enabled(&self) -> bool {
    true
}
```

---

## Item 7 — OpenAI Codex Provider Support [P2]

This is the largest new subsystem. Implement in stages.

### Stage A — Codex OAuth Constants

**New file:** `crates/core/src/codex_oauth.rs`

```rust
pub const CODEX_CLIENT_ID: &str = "app_EMoamEEZ73f0CkXaXp7hrann";
pub const CODEX_AUTHORIZE_URL: &str = "https://auth.openai.com/oauth/authorize";
pub const CODEX_TOKEN_URL: &str = "https://auth.openai.com/oauth/token";
pub const CODEX_REDIRECT_URI: &str = "http://localhost:1455/auth/callback";
pub const CODEX_SCOPES: &str = "openid profile email offline_access";

pub const CODEX_MODELS: &[(&str, &str)] = &[
    ("gpt-5.2-codex",      "GPT-5.2 Codex (default)"),
    ("gpt-5.1-codex",      "GPT-5.1 Codex"),
    ("gpt-5.1-codex-mini", "GPT-5.1 Codex Mini"),
    ("gpt-5.1-codex-max",  "GPT-5.1 Codex Max"),
    ("gpt-5.4",            "GPT-5.4"),
    ("gpt-5.2",            "GPT-5.2"),
];
pub const DEFAULT_CODEX_MODEL: &str = "gpt-5.2-codex";
```

Export from `crates/core/src/lib.rs`: `pub mod codex_oauth;`

### Stage B — Codex Token Storage

**File:** `crates/core/src/oauth_config.rs` (or equivalent auth storage)

Add a `CodexTokens` struct and persist/load functions:

```rust
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct CodexTokens {
    pub access_token: String,
    pub refresh_token: Option<String>,
    pub account_id: Option<String>,
    pub expires_at: Option<u64>,  // Unix timestamp
}

/// Save Codex OAuth tokens to ~/.claude/codex_tokens.json
pub fn save_codex_tokens(tokens: &CodexTokens) -> anyhow::Result<()> { ... }

/// Load Codex OAuth tokens from ~/.claude/codex_tokens.json
pub fn get_codex_tokens() -> Option<CodexTokens> { ... }

/// Clear stored Codex tokens
pub fn clear_codex_tokens() -> anyhow::Result<()> { ... }

/// Returns true if the user has a valid Codex access token AND
/// CLAUDE_CODE_USE_OPENAI=1 is set.
pub fn is_codex_subscriber() -> bool {
    if std::env::var("CLAUDE_CODE_USE_OPENAI").as_deref() != Ok("1") {
        return false;
    }
    get_codex_tokens().map(|t| !t.access_token.is_empty()).unwrap_or(false)
}
```

### Stage C — Codex PKCE OAuth Flow

**New file:** `crates/cli/src/codex_oauth_flow.rs`

Implement the PKCE OAuth 2.0 flow from free-code's `codex-client.ts`:

```rust
use tokio::net::TcpListener;
use base64::{engine::general_purpose::URL_SAFE_NO_PAD, Engine};
use sha2::{Sha256, Digest};

/// Generate a PKCE code verifier (random 64-byte base64url string)
pub fn generate_code_verifier() -> String { ... }

/// Compute PKCE code challenge (SHA-256 of verifier, base64url encoded)
pub fn compute_code_challenge(verifier: &str) -> String {
    let hash = Sha256::digest(verifier.as_bytes());
    URL_SAFE_NO_PAD.encode(hash)
}

/// Build the OpenAI authorization URL
pub fn build_auth_url(code_challenge: &str, state: &str) -> String {
    format!(
        "{}?client_id={}&redirect_uri={}&response_type=code&scope={}&code_challenge={}&code_challenge_method=S256&state={}",
        CODEX_AUTHORIZE_URL,
        CODEX_CLIENT_ID,
        urlencoding::encode(CODEX_REDIRECT_URI),
        urlencoding::encode(CODEX_SCOPES),
        code_challenge,
        state,
    )
}

/// Start local HTTP server on port 1455, open browser, wait for callback,
/// exchange code for tokens, return CodexTokens.
pub async fn run_oauth_flow() -> anyhow::Result<CodexTokens> {
    let verifier = generate_code_verifier();
    let challenge = compute_code_challenge(&verifier);
    let state = generate_state();

    let auth_url = build_auth_url(&challenge, &state);

    // Open browser
    open::that(&auth_url)?;

    // Listen on port 1455 for redirect
    let listener = TcpListener::bind("127.0.0.1:1455").await?;
    let (code, returned_state) = wait_for_callback(listener).await?;

    if returned_state != state {
        anyhow::bail!("OAuth state mismatch");
    }

    // Exchange code for tokens
    exchange_code_for_tokens(&code, &verifier).await
}

async fn exchange_code_for_tokens(code: &str, verifier: &str) -> anyhow::Result<CodexTokens> {
    let client = reqwest::Client::new();
    let resp = client.post(CODEX_TOKEN_URL)
        .form(&[
            ("client_id", CODEX_CLIENT_ID),
            ("code", code),
            ("code_verifier", verifier),
            ("grant_type", "authorization_code"),
            ("redirect_uri", CODEX_REDIRECT_URI),
        ])
        .send()
        .await?
        .json::<serde_json::Value>()
        .await?;

    let access_token = resp["access_token"].as_str().unwrap_or("").to_string();
    let refresh_token = resp["refresh_token"].as_str().map(|s| s.to_string());
    let account_id = extract_account_id_from_jwt(&access_token);

    Ok(CodexTokens { access_token, refresh_token, account_id, expires_at: None })
}

/// Extract chatgpt-account-id from the JWT access token (middle segment, base64 decode, parse JSON)
fn extract_account_id_from_jwt(token: &str) -> Option<String> {
    let parts: Vec<&str> = token.splitn(3, '.').collect();
    let payload = URL_SAFE_NO_PAD.decode(parts.get(1)?).ok()?;
    let json: serde_json::Value = serde_json::from_slice(&payload).ok()?;
    json["https://api.openai.com/auth"]["account_id"]
        .as_str()
        .map(|s| s.to_string())
}
```

Add to `Cargo.toml` for `crates/cli`:
```toml
base64 = "0.22"
sha2 = "0.10"
open = "5"
urlencoding = "2"
```

### Stage D — Codex Schema Translation Layer

**New file:** `crates/api/src/codex_adapter.rs`

This is the most complex part — translating Anthropic Messages API ↔ OpenAI Responses API. Key mappings from free-code's `codex-fetch-adapter.ts`:

```rust
const CODEX_ENDPOINT: &str = "https://chatgpt.com/backend-api/codex/responses";

/// Translate an Anthropic CreateMessageRequest into an OpenAI Responses API body.
pub fn translate_request_to_codex(
    req: &CreateMessageRequest,
    account_id: &str,
) -> serde_json::Value {
    let input = translate_messages(&req.messages);
    let tools = req.tools.as_ref().map(translate_tools);

    serde_json::json!({
        "model": map_model_to_codex(&req.model),
        "input": input,
        "tools": tools,
        "stream": true,
        "reasoning": { "effort": "medium", "summary": "auto" },
    })
}

/// Map Claude model names to Codex equivalents
fn map_model_to_codex(model: &str) -> &str {
    match model {
        m if m.contains("opus") => "gpt-5.1-codex-max",
        m if m.contains("sonnet") => "gpt-5.2-codex",
        m if m.contains("haiku") => "gpt-5.1-codex-mini",
        _ => DEFAULT_CODEX_MODEL,
    }
}

/// Translate Anthropic messages array to OpenAI input array
fn translate_messages(messages: &[ApiMessage]) -> serde_json::Value {
    // tool_use -> function_call
    // tool_result -> function_call_output
    // base64 image -> input_image
    // Strip cache_control annotations
    // ... (implement per changes.md spec)
}

/// Translate Anthropic tool definitions to OpenAI function definitions
fn translate_tools(tools: &[ApiToolDefinition]) -> serde_json::Value {
    // Anthropic: { name, description, input_schema }
    // OpenAI:    { type: "function", name, description, parameters }
    // ...
}

/// Send a Codex request and translate the SSE stream back to Anthropic StreamEvents.
pub async fn send_codex_request(
    req: &CreateMessageRequest,
    tokens: &CodexTokens,
    tx: mpsc::Sender<StreamEvent>,
) -> anyhow::Result<()> {
    let body = translate_request_to_codex(req, tokens.account_id.as_deref().unwrap_or(""));

    let client = reqwest::Client::new();
    let mut response = client
        .post(CODEX_ENDPOINT)
        .bearer_auth(&tokens.access_token)
        .header("chatgpt-account-id", tokens.account_id.as_deref().unwrap_or(""))
        .header("originator", "pi")
        .header("OpenAI-Beta", "responses=experimental")
        .json(&body)
        .send()
        .await?;

    // Parse OpenAI SSE stream and translate back to Anthropic StreamEvents
    translate_codex_stream(&mut response, tx).await
}

/// Translate OpenAI SSE events → Anthropic StreamEvents.
/// Key: response.reasoning.delta → thinking_delta ContentBlock
async fn translate_codex_stream(
    response: &mut reqwest::Response,
    tx: mpsc::Sender<StreamEvent>,
) -> anyhow::Result<()> {
    // Parse SSE frames
    // Map:
    //   response.created                    → message_start
    //   response.output_item.added (text)   → content_block_start (text)
    //   response.output_text.delta          → content_block_delta (text_delta)
    //   response.reasoning.delta            → content_block_delta (thinking_delta)
    //   response.output_item.done           → content_block_stop
    //   response.completed                  → message_stop + usage
    //   response.failed                     → error
}
```

### Stage E — Provider Selection in AnthropicClient

**File:** `crates/api/src/lib.rs` — `AnthropicClient::new()` or equivalent factory

```rust
use cc_core::oauth_config::is_codex_subscriber;
use crate::codex_adapter::send_codex_request;

impl AnthropicClient {
    pub fn new() -> Self {
        // Check Codex first
        if is_codex_subscriber() {
            return Self::new_codex();
        }
        // ... existing Anthropic/Bedrock/Vertex logic
    }

    fn new_codex() -> Self {
        // Set internal flag to route create_message() through send_codex_request()
        Self { provider: Provider::Codex, ... }
    }
}

impl AnthropicClient {
    pub async fn create_message_stream(
        &self,
        req: CreateMessageRequest,
        tx: mpsc::Sender<StreamEvent>,
    ) -> anyhow::Result<()> {
        match self.provider {
            Provider::Codex => {
                let tokens = cc_core::oauth_config::get_codex_tokens()
                    .ok_or_else(|| anyhow::anyhow!("No Codex tokens"))?;
                send_codex_request(&req, &tokens, tx).await
            }
            _ => self.send_anthropic_request(req, tx).await,
        }
    }
}
```

### Stage F — `/codex-login` Command

**New file:** `crates/commands/src/codex_login.rs`

```rust
pub struct CodexLoginCommand;

impl Command for CodexLoginCommand {
    fn name(&self) -> &str { "codex-login" }
    fn description(&self) -> &str { "Authenticate with OpenAI Codex (ChatGPT Pro)" }
    fn is_enabled(&self) -> bool {
        // Enable when CLAUDE_CODE_USE_OPENAI=1
        std::env::var("CLAUDE_CODE_USE_OPENAI").as_deref() == Ok("1")
    }

    async fn run(&self, _args: &str, ctx: &CommandContext) -> anyhow::Result<()> {
        let tokens = crate::codex_oauth_flow::run_oauth_flow().await?;
        cc_core::oauth_config::save_codex_tokens(&tokens)?;
        ctx.print("Codex authentication successful.");
        Ok(())
    }
}
```

Register in command registry (`crates/commands/src/lib.rs`).

---

## Item 8 — Feature Flag System: Compile-Time Flags [P2]

### Current state

`crates/core/src/feature_flags.rs` has runtime GrowthBook fetching. The free-code build uses **compile-time** feature flags via Bun macros.

### What to implement

Add a Cargo feature flag for each key experimental feature. This lets you ship different binaries with different capability sets.

**File:** `crates/core/Cargo.toml` — add feature section:

```toml
[features]
default = ["voice_mode"]

# Interaction & UI
voice_mode = []
ultraplan = []
ultrathink = []
history_picker = []
token_budget = []
message_actions = []
quick_search = []
away_summary = []
hook_prompts = []

# Agents & Memory
agent_triggers = []
agent_triggers_remote = []
extract_memories = []
verification_agent = []
builtin_explore_plan_agents = []
cached_microcompact = []
compaction_reminders = []
agent_memory_snapshot = []
teammem = []

# Tools & Infrastructure
bash_classifier = []
bridge_mode = []
mcp_rich_output = []
connector_text = []
unattended_retry = []
new_init = []
powershell_auto_mode = []
shot_stats = []
tree_sitter_bash = []
prompt_cache_break_detection = []

# Full dev build — all of the above
dev_full = [
    "voice_mode", "ultraplan", "ultrathink", "history_picker",
    "token_budget", "message_actions", "quick_search", "away_summary",
    "hook_prompts", "agent_triggers", "agent_triggers_remote",
    "extract_memories", "verification_agent", "builtin_explore_plan_agents",
    "cached_microcompact", "compaction_reminders", "agent_memory_snapshot",
    "teammem", "bash_classifier", "bridge_mode", "mcp_rich_output",
    "connector_text", "unattended_retry", "new_init",
    "powershell_auto_mode", "shot_stats", "tree_sitter_bash",
    "prompt_cache_break_detection",
]
```

**Usage pattern in code:**

```rust
// crates/commands/src/ultraplan.rs
impl Command for UltraplanCommand {
    fn is_enabled(&self) -> bool {
        #[cfg(feature = "ultraplan")] { return true; }
        #[cfg(not(feature = "ultraplan"))] { return false; }
    }
}
```

**Build scripts analogous to free-code:**
```makefile
# Makefile or build script
build:
    cargo build --release

build-dev-full:
    cargo build --release --features dev_full

build-dev:
    cargo build --features voice_mode
```

---

## Item 9 — Telemetry Init No-op [P0]

### Current state

```bash
grep -r "initializeTelemetry\|flushTelemetry\|bootstrapTelemetry\|otel\|opentelemetry" src-rust/
```

### What to implement

If any Rust code calls out to an OTEL endpoint on startup, replace with no-ops.

**File:** `crates/core/src/analytics.rs` — add:

```rust
/// No-op. The Rust port does not initialize OpenTelemetry exporters.
pub fn initialize_telemetry() {}

/// No-op. Nothing to flush.
pub async fn flush_telemetry() {}

/// Always returns false. Enhanced telemetry is disabled.
pub fn is_enhanced_telemetry_enabled() -> bool { false }

/// Always returns false.
pub fn is_telemetry_enabled() -> bool {
    // Honour opt-in env var if present, but default to off
    std::env::var("CLAUDE_CODE_ENABLE_TELEMETRY")
        .as_deref()
        .unwrap_or("0") == "1"
}
```

---

## Item 10 — System Prompt: Remove Managed Security Overlay Fetching [P1]

### Current state

```bash
grep -r "managed_settings\|remote_settings\|security_overlay\|managedSettings" src-rust/
```

Free-code removes the fetching of Anthropic's server-pushed managed settings that inject security overlays into the system prompt.

### What to implement

**File:** `crates/core/src/remote_settings.rs`

If there is any HTTP call that fetches `https://api.anthropic.com/...` managed settings and injects additional system prompt sections, make that call a no-op or return an empty settings struct.

```rust
/// Stub: Returns empty managed settings.
/// The free/OSS build does not fetch server-pushed security overlays.
pub async fn fetch_remote_managed_settings() -> RemoteManagedSettings {
    RemoteManagedSettings::default()
}
```

**File:** `crates/core/src/system_prompt.rs`

In `build_system_prompt()`, find where managed settings sections are appended and make them conditional:

```rust
// Remove or guard with cfg flag:
// if let Some(security_section) = managed_settings.security_overlay {
//     sections.push(SystemPromptSection::cached("security", security_section));
// }
```

---

## Item 11 — Context Collapse Service [P3]

The free-code adds `src/services/contextCollapse/` with three files. Port the core logic to Rust.

### What to implement

**New file:** `crates/core/src/context_collapse.rs`

```rust
//! Context collapse — automatically reduces conversation size to fit within
//! the model's context window.

/// Strategy for collapsing a conversation.
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum CollapseStrategy {
    /// Drop oldest non-system messages first.
    DropOldest,
    /// Summarise the middle of the conversation.
    Summarise,
}

/// Collapse a message list to fit within `max_tokens` estimated tokens.
pub fn collapse_context(
    messages: Vec<cc_core::types::Message>,
    max_tokens: usize,
    strategy: CollapseStrategy,
) -> Vec<cc_core::types::Message> {
    // 1. Estimate token count (use simple word-count heuristic or tiktoken-rs)
    // 2. If under limit, return as-is
    // 3. Apply strategy until under limit
    todo!()
}

/// Persist collapse state to ~/.claude/context_collapse_state.json
pub fn save_collapse_state(session_id: &str, state: &CollapseState) -> anyhow::Result<()> { todo!() }
pub fn load_collapse_state(session_id: &str) -> Option<CollapseState> { todo!() }
```

Enable behind `#[cfg(feature = "cached_microcompact")]` where appropriate.

---

## Item 12 — Build Metadata Macros [P3]

The free-code injects `MACRO.VERSION`, `MACRO.BUILD_TIME`, `MACRO.PACKAGE_URL`, `MACRO.FEEDBACK_CHANNEL` at build time. In Rust this is done via `build.rs`.

### What to implement

**New file:** `crates/cli/build.rs`

```rust
fn main() {
    // Embed build time
    println!("cargo:rustc-env=BUILD_TIME={}", chrono::Utc::now().to_rfc3339());

    // Embed git commit hash
    let output = std::process::Command::new("git")
        .args(["rev-parse", "--short", "HEAD"])
        .output()
        .unwrap_or_default();
    let commit = String::from_utf8_lossy(&output.stdout).trim().to_string();
    println!("cargo:rustc-env=GIT_COMMIT={commit}");

    // Package URL
    println!("cargo:rustc-env=PACKAGE_URL=claude-code-source-snapshot");
    println!("cargo:rustc-env=FEEDBACK_CHANNEL=github");
    println!("cargo:rustc-env=ISSUES_EXPLAINER=This build does not include Anthropic internal issue routing.");

    println!("cargo:rerun-if-changed=.git/HEAD");
}
```

**Usage in binary:**
```rust
// crates/cli/src/main.rs
pub const BUILD_TIME: &str = env!("BUILD_TIME");
pub const GIT_COMMIT: &str = env!("GIT_COMMIT");
pub const PACKAGE_URL: &str = env!("PACKAGE_URL");
pub const FEEDBACK_CHANNEL: &str = env!("FEEDBACK_CHANNEL");
```

---

## Item 13 — Token Budget Tracking [P2]

Feature flag `TOKEN_BUDGET` in free-code. The core `SessionMetrics` already tracks tokens. The UI piece is what's gated.

### What to implement

**File:** `crates/tui/src/` — find the status line or header component.

Add a token budget display that shows `{used}/{budget}` tokens when:
1. `CLAUDE_CODE_TOKEN_BUDGET` env var is set to a number, OR
2. The model's default context window is known

```rust
#[cfg(feature = "token_budget")]
pub fn render_token_budget(metrics: &SessionMetrics, max_tokens: u64) -> String {
    let used = metrics.total_input_tokens.load(Ordering::Relaxed)
             + metrics.total_output_tokens.load(Ordering::Relaxed);
    let pct = (used as f64 / max_tokens as f64 * 100.0) as u64;
    format!("Tokens: {used}/{max_tokens} ({pct}%)")
}
```

---

## Implementation Order (Dependency Graph)

```
P0 — Do first (no external dependencies):
  Item 1  → Stub log_event in analytics.rs
  Item 2  → Strip OTEL from Cargo.toml
  Item 9  → Add telemetry no-op fns
  Item 10 → Remove managed settings injection

P1 — Do second (touches billing/signing):
  Item 3  → Empty CYBER_RISK_INSTRUCTION
  Item 4  → Implement CCH xxHash64 signing
  Item 5  → Remove USER_TYPE=ant GrowthBook gate

P2 — Feature work (can parallelize):
  Item 6  → Ultraplan always enabled
  Item 7A → Codex OAuth constants
  Item 7B → Codex token storage
  Item 7C → Codex PKCE OAuth flow
  Item 7D → Codex schema adapter
  Item 7E → Provider selection in AnthropicClient
  Item 7F → /codex-login command
  Item 8  → Cargo feature flags for 35 experimental features
  Item 13 → Token budget display

P3 — Polish:
  Item 11 → Context collapse service
  Item 12 → Build metadata via build.rs
```

---

## Verification Checklist

After implementing each item, verify:

- [ ] `cargo build` passes with zero OTEL-related imports remaining
- [ ] `grep -r "opentelemetry\|logEvent.*datadog\|bigquery" src-rust/` returns nothing
- [ ] `CYBER_RISK_INSTRUCTION` constant is empty string
- [ ] CCH signing test: `compute_cch(b"test") == "cch=XXXXX"` (compare to TypeScript implementation)
- [ ] `CLAUDE_CODE_USE_OPENAI=1` + valid Codex token → requests route to `chatgpt.com/backend-api/codex/responses`
- [ ] `cargo build --features dev_full` compiles with all experimental flags
- [ ] `/ultraplan` command responds without ANT-check failure
- [ ] `x-anthropic-billing-header` always includes `cch=` (non-zero value after signing)
