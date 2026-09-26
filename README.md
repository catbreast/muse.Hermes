# muse.Hermes

用 WorkBuddy 的免费 AI 额度，搭一个属于你自己的微信机器人。

## 这是什么

两样东西拼在一起：

- **[WorkBuddy Gateway](https://github.com/CangShui/workbuddy-gateway)（作者 CangShui），
  把你的 WorkBuddy 账号转成标准的 OpenAI 接口，跑在 `127.0.0.1:8317`。
- **Hermes**：跑在沙盒里的 AI Agent，接上网关之后，就能在微信里跟你聊天、帮你干活。

主模型固定用 **`deepseek-v4.1-flash`**，走 WorkBuddy 账号的免费额度。

## 花钱吗

不花。WorkBuddy 账号自带免费额度，网关只是本地转发；Hermes 本体开源免费。
整套跑下来，模型调用 0 成本。

## 仓库里有什么

| 文件 | 给谁看的 |
|---|---|
| `README.md`（本篇） | 给人看的：这是什么、怎么用、注意什么 |
| `skills/workbuddy-hermes-deploy/SKILL.md` | 给 AI 看的：完整部署手册，一步一步照着执行就行 |

## 快速开始（概述）

1. 在一台机器上编译并启动 WorkBuddy 网关；
2. 在浏览器里登录你的 WorkBuddy 账号，凭据自动保存到本地；
3. 安装 Hermes，把它的模型配置指向本地网关；
4. 微信扫码，把机器人绑定到你的微信；
5. 加上保活：网关挂了 1 分钟内自动重启。

具体每一步的命令、话术、驗證方法和排错表都在
`skills/workbuddy-hermes-deploy/SKILL.md` 里——直接把那篇交给你的 AI 助手，
让它"走一步问一步"带你装完。

## 稳定性是怎么保证的

- **网关保活**：每 1 分钟检查一次 `/health`，挂了自动重启，正常时静默；
- **沙盒重建**：所有东西都在 `$HOME` 下（重建不丢），配合恢复脚本自动拉起；
- **微信/Telegram 通道**：由 Hermes 网关的独立保活盯着。

## 注意事项

- 凭据文件（`workbuddy-*.json`）权限设为 600，**不要**提交到公开仓库；
- 不要批量探测网关的 `/v1/models` 模型列表——里面混着不存在的别名，
  乱探测会触发网关的限流误伤，把正常模型也拖下水；
- Hermes 配置里 `discover_models` 保持 `false`（原因同上）；
- 免费公网隧道（如果用）域名会变，保活脚本会自动更新并通知你。

## 致谢

- [WorkBuddy Gateway](https://github.com/CangShui/workbuddy-gateway)（作者 CangShui）——
  本方案的网关基础：把 WorkBuddy / CodeBuddy 账号转成标准 OpenAI 接口的开源项目，
  没有它就没有这套免费方案，感谢开源。

## License

MIT
