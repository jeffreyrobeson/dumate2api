# dumate2api 部署指南（Linux 服务器 + systemd）

> 本指南记录一次在 Linux 服务器上的完整部署过程：从克隆仓库到网关与管理端以 systemd 服务运行、登录 Qoder 账号、签发网关 Key，并接入 Magpie 使用。文中所有服务器地址、域名、密钥、密码、邮箱均为占位符，请替换为你自己的值。**不要把真实地址、账号或密钥写进任何公开仓库。**

## 适用范围

- 服务器：任意 Linux 服务器（本指南以 Debian / Ubuntu 为例），能出站访问互联网。
- 客户端：Qoder / 千问办公等订阅账号（用于提供模型额度）。
- 下游：任意 OpenAI / Anthropic 兼容客户端，本指南以 Magpie 为例。

## 架构一览

```
                        ┌─────────────────────┐
                        │   dumate2api 网关    │
  客户端 (Codex /       │  http://127.0.0.1:9080  │
  Claude Code / Magpie) │  /v1/chat/completions │
  ─────────────────────▶│  /v1/responses      │────▶ Qoder / 千问办公 /
                        │  /v1/messages       │      TRAE Work / DuMate /
                        │  /v1/models         │      WorkBuddy（账号额度）
                        └─────────────────────┘
                        ┌─────────────────────┐
                        │   dumate2api 管理端  │
                        │  http://127.0.0.1:9081  │
                        │  账号管理 / API Key / │
                        │  用量统计 / 聊天测试  │
                        └─────────────────────┘
```

模型名前缀决定路由：`qoder/` → Qoder，`qwen/` → 千问办公，`traework/` → TRAE Work，无前缀 → DuMate 默认通道。

## 1. 环境要求

- **Node.js 20+**（推荐 22）。检查：`node -v`、`npm -v`。
- **Git**。
- 一台能长期运行的 Linux 服务器（网关需要一直跑着，客户端才能调用）。
- 一个 Qoder 或千问办公账号（个人版即可；Qoder 免费版、Pro 试用版均可）。

## 2. 安装

### 2.1 克隆仓库

```bash
git clone https://github.com/<你的 GitHub 用户名>/dumate2api.git
cd dumate2api
```

### 2.2 安装网关依赖

网关本身接近零依赖（只用 Node 内置模块），根目录安装即可：

```bash
npm install
```

> 仓库锁文件是 `pnpm-lock.yaml`，也可以用 `pnpm install`。两种方式均可，任选其一。

### 2.3 构建管理端前端（关键一步，不可跳过）

管理端是一个 Vue 应用，首次使用前必须构建，否则访问管理端会 404：

```bash
cd web && npm install && npm run build
cd ..
```

构建产物在 `web/dist/`。如果之后管理端出现 404，先检查 `web/dist/index.html` 是否存在，不存在就重新执行上面这条命令。

## 3. 以 systemd 服务运行（推荐）

让网关和管理端开机自启、异常自动重启。以下以 Debian / Ubuntu + systemd 为例，路径请按你的实际安装位置修改。

### 3.1 网关服务

新建 `/etc/systemd/system/dumate2api-gateway.service`：

```ini
[Unit]
Description=dumate2api gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/dumate2api
ExecStart=/usr/bin/node src/server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.2 管理端服务

新建 `/etc/systemd/system/dumate2api-admin.service`：

```ini
[Unit]
Description=dumate2api admin panel
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/dumate2api
ExecStart=/usr/bin/node src/admin/server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.3 启动并验证

```bash
systemctl daemon-reload
systemctl enable --now dumate2api-gateway.service dumate2api-admin.service
systemctl status dumate2api-gateway.service dumate2api-admin.service
```

验证网关与管理端：

```bash
# 网关健康检查（不需要鉴权）
curl -s http://127.0.0.1:9080/health

# 管理端应返回 200（HTML 页面）
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:9081/
```

`/health` 返回示例（通道未登录时 `ready` 为 `false` 属正常）：

```json
{
  "status": "ok",
  "channels": {
    "qoder": { "ready": false, "accounts": 0 },
    "qwenwork": { "ready": false, "accounts": 0 }
  }
}
```

## 4. 首次配置：登录账号与签发网关 Key

### 4.1 登录管理端

用浏览器打开管理端：

