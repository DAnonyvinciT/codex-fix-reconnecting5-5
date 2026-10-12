---
name: codex-fix-reconnecting5-5
description: 排查并修复 Windows 上 Codex 桌面版或 ChatGPT Work 每次回答前出现 Reconnecting 1/5 至 5/5、WebSocket 超时后回退 HTTP 的问题。适用于系统代理已开启、浏览器可用但任务反复重连的场景；通过原生 doctor 前后对照验证代理设置，保留现有 provider 和聊天状态。侧边栏“无法加载工作空间”同时出现时区分诊断，不直接归因。
---

# Codex 重连排查与修复

按“读取现状 → 同一环境对照 → 最小修改 → 分层验证”执行。历史实例见 [已验证案例](references/verified-case.md)。此技能的修复是版本相关方案，不能假定所有重连都由系统代理引起。

## 1. 确认正在失败的连接

- 从现有对话确认操作系统、发生位置、是否每次回答都先重试 5 次。能从环境或日志获得的信息不要重复询问。
- 区分模型响应 WebSocket、Remote Control 和账户工作空间列表。侧边栏或设置中的工作空间加载失败可能属于另一条请求链路，不能据此判定模型连接的根因。
- 查找实际随桌面应用运行的 `codex.exe`；不要假定 PATH 中的 CLI 与桌面版本一致。记录路径和 `--version`，下文用 PowerShell 变量 `$codexExe` 表示已确认的路径。
- 确认实际 CODEX_HOME 和当前配置路径，仅读取排查所需字段。沙箱可能重定向 APPDATA/LOCALAPPDATA；不要把路径不可访问或进程不可见解释为文件丢失、数据库损坏或应用已退出。

## 2. 读取代理与原生诊断

执行只读检查，按当前 CLI 的 `--help` 调整不支持的参数：

```powershell
& $codexExe --version
& $codexExe features list
& $codexExe doctor --summary --ascii --no-color
Get-ItemProperty -LiteralPath 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' -Name ProxyEnable,ProxyServer,AutoConfigURL -ErrorAction SilentlyContinue
Get-ChildItem Env: | Where-Object { $_.Name -match '^(HTTP_PROXY|HTTPS_PROXY|ALL_PROXY|NO_PROXY|WS_PROXY|WSS_PROXY)$' }
```

代理 URL 可能带凭据；输出前遮盖用户名、密码。记录系统代理是否开启、实际地址和端口、环境代理是否冲突，以及 doctor 的 websocket 项结果。代理客户端配置可辅助确认路由和 TUN 状态，不要读取订阅密钥或无关配置。

`features list` 中的 `respect_system_proxy` 是本案例的关键开关；必须确认当前版本存在。标记为 removed 的 `responses_websockets`、`responses_websockets_v2` 不应作为修复建议。`network_proxy` 不等于模型客户端采用 Windows 系统代理。

doctor 的非零退出码可能来自不相关检查。分别判断 WebSocket、认证、HTTP 可达性和配置解析，不要因为沙箱中的 SQLite 或进程检查失败而删库或改变安全设置。

## 3. 对照验证候选修复

若系统代理开启，默认 doctor 的 Responses WebSocket 超时，且版本支持该开关，执行临时覆盖，不先写配置：

```powershell
& $codexExe -c 'features.respect_system_proxy=true' doctor --summary --ascii --no-color
```

与默认 doctor 在同一身份、同一配置、同一网络环境下比较。默认超时而临时覆盖得到 `HTTP 101 Switching Protocols`，才有较强证据支持持久化此修改。若两者都失败，继续检查实际代理是否监听、规则路由、认证或服务状态；不要宣称已修好。

必要时辅助比较直连与已确认的代理到同一端点 `https://chatgpt.com/backend-api/codex/responses` 的 TLS/HTTP 可达性。使用无凭据的 curl 请求、短连接/总超时、丢弃响应体；直连用 `--noproxy '*'`，代理用 `--proxy` 指定实际地址。不要复用会话令牌或硬编码本案例端口。未认证请求得到 HTTP 401 只能证明网络/TLS 可达，不能证明认证后的 WebSocket 握手成功。

若 CLI 没有 doctor 或该 feature，先核对版本与可用诊断方式；不能静默写入未知开关。需要查在线资料时优先官方来源，并核对与当前版本的关系。

## 4. 持久化单一已验证改动

1. 在授权目录备份当前配置，避免覆盖已有备份。备份包含本机信息，不公开上传。
2. 读取最新配置，向已有 `[features]` 合并 `respect_system_proxy = true`；若无该表则新增。避免重复表、重复键或把顶层键误放入表中。
3. 保留模型、provider、插件、MCP、管道地址和其他设置。使用 TOML 解析检查结构，并比较修改前后，确保仅目标键变化。不要用过期备份覆盖运行时更新。

目标增量：

```toml
[features]
respect_system_proxy = true
```

只按工具明确要求申请窄范围文件写入权限。用户要求修复已授权常规可逆操作；不要增加没有依据的确认环节。

禁止把创建自定义 HTTP provider 当作此场景首选修复。内置 `openai` provider 可能禁止覆盖；自定义 provider 即使通过配置解析，也不代表已有线程会采用它或桌面功能兼容。不要为了重连修改线程数据库、身份或 Remote Control 设置。

## 5. 验证并交接

- 保存后，运行**不带 `-c` 覆盖**的 doctor。确认配置可加载、认证已配置、WebSocket 得到 HTTP 101；只报实际观察到的结果。
- 当前桌面进程可能需要完全退出再打开才能载入配置。保存工作后提示用户重开；不要主动终止承载当前对话的应用。
- 最终以重开后的真实任务是否仍发生 1/5 至 5/5 重连判断桌面体验。CLI 握手通过但尚未实测时，明确写“连接诊断通过，桌面重开后的任务待验证”。
- 若出现新问题，先撤回本次新增键或恢复该键原值，保留其他并发配置变化；不要直接覆盖整个配置。

仅在 doctor 不能定位时读日志：优先筛选 `codex_core::responses_retry`、`codex_core::client`、`codex_api::endpoint::responses_websocket` 和 `codex_core::session::startup_prewarm`，限制时间范围、条数和消息长度。Windows 本案例数据库为 CODEX_HOME 下 `logs_2.sqlite`，表结构会变化，先检查结构。SQLite 读取失败时区分权限、锁与损坏；若复制数据库，必须考虑 WAL 一致性，不能用不完整副本下结论。不要输出整段 debug 日志、用户提示词、工具调用或认证数据。

最终报告应含：根因证据、唯一配置改动、验证结果、仍待用户完成的桌面实测。不要把重连次数、耗时、端口或某个版本的行为当作通用保证。
