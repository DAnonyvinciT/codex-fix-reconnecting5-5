# 本次排查记录

此记录总结实际观察，供复现诊断思路。未包含用户配置、令牌、完整日志或个人路径。

## 环境与症状

- Windows，Codex / Work 任务每次回答先出现正在重新连接 1/5 至 5/5。
- 实测 CLI：`codex-cli 0.162.0-alpha.17.2`。后续版本需重新发现功能。
- Windows 系统代理开启，指向本地 Clash Verge；此实例地址是 `127.0.0.1:7897`，规则模式，TUN 关闭，无 HTTP_PROXY / HTTPS_PROXY 环境变量。其他机器必须重新检测地址。
- 同时报告侧边栏或设置“无法加载工作空间”，未取得该错误的独立根因证据。

## 因果证据

1. 相关日志显示 Responses WebSocket 启动预热超时，采样请求从 1/5 重试至 5/5，之后 `falling back to HTTP`。目标为 `wss://chatgpt.com/backend-api/codex/responses`。
2. 无认证直连该端点连接超时；显式通过本地代理的测试约 1.4 秒收到 HTTP 401，证明经代理的 TLS/HTTP 路径可达。401 不代表 WebSocket 成功。
3. 当前版本 `features list` 显示 `respect_system_proxy` 默认 false，状态 under development；旧 `responses_websockets*` 已 removed。
4. 默认 `codex doctor --summary --ascii --no-color` 显示 Responses WebSocket 超时。
5. 同一环境用 `-c 'features.respect_system_proxy=true'` 临时覆盖后，doctor 显示 `websocket connected (HTTP 101 Switching Protocols)`。
6. 合并 `[features] respect_system_proxy = true` 后，再次运行不带临时覆盖的 doctor，配置、认证、WebSocket 和 HTTP 可达性检查均通过。结构对比确认仅新增这一配置键。

证据支持：在该版本与网络环境下，让客户端采用 Windows 系统代理修复了 CLI 的 Responses WebSocket 连接。用户尚未明确确认桌面完全重开后的真实任务表现，不能把这一层写成已完成验证。

## 无效尝试与边界

- 尝试覆盖 `[model_providers.openai]` 来关闭 WebSocket，被 CLI 拒绝：内置 provider ID 为保留项。
- 尝试自定义 `openai-http` provider，配置虽通过解析，已有任务仍重连 5/5，随后已撤销。未证明它造成工作空间错误，不能建立该因果关系。
- 工作空间发现请求属于账户工作空间接口，模型生成使用另一条链路；错误同时出现不足以说明根因相同。
- 沙箱中 SQLite、进程或应用包查询有权限限制。不能据此判断真实数据库损坏或桌面未运行。
- 没有通过删除聊天数据库、重置账户、修改 Remote Control 或永久写全局代理环境变量来解决本案例。

## 外部资料的定位

曾检索社区技能，但没有安装或运行；它们不是本次成功修复的验证证据：

- [zhenyangLab/Codex-Connectivity](https://github.com/zhenyangLab/Codex-Connectivity/blob/main/SKILL.md)：Windows 进程级代理启动器方案，可作为后续研究线索。
- [baixinpan/codex-reconnecting-doctor](https://github.com/baixinpan/codex-reconnecting-doctor/blob/main/SKILL.md)：其他连接诊断思路，需独立核对当前版本。

配置与环境背景可参考 [配置参考](https://learn.chatgpt.com/docs/config-file/config-reference)、[环境变量](https://learn.chatgpt.com/docs/config-file/environment-variables)。本案例开关的支持性来自本机 `features list` 和 doctor 对照，而非假定文档保证所有版本支持。
