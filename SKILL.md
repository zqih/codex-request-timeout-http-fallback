---
name: codex-request-timeout-fixes
description: Fix Codex "request timed out" and 5 retries in proxy environments. Use when Codex WebSocket streaming is unstable, when logs mention responses_websocket, when the user wants HTTP/SSE fallback, or when the user wants to keep WebSocket by configuring local proxy, launchctl proxy variables, and Clash/FLClash rules.
---

# Codex Request Timeout Fixes

## Overview

This skill fixes Codex `request timed out` and repeated retry failures in proxy environments. There are two methods:

1. **HTTP/SSE fallback**: add a custom `openai-http` provider with `supports_websockets = false`.
2. **Keep WebSocket**: restore the default `openai` provider and make the proxy path stable for WebSocket traffic.

Ask which method the user wants only when their intent is unclear. If the user wants the quickest workaround, use Method 1. If the user says they still want WebSocket, use Method 2.

Before changing any user config, create timestamped backups. Do not print proxy subscription secrets, node passwords, tokens, or full Clash YAML contents.

## Method 1: HTTP/SSE Fallback

Use this method to directly avoid WebSocket streaming.

### Configuration

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

### Apply Safely

Back up the real config:

```bash
cp ~/.codex/config.toml ~/.codex/config.toml.bak-$(date +%Y%m%d-%H%M%S)
```

Then edit `~/.codex/config.toml` so it contains exactly one `model_provider = "openai-http"` setting and one `[model_providers.openai-http]` block. If an older `openai-http` block already exists, update it in place instead of adding a duplicate.

Recommended placement:

1. Put `model_provider = "openai-http"` near other top-level settings such as `service_tier`.
2. Put `[model_providers.openai-http]` before marketplace and project tables.
3. Keep project trust settings unchanged.

### Validate

Inspect the first part of the config:

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

### Restart

Fully quit and reopen Codex Desktop after changing the provider. Existing app-server processes and already-open threads may keep the previous provider snapshot until restart.

## Method 2: Keep WebSocket And Fix Proxy

Use this method when the user wants to keep Codex's default WebSocket streaming.

### Restore The Default Provider

Back up the config:

```bash
cp ~/.codex/config.toml ~/.codex/config.toml.bak-restore-websocket-$(date +%Y%m%d-%H%M%S)
```

Edit `~/.codex/config.toml` and remove the custom HTTP fallback selection:

```toml
model_provider = "openai-http"
```

Also remove the full custom block:

```toml
[model_providers.openai-http]
name = "OpenAI HTTP"
base_url = "https://chatgpt.com/backend-api/codex"
wire_api = "responses"
supports_websockets = false
requires_openai_auth = true
```

Keep project trust settings and unrelated desktop/plugin settings unchanged. After removal, Codex should use the built-in `openai` provider.

### Find And Test The Local Proxy Port

Check common Clash/FLClash ports:

```bash
lsof -nP -iTCP:7890 -sTCP:LISTEN
lsof -nP -iTCP:7891 -sTCP:LISTEN
```

If these are not listening, read the user's Clash/FLClash config to find `mixed-port`, `port`, or `socks-port`. Use the real port in all commands below.

Verify that the proxy reaches ChatGPT/Codex. HTTP 401 or 403 is acceptable because the request lacks Codex auth; timeout is not acceptable.

```bash
curl -I --connect-timeout 10 --proxy http://127.0.0.1:7890 https://chatgpt.com/
curl -I --connect-timeout 10 --proxy http://127.0.0.1:7890 https://chatgpt.com/backend-api/codex/models
```

Test WebSocket Upgrade reachability. HTTP 401 or 403 is acceptable; timeout is not.

```bash
curl -vk --http1.1 --connect-timeout 10 \
  --proxy http://127.0.0.1:7890 \
  -H 'Connection: Upgrade' \
  -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Key: SGVsbG8sIHdvcmxkIQ==' \
  -H 'Sec-WebSocket-Version: 13' \
  https://chatgpt.com/backend-api/codex
```

### Set Desktop Proxy Environment

On macOS, GUI apps do not reliably inherit shell `export` values. Use `launchctl setenv` so newly launched Codex Desktop receives proxy variables:

```bash
launchctl setenv HTTP_PROXY http://127.0.0.1:7890
launchctl setenv HTTPS_PROXY http://127.0.0.1:7890
launchctl setenv ALL_PROXY http://127.0.0.1:7890
launchctl setenv WS_PROXY http://127.0.0.1:7890
launchctl setenv WSS_PROXY http://127.0.0.1:7890
launchctl setenv NO_PROXY localhost,127.0.0.1,::1
```

If the local port is a mixed port and HTTP proxying still fails for WebSocket, switch the WebSocket-related variables to SOCKS DNS mode:

```bash
launchctl setenv ALL_PROXY socks5h://127.0.0.1:7890
launchctl setenv WS_PROXY socks5h://127.0.0.1:7890
launchctl setenv WSS_PROXY socks5h://127.0.0.1:7890
```

### Pin OpenAI/Codex Domains To One Proxy Group

Add these rules to the active Clash/FLClash profile. Replace `Research + AI` with the user's existing stable proxy group if that group does not exist.

```yaml
rules:
  - DOMAIN-SUFFIX,chatgpt.com,Research + AI
  - DOMAIN-SUFFIX,openai.com,Research + AI
  - DOMAIN-SUFFIX,oaistatic.com,Research + AI
  - DOMAIN-SUFFIX,oaiusercontent.com,Research + AI
  - DOMAIN-SUFFIX,auth0.com,Research + AI
  - DOMAIN-KEYWORD,openai,Research + AI
```

For FLClash, inspect these paths when present:

- `~/Library/Application Support/com.follow.clash/config.yaml`
- `~/Library/Application Support/com.follow.clash/profiles/*.yaml`
- `~/Library/Preferences/com.follow.clash.plist`

For the plist, use `plistlib` to read `flutter.config` JSON:

1. Find `currentProfileId`.
2. Find the matching profile.
3. Set `overrideData.enable = true`.
4. Set `overrideData.rule.type = "added"`.
5. Add the OpenAI/Codex rules to `overrideData.rule.addedRules`.
6. If the profile has `selectedMap["Research + AI"]`, pin it to an existing stable node such as `US Fixed IP`. If the exact node does not exist, choose a healthy existing non-auto node or ask the user.

Back up every FLClash file before writing. Avoid printing full YAML or secrets.

### Restart And Validate

Ask the user to reload Clash/FLClash or fully restart it. Then fully quit and reopen Codex Desktop so it inherits the new `launchctl` environment.

Validate with a new Codex thread:

- No `request timed out`.
- No 5 repeated retries.
- Logs can mention `responses_websocket`; that is expected when WebSocket is kept.
- If `responses_websocket` appears without timeout, the WebSocket path is working.
