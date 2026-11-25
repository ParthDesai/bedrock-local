# bedrock-local

> **Status: Work in Progress (WIP)** — Core wire types, trait interfaces, and the `v0.1` `Converse` fixture server are currently under active development.

A lightweight local emulator and chaos-testing proxy for **Amazon Bedrock** (`bedrock-runtime`), written in Rust.

## Why

When building agents and backend services on Amazon Bedrock (`Converse`, `ConverseStream`, `InvokeModel`, tool use, and guardrails), local development and CI pipelines usually run into three problems:

1. **No free official local runtime:** LocalStack gates Bedrock behind paid tiers or only mocks basic happy-path calls.
2. **Non-deterministic CI tests:** Multi-turn tool-calling agents (e.g., MCP servers and governance workflows) need deterministic responses to test state transitions and error recovery reliably.
3. **Hard-to-test AWS failure modes:** It is difficult to reproduce Bedrock-specific operational failures locally—such as `ThrottlingException` (RPM/TPM token-bucket exhaustion), `ModelTimeoutException`, cross-region inference profile (`us.*`, `eu.*`) failovers, or AWS `application/vnd.amazon.eventstream` mid-stream disconnects.

`bedrock-local` runs as a single local binary (`http://localhost:8080`). You point any official AWS SDK (`boto3`, `@aws-sdk/client-bedrock-runtime`, `aws-sdk-go-v2`, `aws-sdk-bedrockruntime`) or the AWS CLI at it using `--endpoint-url` with zero application code changes.

## Goals

- **Wire-compatible with official AWS SDKs:** Match Smithy REST-JSON1 request/response envelopes, `x-amzn-requestid` correlation headers, `x-amzn-errortype` error dispatch, and AWS EventStream (`application/vnd.amazon.eventstream`) binary CRC32 framing for streaming APIs.
- **Deterministic multi-turn tool-use fixtures:** Define YAML rules that match conversation history or `toolResult` blocks and return scripted `toolUse` or text responses for fast, offline E2E agent tests.
- **Chaos and quota simulation:** Model per-model RPM and TPM token buckets, latency jitter, and fault injection to test how client retry policies (`AdaptiveRetryMode`, `amz-sdk-invocation-id`) behave under pressure.
- **Pluggable backends:** Route requests through trait-based backends—starting with deterministic YAML fixtures, and expanding to local LLM engines (Ollama / vLLM) and record/replay cassettes against real AWS Bedrock.
- **AWS internals fidelity:** Incrementally implement SigV4 (`AWS4-HMAC-SHA256`) request verification, mock IAM ARN policy evaluation, and local Guardrail filters.

## Project Structure

The repository is organized as a modular Cargo workspace so each protocol layer can be implemented and tested independently:

```text
bedrock-local/
├── Cargo.toml
└── crates/
    ├── bedrock-types/      # Smithy REST-JSON1 wire structs, enums, and AWS error mappings
    ├── bedrock-core/       # Core traits (InferenceBackend, ModelPayloadCodec, AuthVerifier, QuotaLimiter)
    ├── bedrock-backends/   # Backend implementations (YAML fixture matcher, local LLM bridge)
    ├── bedrock-server/     # Axum HTTP routes, request context extraction, and Tower middleware
    └── bedrock-cli/        # CLI binary entrypoint
```

## Roadmap (Incremental Milestones)

- [ ] **v0.1 (In Progress):** `POST /model/{modelId}/converse` endpoint, Smithy union serialization (`ContentBlock`, `ToolConfiguration`), AWS error headers (`x-amzn-errortype`, `x-amzn-requestid`), and `FixtureBackend` for multi-turn tool-use testing.
- [ ] **v0.2:** Binary streaming support (`POST /model/{modelId}/converse-stream`) implementing `application/vnd.amazon.eventstream` framing, plus `InvokeModel` vendor payload codecs.
- [ ] **v0.3:** RPM/TPM token-bucket quota enforcement, chaos fault injection (`ThrottlingException`, timeouts, latency), and local Ollama/vLLM routing.
- [ ] **v0.4:** AWS SigV4 canonical signature verification and local IAM policy / ARN evaluation.
- [ ] **v0.5:** Local Bedrock Guardrails (`ApplyGuardrail`) and VCR record/replay mode.

## Usage (Planned CLI)

Start the local server with a fixture file:

```bash
cargo run -p bedrock-cli -- serve --port 8080 --fixtures ./fixtures/example.yaml
```

Point the standard AWS CLI or any AWS SDK to `http://localhost:8080`:

```bash
aws bedrock-runtime converse \
  --endpoint-url http://localhost:8080 \
  --region us-east-1 \
  --model-id us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --messages '[{"role":"user","content":[{"text":"diagnose vault policy for tx-991"}]}]'
```

## Development

```bash
# Type-check the workspace
cargo check

# Run unit and integration tests
cargo test
```

## License

MIT OR Apache-2.0