- 本机：`http://127.0.0.1:9081`
- 远程：见第 5 节（通过隧道暴露，或 SSH 端口转发）。

首次登录用户名为 `admin`。密码是服务首次启动时在日志里打印的一次性生成密码（只显示一次，登录后请立即修改）。查看方式：

```bash
journalctl -u dumate2api-admin.service --no-pager | head -30
```

> 如果密码丢失，可重置：`node src/admin/server.js --reset-admin --password=<新密码>`，然后重启管理端服务。

### 4.2 登录 Qoder 账号

在管理端左侧进入「账号管理」页面：

1. 顶部切换到 **Qoder** 通道（页面标题为「Qoder 登录态（凭证自持，无需客户端）」）。
2. 点击「添加账号」（列表为空时按钮显示为「添加第一个账号」）。
3. 在弹出的「添加 Qoder 账号」窗口中点击「生成登录链接」，再点击「① 点此打开登录页 →」。
4. 在打开的页面中登录你的 Qoder 账号（国际版与国内版对应不同域名，按你的账号区域选择）。
5. 登录成功后浏览器会跳转到一个打不开的页面，这是正常的。回到弹窗，点击「已登录，开始取票」，等待轮询完成（约 1 分钟内），账号即保存成功。

登录后，「账号管理」会显示账号昵称、套餐与区域。**建议登录 2 个或以上账号**：单个账号在连续遇到 3 次上游 5xx 后会进入约 30 分钟冷却，期间该通道无可用账号；多账号可自动轮换。

> 千问办公的登录方式不同：其请求体需要官方客户端的 wasm，登录态需从客户端的 `auth-v2.dat` 导入，或在同一「账号管理」页走设备授权流程。详见 `docs/CHANNEL-QWENWORK.md`。

### 4.3 签发网关 Key

网关默认开启鉴权（`require_key: true`），没有 Key 的请求会被拒绝。签发步骤：

1. 在管理端左侧进入「API Key」页面。
2. 点击右上角「新建 Key」。
3. 创建成功后会弹出一次完整 Key（以 `dmk_` 开头），点击「复制」保存好。**明文只显示这一次，关闭后无法再查看**，丢失只能重建。

这个 Key 就是下游客户端（Magpie、Codex 等）调用网关时用的 `Authorization: Bearer <Key>`。

## 5. 远程访问管理端（可选）

网关和管理端默认只监听 `127.0.0.1`。如需从其他电脑打开管理端，推荐以下任一方式（不要把管理端直接暴露到公网且不设密码）：

- **SSH 端口转发**（最简单）：`ssh -L 9081:127.0.0.1:9081 <你的服务器>`，然后本地打开 `http://127.0.0.1:9081`。
- **Cloudflare Tunnel**：在隧道配置里加一条路由，把你的域名（如 `https://dumate-admin.<你的域名>`）指向 `http://localhost:9081`，并给该域名配好 DNS 记录。

## 6. 接入 Magpie

以 Magpie 为例，把 dumate2api 作为一个 OpenAI 兼容 provider 接入：

| 字段 | 值 |
|---|---|
| Provider ID | `dumate2api` |
| 协议（baseAPI） | `chat`（OpenAI 兼容） |
| Chat 地址 | `http://127.0.0.1:9080/v1` |
| Responses 地址 | `http://127.0.0.1:9080/v1` |
| Anthropic 地址 | `http://127.0.0.1:9080` |
| Key | 第 4.3 节签发的 `dmk_` Key |

在 Magpie 的 `providers.json` 里添加该 provider 后，重启 Magpie，网关模型列表里会出现 `dumate2api/qoder/*` 等模型。验证：

```bash
# 模型列表应包含 dumate2api/qoder/*（以 Magpie 自身的鉴权方式调用）
curl -s -H "Authorization: Bearer <Magpie 网关 Key>" \
  http://127.0.0.1:3425/v1/models | grep -o "dumate2api/qoder/[^\"]*"

# 实测一次对话（用小 max_tokens，避免消耗额度）
curl -s -H "Authorization: Bearer <Magpie 网关 Key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"dumate2api/qoder/GLM-5.3-Flash","messages":[{"role":"user","content":"hi"}],"max_tokens":16}' \
  http://127.0.0.1:3425/v1/chat/completions
```

