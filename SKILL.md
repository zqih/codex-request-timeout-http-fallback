---
name: codex-http-fallback
description: Configure Codex to use an HTTP/SSE OpenAI provider instead of WebSocket streaming. Use when Codex should directly add or verify the openai-http provider module in config.toml with wire_api, supports_websockets, requires_openai_auth, and model_provider settings to downgrade WebSocket transport to HTTP/SSE.
---

# Codex HTTP Fallback

## Overview

Use this skill to directly configure Codex to avoid WebSocket streaming by adding a custom `openai-http` provider and making it the default provider. Keep the workflow focused on backing up `config.toml`, writing the provider module, validating the shape, and restarting Codex.

## Configuration

Add this top-level provider selection:

```toml
model_provider = "openai-http"
```

Add this provider module:

```toml
[model_providers.openai-http]
name = "OpenAI HTTP"
base_url = "https://chatgpt.com/backend-api/codex"
wire_api = "responses"
supports_websockets = false
requires_openai_auth = true
```

## Apply Safely

Before editing the real config, back it up.

```bash
cp ~/.codex/config.toml ~/.codex/config.toml.bak-$(date +%Y%m%d-%H%M%S)
```

Then edit `~/.codex/config.toml` so it contains one `model_provider = "openai-http"` setting and one `[model_providers.openai-http]` block. If an older `openai-http` block already exists, update it in place instead of adding a duplicate.

Recommended placement:

1. Put `model_provider = "openai-http"` near other top-level settings such as `service_tier`.
2. Put `[model_providers.openai-http]` before marketplace and project tables.
3. Keep project trust settings unchanged.

## Validate

After editing, inspect the first part of the config:

```bash
sed -n '1,80p' ~/.codex/config.toml
```

Confirm these exact keys are present:

- `model_provider = "openai-http"`
- `[model_providers.openai-http]`
- `wire_api = "responses"`
- `supports_websockets = false`
- `requires_openai_auth = true`

Run Codex strict config validation if the installed CLI exposes it. If validation requires a live request, prefer a short non-destructive command and stop immediately on unknown-field errors.

## Restart

Fully quit and reopen Codex desktop after changing the provider. Existing app-server processes and already-open threads may keep the previous provider snapshot until restart.
