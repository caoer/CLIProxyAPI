# Using Claude Code with OpenAI-Compatible Backends

This guide explains how to use [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with OpenAI-compatible backends through CLIProxyAPI. The proxy accepts Anthropic Messages API requests from Claude Code, translates them to OpenAI Chat Completions format, and routes them to any OpenAI-compatible backend.

## How It Works

CLIProxyAPI includes a bidirectional translation layer between the Anthropic Messages API (used by Claude Code) and the OpenAI Chat Completions API. When Claude Code sends a request:

```
Claude Code (Anthropic format)
    ↓  POST /v1/messages
CLIProxyAPI Proxy
    ↓  Translate request: Anthropic → OpenAI
OpenAI-Compatible Backend (e.g., OpenAI, OpenRouter, local LLM)
    ↓  Response in OpenAI format
CLIProxyAPI Proxy
    ↓  Translate response: OpenAI → Anthropic
Claude Code (receives Anthropic format)
```

The translation handles:
- Message format conversion (system, user, assistant roles)
- Streaming and non-streaming responses
- Tool/function calling
- Multimodal input (text and images)
- Thinking/reasoning mode (mapped between `thinking.budget_tokens` and `reasoning_effort`)
- Token counting
- Stop sequences

## Quick Start

### 1. Configure the Proxy

Create a `config.yaml` with your OpenAI-compatible backend:

```yaml
port: 8317

api-keys:
  - "my-proxy-key"

openai-compatibility:
  - name: "openai"
    base-url: "https://api.openai.com/v1"
    api-key-entries:
      - api-key: "sk-your-openai-key"
    models:
      - name: "gpt-4o"                        # Upstream OpenAI model
        alias: "claude-sonnet-4-20250514"      # Model name Claude Code sends
      - name: "gpt-4o"
        alias: "claude-sonnet-4-5-20250929"
      - name: "gpt-4o-mini"
        alias: "claude-haiku-4-5-20251001"
```

The `alias` field is what Claude Code requests, and `name` is the actual model sent to the upstream backend.

### 2. Start the Proxy

```bash
./cliproxy-api -config config.yaml
```

### 3. Configure Claude Code

Point Claude Code to the proxy by setting these environment variables:

```bash
export ANTHROPIC_BASE_URL=http://localhost:8317
export ANTHROPIC_API_KEY=my-proxy-key
```

Then run Claude Code as usual:

```bash
claude
```

## Configuration Examples

### OpenAI Backend

Use OpenAI GPT models through Claude Code:

```yaml
openai-compatibility:
  - name: "openai"
    base-url: "https://api.openai.com/v1"
    api-key-entries:
      - api-key: "sk-your-openai-key"
    models:
      - name: "gpt-4o"
        alias: "claude-sonnet-4-20250514"
      - name: "gpt-4o"
        alias: "claude-sonnet-4-5-20250929"
      - name: "o3"
        alias: "claude-opus-4-20250918"
      - name: "gpt-4o-mini"
        alias: "claude-haiku-4-5-20251001"
```

### OpenRouter Backend

Use any model available on OpenRouter:

```yaml
openai-compatibility:
  - name: "openrouter"
    base-url: "https://openrouter.ai/api/v1"
    api-key-entries:
      - api-key: "sk-or-v1-your-key"
    models:
      - name: "google/gemini-2.5-pro"
        alias: "claude-sonnet-4-20250514"
      - name: "openai/gpt-4o"
        alias: "claude-sonnet-4-5-20250929"
```

### Local LLM Server (e.g., Ollama, vLLM, LM Studio)

Use a locally-hosted model:

```yaml
openai-compatibility:
  - name: "local-llm"
    base-url: "http://localhost:11434/v1"   # Ollama with OpenAI compat
    api-key-entries:
      - api-key: "not-needed"              # Some local servers don't require keys
    models:
      - name: "llama3.1:70b"
        alias: "claude-sonnet-4-20250514"
```

### Multiple Backends with Load Balancing

Combine multiple backends for redundancy:

```yaml
openai-compatibility:
  - name: "openai-primary"
    base-url: "https://api.openai.com/v1"
    api-key-entries:
      - api-key: "sk-key-1"
      - api-key: "sk-key-2"               # Round-robin between keys
    models:
      - name: "gpt-4o"
        alias: "claude-sonnet-4-20250514"

  - name: "openrouter-fallback"
    base-url: "https://openrouter.ai/api/v1"
    api-key-entries:
      - api-key: "sk-or-v1-your-key"
    models:
      - name: "openai/gpt-4o"
        alias: "claude-sonnet-4-20250514"  # Same alias = automatic fallback
```

## Feature Translation Reference

The following table shows how features are translated between formats:

| Claude Code (Anthropic) | OpenAI Chat Completions | Notes |
|---|---|---|
| `messages[].role: "user"` | `messages[].role: "user"` | Direct mapping |
| `messages[].role: "assistant"` | `messages[].role: "assistant"` | Direct mapping |
| `system` (top-level) | `messages[0].role: "system"` | Extracted to system message |
| `max_tokens` | `max_tokens` | Direct mapping |
| `temperature` | `temperature` | Direct mapping |
| `top_p` | `top_p` | Direct mapping |
| `stop_sequences` | `stop` | Array to string/array |
| `stream: true` | `stream: true` | SSE streaming supported |
| `tools[].input_schema` | `tools[].function.parameters` | Schema preserved |
| `tool_choice.type: "auto"` | `tool_choice: "auto"` | |
| `tool_choice.type: "any"` | `tool_choice: "required"` | |
| `tool_choice.type: "tool"` | `tool_choice: {type: "function", ...}` | |
| `thinking.type: "enabled"` | `reasoning_effort` | Budget mapped to effort level |
| Content block `type: "thinking"` | `reasoning_content` | Assistant messages only |
| Content block `type: "tool_use"` | `tool_calls[]` | ID and args preserved |
| Content block `type: "tool_result"` | `role: "tool"` message | Follows tool call |
| Image `source.type: "base64"` | `image_url` with data URI | Media type preserved |

## Troubleshooting

### Claude Code can't connect

Verify the proxy is running and the base URL is correct:

```bash
curl http://localhost:8317/v1/models \
  -H "Authorization: Bearer my-proxy-key"
```

### Model not found

Ensure your `models` configuration has an `alias` that matches the model name Claude Code requests. You can check what model Claude Code requests by enabling debug logging:

```yaml
debug: true
```

### Streaming issues

If responses appear slow or chunked incorrectly, check that your upstream backend supports SSE streaming. The proxy translates between Anthropic SSE events and OpenAI SSE events automatically.

### Tool calling not working

Tool/function calling translation is fully supported. If a specific tool fails, enable debug logging to inspect the translated request payload.

## Related Documentation

- [Configuration Reference](../config.example.yaml) — Full configuration options
- [SDK Advanced Guide](sdk-advanced.md) — Custom providers and translators
- [SDK Usage Guide](sdk-usage.md) — Embedding the proxy as a Go library
