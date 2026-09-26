---
name: workbuddy-hermes-deploy
description: "在沙盒上从零部署「WorkBuddy 免费网关 + Hermes 微信机器人」：编译并启动 workbuddy-gateway、引导用户在浏览器登录 WorkBuddy 国际站并保存账号凭据、安装 Hermes Agent、把 deepseek-v4.1-flash 配成主要模型（provider custom + discover_models:false）、微信扫码绑定、端到端验证、加独立保活。触发词：部署 workbuddy、免费模型网关、hermes 微信机器人、deepseek-v4.1-flash。"
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [workbuddy, gateway, hermes, weixin, deployment, free-model]
    related_skills: [sandbox-keepalive-wechat]
---

# WorkBuddy 免费网关 + Hermes 微信机器人：部署手册

## Purpose

WorkBuddy Local Gateway（Go 单二进制）把 WorkBuddy/CodeBuddy 账号反代成标准 OpenAI 协议，
Hermes Agent 把它当 `custom` provider 用，主模型固定 **`deepseek-v4.1-flash`**（WorkBuddy 免费额度，
0 credit 消耗）。本手册覆盖：网关部署 → 用户浏览器登录 → 凭据落盘 → Hermes 安装 →
模型配置 → 微信绑定 → 端到端验证 → 独立保活。

**走一步问一步**：不要一次性抛所有步骤，每步做完停下来跟用户确认再往下走。
每一步固定五段式：① AI 执行（可直接复制的命令）② 对用户说（可原样发送的话术）
③ 用户手动做 ④ 做完的标志（可验证判据，做不到不往下走）⑤ 排错表。

**与 `sandbox-keepalive-wechat` 的分工**：那个 skill 管探针 + 三层保活 + Hermes 数据持久化；
本手册只管"网关 + 免费模型 + Hermes 初装"。沙盒重建后的恢复走那边的 `restore-all.sh`，
本手册不重复造轮子。

## 步骤 0：环境探测

### ① AI 执行

```bash
whoami; echo "HOME=$HOME"
curl -sS -m 8 -o /dev/null -w 'github %{http_code}\n' https://github.com
ls ~/workspace/tools/go/bin/go 2>/dev/null || echo "no persistent go"
```

### ② 对用户说

> 先看一眼环境：确认能上 GitHub、Go 工具链在不在。

### ③ 用户手动做

无。

### ④ 做完的标志

GitHub 返回的不是 000；Go 存在（`~/workspace/tools/go/bin/go`，见排错）。

### ⑤ 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| GitHub 000 | 沙盒出网受限 | 换代理或稍后重试；源码已有本地副本 `~/workspace/projects/workbuddy-gateway/` 可直接用 |
| 没有 Go | `/usr/local/go` 被沙盒重建清空过 | 用持久化路径 `~/workspace/tools/go/`（本手册步骤 1 会装） |

## 步骤 1：部署 workbuddy-gateway

**目标**：网关跑在 `127.0.0.1:8317`，`/health` 返回 healthy。**耗时**：5–10 分钟。

### ① AI 执行

```bash
# 1. Go 工具链（持久化到 $HOME，重建不丢）
export GOROOT=~/workspace/tools/go PATH=~/workspace/tools/go/bin:$PATH
go version || { mkdir -p ~/workspace/tools && cd ~/workspace/tools &&
  curl -fsSL https://go.dev/dl/go1.24.7.linux-amd64.tar.gz -o go.tgz &&
  tar xzf go.tgz && rm go.tgz; }

# 2. 取源码（已有本地副本就直接用，里面含 2026-09-27 的 isModelNotFound 修复）
[ -d ~/workspace/projects/workbuddy-gateway ] || \
  git clone https://github.com/CangShui/workbuddy-gateway ~/workspace/projects/workbuddy-gateway

# 3. 编译
export GOROOT=~/workspace/tools/go PATH=~/workspace/tools/go/bin:$PATH
cd ~/workspace/projects/workbuddy-gateway
go vet ./... && go build -o workbuddy-gateway .

# 4. 运行目录 + 启动（serve 是默认子命令；凭据自动发现工作目录下 workbuddy*.json）
mkdir -p ~/workspace/workbuddy/logs
cd ~/workspace/workbuddy
[ -f config.json ] || cp ~/workspace/projects/workbuddy-gateway/config.example.json config.json
nohup ~/workspace/projects/workbuddy-gateway/workbuddy-gateway serve -port 8317 \
  > logs/serve.log 2>&1 < /dev/null &
sleep 3
curl -s -m 8 http://127.0.0.1:8317/health
```

