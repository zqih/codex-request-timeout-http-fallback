# Codex Request Timeout Fixes

解决 Codex 在代理环境下出现 `request timed out`、自动重试 5 次、WebSocket 不稳定的问题。

Fix Codex `request timed out`, 5 retries, and unstable WebSocket streaming in proxy environments.

这个仓库提供两种可复制给 Codex 或任何 coding agent 的修复方法：

- 方法一：降级为 HTTP/SSE，直接绕开 WebSocket。
- 方法二：保留 WebSocket，把 Codex Desktop 和代理链路配置正确。

## 方法一：降级为 HTTP/SSE

适合只想尽快恢复可用，不强求 WebSocket 的情况。

直接复制下面这段 Markdown 发给 Codex 或任何 coding agent。

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

5. 修改完成后，让我完全退出并重新打开 Codex 桌面端。已启动的 Codex 进程可能继续使用旧 provider snapshot。
````

## 方法二：默认 WebSocket，修好代理链路

适合刚安装 Codex、没有改过 `~/.codex/config.toml`，但在代理环境里遇到 WebSocket timeout 的情况。Codex 默认就会优先走 WebSocket streaming，所以这个方法不改 Codex provider，只把代理链路配稳。核心思路是：

- 确认本地代理端口真的可用。
- 让 Codex Desktop 进程通过 `launchctl` 继承代理环境变量。
- 在 Clash/FLClash 中把 OpenAI/Codex 相关域名固定到同一个稳定节点或策略组，避免 WebSocket 长连接被节点漂移打断。

直接复制下面这段 Markdown 发给 Codex 或任何 coding agent。

````markdown
请帮我修复 Codex 的 WebSocket 连接，不要降级 HTTP/SSE，也不要修改 Codex provider。目标是在新安装 Codex 的默认配置下，让 WebSocket streaming 通过本地代理稳定连通，并解决 `request timed out` 和自动重试 5 次。

你需要按下面步骤执行。修改 Clash/FLClash 配置前要先备份。不要打印代理订阅里的密码、token、节点密码或完整配置内容。

问题现象：
- Codex 首次启动或首次提问时出现 `request timed out`
- Codex 自动重试 5 次
- 日志里可能出现 `responses_websocket`
- 直连 `chatgpt.com` 可能超时，但走本地代理可以连通

目标状态：
- 不修改 `~/.codex/config.toml` 的 `model_provider`
- Codex 继续使用默认 WebSocket streaming
- Codex Desktop 进程能继承代理环境变量
- `chatgpt.com`、`openai.com`、`oaistatic.com`、`oaiusercontent.com`、`auth0.com` 等域名走同一个稳定代理策略组

请执行：

1. 确认本地代理端口。

   优先检查常见 mixed-port：

   ```bash
   lsof -nP -iTCP:7890 -sTCP:LISTEN
   lsof -nP -iTCP:7891 -sTCP:LISTEN
   ```

   如果用户使用的不是 7890，请从 Clash/FLClash 配置里读取实际 mixed-port/http-port/socks-port。后续命令里的 `127.0.0.1:7890` 要替换成真实端口。

2. 验证代理能访问 ChatGPT/Codex 后端。

   下面命令里，HTTP 401/403 都可以接受，因为没有带真实登录态；关键是不能 timeout。

   ```bash
   curl -I --connect-timeout 10 --proxy http://127.0.0.1:7890 https://chatgpt.com/
   curl -I --connect-timeout 10 --proxy http://127.0.0.1:7890 https://chatgpt.com/backend-api/codex/models
   ```

3. 测试 WebSocket Upgrade 链路是否能到达远端。

   HTTP 401/403 可以接受，timeout 不可以接受。

   ```bash
   curl -vk --http1.1 --connect-timeout 10 \
     --proxy http://127.0.0.1:7890 \
     -H 'Connection: Upgrade' \
     -H 'Upgrade: websocket' \
     -H 'Sec-WebSocket-Key: SGVsbG8sIHdvcmxkIQ==' \
     -H 'Sec-WebSocket-Version: 13' \
     https://chatgpt.com/backend-api/codex
   ```

