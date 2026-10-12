# codex-fix-reconnecting5/5

Windows 上 Codex 桌面版 / ChatGPT Work 反复出现 `Reconnecting 1/5` 至 `5/5` 的排查技能，整理自一次实际故障处理。

技能标识为 `codex-fix-reconnecting5-5`（标识与目录名不能包含 `/`）。

## 适用场景

- Windows 系统代理已开启，浏览器可以联网，但 Codex 每次回答前反复重连。
- Responses WebSocket 超时，重试后回退到 HTTP。
- 希望通过原生诊断与最小配置修改解决问题，保留现有模型、provider 和聊天记录。

## 核心方法

先比较默认 `codex doctor` 与临时启用 `features.respect_system_proxy=true` 后的 WebSocket 结果。仅在当前版本支持该设置，且对照检查证明有效时，向现有配置合并：

```toml
[features]
respect_system_proxy = true
```

保存后再次运行不带临时覆盖的诊断，再验证桌面重开后的真实任务。该方案依赖版本和网络环境，并非所有重连问题的通用修复。

## 安装

将本仓库下载或克隆到个人技能目录中的 `codex-fix-reconnecting5-5` 文件夹：

```powershell
git clone https://github.com/DAnonyvinciT/codex-fix-reconnecting5-5.git "$env:USERPROFILE\.codex\skills\codex-fix-reconnecting5-5"
```

如果使用自定义 `CODEX_HOME`，应安装到该目录的 `skills` 子目录。若目标技能文件夹已存在，请先比较版本，避免覆盖个人修改。

新建会话后输入：

```text
使用 $codex-fix-reconnecting5-5 排查每次回答前 Reconnecting 5/5 的问题，并验证最小修复。
```

## 文件

- [SKILL.md](SKILL.md)：完整诊断、修改、验证和回滚流程。
- [references/verified-case.md](references/verified-case.md)：实际证据、失败尝试和验证边界。
- [agents/openai.yaml](agents/openai.yaml)：技能显示名称和默认调用提示。

本仓库仅包含技能与脱敏案例，不包含真实配置备份、完整日志、聊天数据库或认证信息。案例已验证 CLI WebSocket 握手成功；桌面重开后的任务表现尚无用户确认记录。