### ② 对用户说

> 网关正在启动，它是个本地反代服务，把你的 WorkBuddy 账号转成标准 OpenAI 接口给 Hermes 用。
> 跑起来后需要你登录一次 WorkBuddy 账号（下一步）。

### ③ 用户手动做

无（等 AI 确认 health 正常）。

### ④ 做完的标志

```bash
curl -s -m 8 http://127.0.0.1:8317/health
# 期望：{"status":"healthy",...}，model_count>0（未登录时可能为 0，属正常，下一步登录后会涨）
pgrep -fa "workbuddy-gateway serve"   # 期望有进程
```

### ⑤ 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `go: command not found` | GOROOT 没 export | 每次新开 shell 都要先 `export GOROOT=~/workspace/tools/go PATH=~/workspace/tools/go/bin:$PATH` |
| 编译报 vet 失败 | 源码被改坏 | 用 `git stash` / 重新 clone；本地副本的 `main.go` 含 isModelNotFound 修复，不要丢 |
| 8317 已被占用 | 旧网关进程还在 | `pkill -f "workbuddy-gateway serve"` 后重起；或检查是不是保活任务刚重启了它 |
| /health 000 | 启动失败 | 看 `~/workspace/workbuddy/logs/serve.log` 尾部 |

## 步骤 2：用户浏览器登录 + 凭据落盘

**目标**：`~/workspace/workbuddy/workbuddy-intl.json` 生成（600 权限），网关热加载生效。
**耗时**：5 分钟（等用户在浏览器点）。

### ① AI 执行

```bash
cd ~/workspace/workbuddy
# -intl = 国际站 www.workbuddy.ai（浏览器内完成登录）；默认是国内站微信扫码
# -auth 指定凭据文件名；网关运行期每 5 秒热加载，无需重启
~/workspace/projects/workbuddy-gateway/workbuddy-gateway login -intl -auth workbuddy-intl.json
# 命令会输出一个登录 URL，把它发给用户
```

### ② 对用户说

> 网关需要你的 WorkBuddy 账号才能用免费额度。点这个链接（AI 把上面命令输出的 URL 贴过来），
> 在浏览器里登录你的 WorkBuddy 国际站账号，登录成功后凭据会自动存到沙盒，
> 文件权限 600，只有这台机器能读。

### ③ 用户手动做

- 在浏览器打开登录 URL，完成 WorkBuddy 国际站登录；
- 告诉 AI "登好了"。

### ④ 做完的标志

```bash
ls -l ~/workspace/workbuddy/workbuddy-intl.json          # 期望存在，-rw-------
~/workspace/projects/workbuddy-gateway/workbuddy-gateway status  # 期望看到账号（UID/剩余额度）
sleep 6; curl -s -m 8 http://127.0.0.1:8317/health | head -c 300   # model_count 应 >0
```

### ⑤ 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `status` 显示的 UID 和上次一样（想加第二个账号却复用了旧账号） | 浏览器里旧账号没登出，登录页直接复用了会话 | **让用户先在浏览器里彻底登出旧 WorkBuddy 账号**，再重跑 `login -intl -auth workbuddy-intl1.json` |
| 凭据文件生成了但 model_count 仍 0 | 热加载还没扫到（5 秒间隔）或 token 拉取失败 | 等 10 秒重查 `/health`；看 `logs/serve.log` 有无 401 |
| 用户问"这登录安不安全" | — | 凭据只存 `~/workspace/workbuddy/*.json`（600），不进任何文档/记忆/日志；网关只在本地 127.0.0.1 监听 |

