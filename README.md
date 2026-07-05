# Codex Request Timeout HTTP Fallback

中文 | [English](#english)

## 中文

这个仓库提供一个一次性配置方法，用来解决部分代理网络环境下 Codex 首次启动或首次提问时反复出现：

- `request timed out`
- Codex 自动重试 5 次
- WebSocket 连接不稳定
- `responses_websocket` 相关连接失败
- 使用代理、Clash、FLClash、VPN 时 Codex 请求超时

核心方法是：**不要继续依赖 WebSocket streaming，而是在 Codex 的 `config.toml` 中新增一个 HTTP/SSE OpenAI Provider，并把默认 provider 切换到它。**

适合搜索这些问题的开发者：

- Codex request timed out
- Codex 重试 5 次
- Codex WebSocket 超时
- Codex 代理超时
- Codex HTTP SSE 配置
- OpenAI Codex config.toml provider
- FLClash / Clash 下 Codex 不稳定

## 快速配置

编辑：

```bash
~/.codex/config.toml
```

先备份：

```bash
cp ~/.codex/config.toml ~/.codex/config.toml.bak-$(date +%Y%m%d-%H%M%S)
```

加入或更新以下配置：

```toml
model_provider = "openai-http"

[model_providers.openai-http]
name = "OpenAI HTTP"
base_url = "https://chatgpt.com/backend-api/codex"
wire_api = "responses"
supports_websockets = false
requires_openai_auth = true
```

然后完全退出并重新打开 Codex 桌面端。

## 给 Codex 执行

这个方法不需要常驻安装成 skill。你可以把 `SKILL.md` 里的说明复制给 Codex，或者直接对 Codex 说：

```text
按照这个仓库里的 SKILL.md，把我的 Codex config.toml 配置成 HTTP/SSE provider，不走 WebSocket streaming。
```

## English

This repository provides a one-time configuration recipe for fixing a common Codex startup/request failure in proxy-heavy network environments:

- `request timed out`
- Codex retries 5 times
- unstable WebSocket streaming
- `responses_websocket` connection failures
- Codex timeout when using Clash, FLClash, VPN, or other proxy tools

The method is simple: **add a custom HTTP/SSE OpenAI Provider in Codex `config.toml`, then make Codex use that provider instead of WebSocket streaming.**

Useful search keywords:

- Codex request timed out
- Codex retries 5 times
- Codex WebSocket timeout
- Codex proxy timeout
- Codex HTTP SSE config
- OpenAI Codex config.toml provider
- Codex with Clash / FLClash

## Quick Setup

Edit:

```bash
~/.codex/config.toml
```

Back it up first:

```bash
cp ~/.codex/config.toml ~/.codex/config.toml.bak-$(date +%Y%m%d-%H%M%S)
```

Add or update this configuration:

```toml
model_provider = "openai-http"

[model_providers.openai-http]
name = "OpenAI HTTP"
base_url = "https://chatgpt.com/backend-api/codex"
wire_api = "responses"
supports_websockets = false
requires_openai_auth = true
```

Then fully quit and reopen the Codex desktop app.

## Ask Codex to Apply It

This method does not need to be installed as a persistent skill. You can paste `SKILL.md` into Codex, or ask Codex:

```text
Follow the SKILL.md in this repository and configure my Codex config.toml to use an HTTP/SSE provider instead of WebSocket streaming.
```

## Files

- `SKILL.md`: concise instructions that Codex can follow once to update `config.toml`.
