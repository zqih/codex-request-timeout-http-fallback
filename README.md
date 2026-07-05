# Codex Request Timeout HTTP Fallback

解决 Codex 在代理环境下出现 `request timed out`、自动重试 5 次、WebSocket 不稳定的问题。

Fix Codex `request timed out`, 5 retries, and unstable WebSocket streaming in proxy environments.

直接复制下面这段 Markdown 发给 Codex 或任何 coding agent，让它帮你改配置即可。

Copy the Markdown below and send it to Codex or any coding agent.

````markdown
请帮我修改 Codex 配置，让 Codex 使用 HTTP/SSE provider，不走 WebSocket streaming。

问题现象：
- Codex 首次启动或首次提问时出现 `request timed out`
- Codex 自动重试 5 次
- 代理环境下 WebSocket 不稳定
- 日志里可能出现 `responses_websocket`

请执行：

1. 打开 Codex 配置文件：

   `~/.codex/config.toml`

2. 先备份：

   ```bash
   cp ~/.codex/config.toml ~/.codex/config.toml.bak-$(date +%Y%m%d-%H%M%S)
   ```

3. 在配置中加入或更新：

   ```toml
   model_provider = "openai-http"

   [model_providers.openai-http]
   name = "OpenAI HTTP"
   base_url = "https://chatgpt.com/backend-api/codex"
   wire_api = "responses"
   supports_websockets = false
   requires_openai_auth = true
   ```

4. 确保只有一个 `model_provider = "openai-http"`，也只有一个 `[model_providers.openai-http]` 配置块。

5. 修改完成后，让我完全退出并重新打开 Codex 桌面端。
````

Keywords: Codex request timed out, Codex retries 5 times, Codex WebSocket timeout, Codex HTTP/SSE, Codex proxy, Clash, FLClash, config.toml.