## 步骤 3：安装 Hermes Agent

**目标**：`hermes` 可用，`~/.hermes/` 初始化。**耗时**：5–10 分钟。

### ① AI 执行

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh -o /tmp/hermes-install.sh
bash /tmp/hermes-install.sh
export PATH="$HOME/.local/bin:$PATH"
hermes --version
```

### ② 对用户说

> 现在装 Hermes 本体（跑在沙盒里的 AI Agent），装完把它的"大脑"接到刚才的免费网关上。

### ③ 用户手动做

无。

### ④ 做完的标志

`hermes --version` 有输出；`~/.hermes/config.yaml` 存在。

### ⑤ 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| 安装脚本下载失败 | GitHub 不通 | 用步骤 0 的代理/重试；或找已有沙盒拷 `~/.local/bin/hermes*` |
| `hermes: command not found` | PATH 没含 `~/.local/bin` | `export PATH="$HOME/.local/bin:$PATH"`（写进 shell profile 持久化） |

## 步骤 4：配置模型为 deepseek-v4.1-flash

**目标**：Hermes 主模型走 `http://127.0.0.1:8317/v1` 的 `deepseek-v4.1-flash`。**耗时**：3 分钟。

> 关键事实：网关**不校验** key，`key_env` 指向的变量只要非空即可（填 `dummy` 也行）；
> `discover_models: false` 是必须的——网关 `/v1/models` 目录里混着上游不存在的别名
> （auto/default/primary-model 等），不关掉选择器会把它们当可用模型。

### ① AI 执行

```bash
# 1. key：网关不校验，任意非空值即可（不要把真实 key 写进 config）
grep -q '^MY_API_KEY=' ~/.hermes/.env 2>/dev/null || echo 'MY_API_KEY=dummy' >> ~/.hermes/.env
chmod 600 ~/.hermes/.env

# 2. 模型配置（直接改文件最可靠；hermes config set 对某些段写不进去）
#    只改顶层 model: 段内的四个键，不碰其他段的同名键
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak-before-workbuddy
python3 - <<'EOF'
import re
p = '/home/hatch/.hermes/config.yaml'
lines = open(p).read().split('\n')
out, in_model, changed = [], False, set()
for i, ln in enumerate(lines):
    if re.match(r'^model:\s*$', ln):
        in_model = True
        out.append(ln)
        continue
    if in_model and re.match(r'^[^\s#]', ln):  # 下一个顶层键，model 段结束
        in_model = False
    if in_model:
        m = re.match(r'^(\s*)(provider|base_url|key_env|default)(\s*:).*', ln)
        if m and m.group(2) not in changed:
            vals = {'provider': '"custom"',
                    'base_url': '"http://127.0.0.1:8317/v1"',
                    'key_env': '"MY_API_KEY"',
                    'default': '"deepseek-v4.1-flash"'}
            ln = f"{m.group(1)}{m.group(2)}: {vals[m.group(2)]}"
            changed.add(m.group(2))
    out.append(ln)
assert changed == {'provider', 'base_url', 'key_env', 'default'}, f"未找齐四个键: {changed}"
open(p, 'w').write('\n'.join(out))
print("model 段已更新:", sorted(changed))
EOF
grep -A4 '^model:' ~/.hermes/config.yaml | grep -v '^#' | head -8

# 3. 关掉模型自动发现（插在 model: 段内，保持两空格缩进）
python3 - <<'EOF'
p = '/home/hatch/.hermes/config.yaml'
lines = open(p).read().split('\n')
if not any(l.strip() == 'discover_models: false' for l in lines):
    idx = next(i for i, l in enumerate(lines) if l.rstrip() == 'model:')
    lines.insert(idx + 1, '  discover_models: false')
    open(p, 'w').write('\n'.join(lines))
    print("已插入 discover_models: false")
else:
    print("discover_models 已存在，跳过")
EOF
python3 -c "import yaml;yaml.safe_load(open('/home/hatch/.hermes/config.yaml'));print('YAML OK')"
```