Magpie 的网关地址与鉴权 Key 见 Magpie 自身的「网关」设置页；注意 Magpie 本地调用与远程调用的 Key 可能不同（远程调用需用「网关 Key」而非本地 bearer）。

## 7. 模型说明与已知问题

### 7.1 Qoder 模型

Qoder 通道登录后，网关提供以下模型（以实际 `/v1/models` 返回为准）：

`qoder/Auto`、`qoder/Ultimate`、`qoder/Fast`、`qoder/Efficient`、`qoder/Sonus`、`qoder/Cantus`、`qoder/Qwen4-Max`、`qoder/Qwen3.8-Max`、`qoder/Qwen3.8-Flash`、`qoder/Kimi-K3.1-Pro`、`qoder/Kimi-K2.8-Preview`、`qoder/GLM-5.4`、`qoder/GLM-5.3`、`qoder/GLM-5.3-Flash`、`qoder/DeepSeek-V4-Pro`、`qoder/DeepSeek-Flash`、`qoder/MiniMax-M3.1-Flash`。

### 7.2 `qoder/Auto` 不可用

`qoder/Auto` 调用会返回 `400: All models failed`，这是 dumate2api 侧自动选模型的问题，与账号无关。请使用具体模型名（如 `qoder/GLM-5.3-Flash`、`qoder/Kimi-K3.1-Pro`），不要用 `Auto`。

### 7.3 HungerG（Qoder 匿名模型）

HungerG 是 Qoder 的匿名模型，2026 年 11 月 23 日至 30 日对个人用户免费（仅限 Qoder / Qoder IDE 客户端内使用，活动结束后可能收费或下线）。

注意：**HungerG 不在 `/v1/models` 返回的列表里**，但直接按名调用是通的：

```bash
curl -s -H "Authorization: Bearer <dmk_ Key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"qoder/HungerG","messages":[{"role":"user","content":"hi"}],"max_tokens":16}' \
  http://127.0.0.1:9080/v1/chat/completions
```

实测返回正常，且 `credits: 0`、`billable: false`。如需在 Magpie / Codex 里使用，把 `qoder/HungerG` 手动加入 provider 的模型列表即可。**活动结束后请及时检查它是否开始计费或失效**，如已失效应从模型列表中移除。

## 8. 常见问题

| 现象 | 原因与处理 |
|---|---|
| 管理端打开是 404 | `web/dist` 未构建。执行 `cd web && npm install && npm run build` 后重启管理端服务。 |
| 网关返回 `401` / 要求 Key | 网关开启了 `require_key`。用第 4.3 节签发的 `dmk_` Key 作 `Bearer` 调用；不要用 `nokey` 占位符。 |
| `qoder/Auto` 报 `All models failed` | 见 7.2，改用具体模型名。 |
| 通道显示 `ready: false`，账号数为 0 | 尚未登录账号，按第 4.2 节登录。 |
| 请求报「没有可用账号」 | 账号进入冷却（连续 3 次上游 5xx 后约 30 分钟）。登录多个账号可轮换；也可等待冷却结束。 |
| 服务启动后端口未监听 | 检查 `systemctl status` 与日志：`journalctl -u dumate2api-gateway.service -n 50`。确认 Node 版本 ≥ 20 且端口未被占用。 |

## 9. 安全提醒

- **不要把网关或管理端无鉴权地暴露到公网。** 网关默认只监听本地；如需远程访问，用 SSH 隧道或带鉴权的反向代理。
- **密钥与密码只保存在服务器本地与你自己的密码管理器里**，不要写进代码、文档或公开仓库。`dmk_` Key、Magpie Key、管理端密码、账号 Token 一律按敏感信息处理。
- **不要把真实服务器 IP、域名、邮箱、密钥写进提交到 GitHub 的文档**，用占位符（如 `<你的服务器>`、`<你的域名>`、`<dmk_ Key>`）代替，提交前做一次全文敏感信息检查。
- 多账号登录后，凭证保存在服务器 `data/` 目录下（如 `data/qoder-accounts.json`），请做好服务器本身的访问控制与备份。

## 10. 参考

- 项目主页与用户手册：`README.md`、`docs/USER_MANUAL.md`
- Qoder 通道说明：`docs/CHANNEL-QODER.md`
- 千问办公通道说明：`docs/CHANNEL-QWENWORK.md`
- 架构说明：`ARCHITECTURE.md`