4. 设置 Codex Desktop 可继承的代理环境变量。

   macOS 桌面应用不一定继承 shell 里的 `export`，所以用 `launchctl setenv`：

   ```bash
   launchctl setenv HTTP_PROXY http://127.0.0.1:7890
   launchctl setenv HTTPS_PROXY http://127.0.0.1:7890
   launchctl setenv ALL_PROXY http://127.0.0.1:7890
   launchctl setenv WS_PROXY http://127.0.0.1:7890
   launchctl setenv WSS_PROXY http://127.0.0.1:7890
   launchctl setenv NO_PROXY localhost,127.0.0.1,::1
   ```

   如果 7890 是 mixed-port 且 HTTP 代理不稳定，可把 `ALL_PROXY`、`WS_PROXY`、`WSS_PROXY` 改成：

   ```bash
   launchctl setenv ALL_PROXY socks5h://127.0.0.1:7890
   launchctl setenv WS_PROXY socks5h://127.0.0.1:7890
   launchctl setenv WSS_PROXY socks5h://127.0.0.1:7890
   ```

5. 固定 Clash/FLClash 规则。

   在当前生效配置中添加 OpenAI/Codex 域名规则，让它们走同一个稳定策略组。策略组名称优先使用用户已有的 AI 组，例如 `Research + AI`；如果没有，就使用用户当前主要代理组，例如 `PROXY`、`Proxy`、`🚀 节点选择`。不要凭空创建会破坏订阅结构的新组。

   推荐规则：

   ```yaml
   rules:
     - DOMAIN-SUFFIX,chatgpt.com,Research + AI
     - DOMAIN-SUFFIX,openai.com,Research + AI
     - DOMAIN-SUFFIX,oaistatic.com,Research + AI
     - DOMAIN-SUFFIX,oaiusercontent.com,Research + AI
     - DOMAIN-SUFFIX,auth0.com,Research + AI
     - DOMAIN-KEYWORD,openai,Research + AI
   ```

   如果用户使用 FLClash，可以同时检查这些位置：

   - `~/Library/Application Support/com.follow.clash/config.yaml`
   - `~/Library/Application Support/com.follow.clash/profiles/*.yaml`
   - `~/Library/Preferences/com.follow.clash.plist`

   对 FLClash 的 `plist`，用 `plistlib` 读取 `flutter.config` JSON：

   - 找到 `currentProfileId`
   - 找到对应 profile
   - 设置 `overrideData.enable = true`
   - 设置 `overrideData.rule.type = "added"`
   - 把上面的规则加入 `overrideData.rule.addedRules`
   - 如果存在 `selectedMap["Research + AI"]`，把它固定到一个稳定节点，例如用户已有的 `US Fixed IP`

   重要：修改前备份这些文件，修改时不要把节点密码、订阅 token 或完整 YAML 打印到聊天里。

6. 让配置生效。

   - 在 Clash/FLClash 里重载配置，或完全退出后重新打开。
   - 完全退出 Codex Desktop，然后重新打开。
   - 新开一个 Codex 线程测试首次提问。

7. 验证。

   - 如果不再出现 `request timed out` 和 5 次 retry，说明 WebSocket 链路已恢复。
   - 如果日志中出现 `responses_websocket` 且没有 timeout，说明仍在使用 WebSocket 并且连接成功。
   - 如果仍然 5 次 retry，再检查 Codex 进程是否继承了 `launchctl` 变量、代理端口是否监听、OpenAI/Codex 域名是否被同一策略组命中。
````

## Keywords

Codex request timed out, Codex retries 5 times, Codex WebSocket timeout, Codex HTTP/SSE, Codex proxy, Clash, FLClash, config.toml, responses_websocket, responses_http.