### ② 对用户说

> 把 Hermes 的大脑接到免费网关：模型固定用 `deepseek-v4.1-flash`，走你刚登录的 WorkBuddy 免费额度，
> 不需要任何付费 API key。

### ③ 用户手动做

无。

### ④ 做完的标志

```bash
hermes config get model.provider model.base_url model.key_env model.default
# 期望：custom / http://127.0.0.1:8317/v1 / MY_API_KEY / deepseek-v4.1-flash
```

### ⑤ 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `config get` 显示的值和文件不一致 | Hermes 读了别的 profile | 确认改的是 `~/.hermes/config.yaml` 本体；备份在 `config.yaml.bak-before-workbuddy` |
| 之后 `hermes -z` 报 No LLM provider configured | `.env` 没被加载（Hermes 二进制直接读 `~/.hermes/.env`） | 确认 `MY_API_KEY` 在 `~/.hermes/.env` 里且文件 600 |

## 步骤 5：微信绑定

**目标**：`hermes gateway run` 显示微信通道 connected。**耗时**：5 分钟（等用户扫码）。

### ① AI 执行

```bash
mkdir -p ~/.hermes/logs
nohup hermes gateway run >> ~/.hermes/logs/weixin-gateway.log 2>&1 < /dev/null &
sleep 5
tail -20 ~/.hermes/logs/weixin-gateway.log
```

### ② 对用户说

> 网关已启动。现在需要绑定你的微信：我在日志里看到二维码（或配对码）后会告诉你，
> 你用微信扫一下完成绑定。

### ③ 用户手动做

- 按 AI 给的二维码/配对码完成微信扫码绑定。

### ④ 做完的标志

日志里出现 `✓ weixin connected`（或中文"微信通道已连接"）；`hermes pairing list` 能看到已配对账号。

### ⑤ 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| 二维码刷不出来 | 日志还没写 | `tail -f ~/.hermes/logs/weixin-gateway.log` 等 30 秒 |
| 扫码后无反应 | 配对码过期 | 重启 `hermes gateway run` 拿新码 |
| 之前绑过、重建后要重新扫 | pairing 数据丢了 | 正常：`~/.hermes` 持久化后一般不需要重扫；真丢了就重走本步 |

## 步骤 6：端到端验证

### ① AI 执行

```bash
hermes -z "你好，请用一句话介绍你自己，并说出你当前使用的模型名"
```

### ② 对用户说

> 发一条端到端测试，确认"微信 → Hermes → 免费网关 → deepseek-v4.1-flash"整条链路是通的。

### ③ 用户手动做

- 在微信里给机器人发一条消息，确认能收到回复。

### ④ 做完的标志

`hermes -z` 返回正常文本且模型名为 deepseek-v4.1-flash；**用户在微信里实测收到回复**（两个都满足）。

### ⑤ 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `hermes -z` 报 401 | 走了别的 provider | 检查步骤 4 的配置是否生效；确认网关 8317 在跑 |
| 微信无回复但 `hermes -z` 正常 | 微信通道掉了 | 看 `weixin-gateway.log`；重启 `hermes gateway run` |
| 网关返回 503 / "Provider temporarily unavailable" | 网关本地熔断（旧版本 bug：上游 400"模型不存在"被误判为限流） | 源码已修（`isModelNotFound`），重编部署；**且永远不要批量探测 `/v1/models` 目录**（见铁律） |

## 步骤 7：独立保活

**目标**：网关挂了 1 分钟内自动重启；健康时静默。**耗时**：3 分钟。

> 铁律：`workbuddy-gateway-keepalive` 是用户明确要求**独立保留**的，不删除、不合并、不改名。

### ① AI 执行

用平台定时任务创建一个每 1 分钟的任务（`cron.add`，id=`workbuddy-gateway-keepalive`），
body 大意：

> 每轮 `curl -s -m 8 http://127.0.0.1:8317/health`；
> 不 healthy 则 `cd ~/workspace/workbuddy && nohup <网关二进制> serve -port 8317 > logs/serve.log 2>&1 &`，
> 重查 health，修好则静默，修不好才通知用户；
> 健康时什么都不做、不打扰用户。
> 绝不复述任何 key/token；绝不动 `hermes-keepalive-monitor`。

### ② 对用户说

> 最后加个保活：网关进程如果被沙盒杀掉，1 分钟内自动拉起来。正常时它不会打扰你。

### ③ 用户手动做

无。

### ④ 做完的标志

平台任务列表里有 `workbuddy-gateway-keepalive`（`cron.list`），schedule 为每 1 分钟；
手动 `pkill -f "workbuddy-gateway serve"` 后 1–2 分钟 `/health` 恢复 healthy。

### ⑤ 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| 任务存在但网关没被拉起 | body 里的工作目录/二进制路径写错 | 路径必须是 `~/workspace/workbuddy` + `~/workspace/projects/workbuddy-gateway/workbuddy-gateway` |
| 重启后 5xx 持续 | 凭据过期 | 跑 `workbuddy-gateway refresh`；不行就重走步骤 2 登录 |

## Output Contract（"装好了"的定义）

- `curl http://127.0.0.1:8317/health` → healthy，`model_count` > 0
- `hermes config get model.default` → `deepseek-v4.1-flash`，provider custom
- `hermes -z` 端到端正常，且明确走 deepseek-v4.1-flash
- 微信通道 connected，用户微信实测收到回复
- `workbuddy-gateway-keepalive` 平台任务存在且每 1 分钟运行
- `~/workspace/workbuddy/workbuddy-intl.json` 存在且 600 权限

## Operating Rules（铁律）

1. **密钥纪律**：`workbuddy-*.json`、`~/.hermes/.env` 权限 600；绝不在文档、skill、日志、记忆里记录
   key/token/UID 原值；`MY_API_KEY` 填 `dummy` 即可，不要编造"真实 key"。
2. **绝不批量探测 `/v1/models`**：目录里混着上游不存在的别名；旧网关曾把 400 误判为限流
   导致账号级冷却、正常模型被 503 误伤（2026-09-27 实测）。只用 `deepseek-v4.1-flash`。
3. **`discover_models` 永远 false**：不让 Hermes 合并网关的模型目录。
4. **Go 与源码放 `$HOME` 下**：`~/workspace/tools/go/`、`~/workspace/projects/workbuddy-gateway/`；
   `/usr/local/go` 会被沙盒重建清空。
5. **网关源码含本地修复**（`isModelNotFound`）：重新 clone 上游会丢掉修复，优先用本地副本编译。
6. **保活任务独立**：`workbuddy-gateway-keepalive` 不删除、不合并、不改名；
   也不要动 `hermes-keepalive-monitor`（微信网关 + 沙盒重建恢复）。
7. **沙盒重建**：进程、cron、/tmp 会丢，`$HOME` 保留；恢复走 `sandbox-keepalive-wechat` 的
   `restore-all.sh`，本手册的步骤 1/3/4 幂等可重跑。
8. **可选增强**（用户要求时再做）：Telegram 通道（token 写 `~/.hermes/.env`）、
   `~/.hermes/skills/duo/`（让 Hermes 用免费网关做 ask/critique/brainstorm，
   wrapper 里 `WORKBUDDY_API_KEY` 为空时填 dummy，并 `cd ~/workspace` 再 `python3 -m duo`）。
