---
title: 在 OpenChamber 显示 Sub2API 今日用量
type: topic
date: 2026-10-08
tags: [ai-engineering, harness, infrastructure]
sources:
  - https://github.com/openchamber/openchamber/blob/v2.1.1/packages/sdk/DOCUMENTATION.md
  - https://github.com/openchamber/openchamber/blob/v2.1.1/packages/docs/content/docs/extensions.mdx
related: [harness-engineering]
---

# 在 OpenChamber 显示 Sub2API 今日用量

> 脱敏说明：网关域名统一使用 `gateway.example.com`，实际 key 名称统一使用
> `example-key`。它们是文档占位符，不能直接用于连接；运行配置没有因此改动。

## 最终实现

已在原版 OpenChamber 的一个 Sub2API 面板中同时显示两把网关 key
各自的今日实际费用。面板经 OpenChamber 的本地 Service 调用网关，
分别用对应 API key 请求一次 `/v1/usage`。

文档中的两把 key 名称分别以 `example-key-a`、`example-key-b` 脱敏。

不修改 `opencode.json`，不修改 OpenChamber 安装包、签名或升级流程。
扩展使用公开 SDK 的 `apiVersion: 1`，当前最低宿主版本为 `2.1.1`。

`example-key-a` / `example-key-b` 代表两把不同密钥的显示名称，不是分组名称。
统计范围必须分别限制在各自的 key，
不是账户总消费，也不是同分组下其他 key 的合计。

## 数据来源与口径

网关管理页面（域名已脱敏）：<https://gateway.example.com/keys>。

对两把 key 分别发出以下请求（域名已脱敏）：

```http
GET https://gateway.example.com/v1/usage?days=1&timezone=Asia%2FShanghai
Authorization: Bearer <ID 1 对应的 API key>

GET https://gateway.example.com/v1/usage?days=1&timezone=Asia%2FShanghai
Authorization: Bearer <ID 2 对应的 API key>
```

每次只显示该 key 返回的 `usage.today.actual_cost`，即今日实际扣费，保留四位小数。
不使用累计字段 `usage.total` 或账户余额，也不合并两把 key 的费用。

| key ID | 显示名称（脱敏） | 读取字段 |
| --- | --- | --- |
| 2 | `example-key-a` | `usage.today.actual_cost`，以 ID 2 的 key 认证 |
| 1 | `example-key-b` | `usage.today.actual_cost`，以 ID 1 的 key 认证 |

Service 并发分别请求这两个固定 key，逐个验证响应。某个 key 失效或响应错误时，
只标记该行，不清除另一行的有效数据，也不把缺失值当作零。

Codex 7d 行通过 Sub2API 管理接口 `GET /api/v1/admin/accounts/{account_id}/usage?source=active`
读取 `data.seven_day.utilization`，对应 Codex 的 `codex_7d_used_percent`；同时读取
`remaining_seconds`，在标签中显示距周重置时间（如 `4d 18h`）。百分比和到期时间只在同次刷新时更新。
需要管理员
access/refresh token 和上游账号 ID；本文和 `.env.example` 只用占位符，不记录实际账号 ID 或 token。
该 endpoint 是 active usage：缓存快照缺失或过期时可能访问 Codex 上游并更新缓存，并非严格只读。
管理 access token 有效期为 24 小时。Service 通过 `/api/v1/auth/refresh` 自动续期，并原子更新
`.env` 中的 access/refresh token；refresh token 被撤销或过期时需要重新登录。

- 网关已配置 `Asia/Shanghai` 自然日口径，面板也以该时区判断跨日。
- 本地 Service 会将两把 key 的网关结果缓存 600 秒，缓存按 `.env` 内容指纹隔离。
- 会话/iframe 重载可能再次访问 loopback Service，但缓存有效期间不会再次访问 Sub2API。
- 面板初次打开和从后台回到前台时刷新，此后每 600 秒刷新；手动刷新会绕过缓存立即请求两把 key。
- 请求失败不显示成 `$0.0000`，而是标记失败并保留当天最后一次成功结果。
- 上海时间跨日后清除前一天的旧值，避免把昨日费用标为今日用量。
- 金额来自实时查询，不把某次观察到的 `$0.2463` 或 `$1.6085` 写死。

### 刷新时机的取舍

**当前实现为可见时每 600 秒（10 分钟）轮询。以下事件驱动优化建议尚未实施。**

初版的 60 秒不是网关要求，而是为简单实现、兼顾同一 key 在其他工具中的消费而选的折中；现已按使用需求改为 600 秒。
它会在没有新增消费时仍然查询，也不能保证一轮对话结束后立即更新。

更合适的策略是事件优先、低频轮询兜底：

1. 首次打开面板、从后台回到前台时刷新。
2. 当前会话从忙碌转为空闲时，延迟约 3 秒刷新，并合并短时间内的重复触发。
   SDK 的 `onSessionLifecycle` 可提供此信号；`completed` 表示空闲，不等同于业务成功。
3. 面板可见时每 10 分钟兜底查询，覆盖其他工具消费、其他会话及遗漏的事件。
4. 保留手动刷新；上海时间跨日时使昨日数值失效并重新查询。

不能只依赖当前会话事件：它不能代表这把 key 在所有客户端、所有会话中的使用情况。
延迟刷新也不保证网关已完成入账，因此仍需要兜底查询。

## 为什么用独立扩展

### 不采用 `opencode.json.metrics`

本次核查的 OpenCode `v2.0.24` 没有可用的 `metrics` 自定义端点配置。
OpenChamber 的内置配额列表也不是从这个假定字段动态加载的。
因此没有向 `opencode.json` 添加无效的 `metrics` 字段。

### 不采用定制客户端

**修改或重新打包 OpenChamber 客户端，是本次需求的错误方向。**

即使能实现，也会引入签名、分发、官方升级覆盖和持续合并上游代码的维护成本。
仅显示一个今日费用指标，不值得维护整套定制客户端。

## 从两个面板修正为一个面板

最初只使用 `/v1/usage`，该接口用模型 key 查询自身用量，因而错误地认为
需要两份扩展集成和两个面板。

用户指出两个 key 可以各自请求 `/v1/usage`。单个 guest 只能安全地使用一份
OpenChamber integration credential，无法让 iframe 安全切换第二个 bearer token。
第二把 key 不放入会传给 iframe 的 integration settings。

当前单面板通过 OpenChamber local Service 分别向 `/v1/usage` 发送两个请求。
两把 key 放入本机 `.env`，Service 只把 key ID、显示名、今日费用和错误状态返回面板。

最终采用宿主已有的扩展能力：

- `contributes.panel`：提供扩展栏面板。
- `contributes.statusSection`：在工作状态面板添加 Sub2API 栏目。
- `contributes.service`：由宿主按需启动 loopback-only Node 服务。
- `host.serviceRequest()`：通过宿主代理访问本地服务，不向 iframe 暴露服务端口或 service token。

扩展文件、登记信息及集成凭证均位于用户数据目录，官方正常升级不会覆盖它们。
如果未来 SDK 出现不兼容变更，只更新扩展，不重新打包客户端。

## 文件与配置位置

当前安装目录：

```text
~/.config/openchamber/extensions-local/sub2api-usage/
├── package.json          # 面板、工作状态栏目、集成和构建命令
├── README.md             # 安装与维护说明
├── validation.json       # 验证及原应用完整性记录
├── .gitignore             # 排除本地 .env 和 node_modules
├── .env.example           # 无 secret 的配置模板，可提交 GitHub
├── .env                   # 网关 origin 与两把 key，0600，永不提交
├── panel/
│   ├── index.html        # 面板结构与样式
│   ├── main.ts           # SDK 连接、刷新、展示与错误状态
│   ├── main.js           # 打包后的独立 IIFE，宿主直接加载
│   ├── usage.ts          # 响应解析与上海自然日计算
│   └── usage.test.ts     # 用量口径与异常数据测试
├── service/main.js       # 分别使用两个 key 查询网关的本地服务
└── bun.lock              # 开发依赖锁文件

```

只保留 `sub2api-usage` 一个扩展登记和一个工作状态栏目；冗余 Relay 扩展已移除。

宿主登记与凭证：

| 文件 | 用途 |
| --- | --- |
| `~/.config/openchamber/extensions.json` | 本地文件夹扩展登记、授权及启用状态 |
| `~/.config/openchamber/extensions-local/sub2api-usage/.env` | 网关 origin 与两把 API key，权限 `0600`，由 `.gitignore` 排除 |

扩展 ID 为 `sub2api-usage`。ID 和显示名从本机 `.env` 读取。Service 返回显示名
和用量，但不会把 API key 返回给 iframe。

key 轮换时更新 `.env` 并保留 `0600` 文件权限；Service 每次请求都会重新读取配置，
手动刷新面板即可生效。

### Service 权限

Service 仅监听 `127.0.0.1`，通过宿主生成的 bearer token 接收代理请求，并固定访问
网关的 `/v1/usage` endpoint。

**OpenChamber guest Service 没有 OS sandbox。** 它以当前用户权限运行，manifest 不能限制
它读取用户的其他文件。当前实现只读取扩展目录的 `.env`，不执行命令、不记录密钥，
也不把密钥返回给面板。批准前应检查 `service/main.js`。

`.env` 还保存 Codex 管理 access/refresh token 和账号 ID。access token 临近过期或收到 401 时，
Service 调用 `/api/v1/auth/refresh`，并以原子写入更新两个 token，保持 `.env` 的 `0600` 权限。
登录密码不存盘。refresh token 被撤销或过期时需重新登录并更新 `.env`。

## 凭证与权限

数据流：

```text
扩展 iframe
  → SDK host.serviceRequest()
  → 原版 OpenChamber loopback proxy
  → extension Service 读取本机 .env
  → GET https://gateway.example.com/v1/usage（ID 1 key）
  → GET https://gateway.example.com/v1/usage（ID 2 key）
  → 只返回今日实际费用及状态
```

扩展申请 `service` 权限，不申请聊天内容、模型调用、文件 API 或 shell command grant。
Service 本身没有 OS sandbox，因此进程实际权限仍是当前用户权限。
API key 不在扩展源码、bundle 或 iframe 中；登录密码也不保存在项目中。

## 安装、刷新与卸载

本机已通过原版宿主完成安装和授权：

单个 Sub2API 面板已安装。先把 `.env.example` 复制为 `.env`，填入两把 key 并运行
`chmod 600 .env`，然后在「设置 → 扩展」检查 Service 权限并选择允许启用。

样式文件通过本地文件夹安装直接加载。修改 `panel/index.html` 后刷新界面即可，
无需重编译或重新签名客户端。修改 TypeScript 后执行：

```bash
cd ~/.config/openchamber/extensions-local/sub2api-usage
bun install
bun run build
bun test
node --check service/main.js
```

禁用或卸载使用「设置 → 扩展 → Sub2API」。本地文件夹安装的卸载
只移除宿主登记，源码文件夹仍保留；集成凭证由宿主管理。

当前支持原版 Web 和桌面版；不在 OpenCode 终端、VS Code 或移动版添加此卡片。

## 面板样式收尾

2026-10-08 完成以下调整：

- key 名称、今日用量标签、金额以及内容容器均显式使用透明背景，不绘制单独的黑色底块。
- 金额字号由 `23px` 缩小为 `14px`，保留等宽数字和四位小数。
- 文本颜色继续使用宿主主题。透明指没有独立背景，不是把深色主题的宿主面板改成白色。
- 保留 SDK 设置的宿主明暗模式；强行改成 `color-scheme: normal` 会让深色宿主中的 iframe 变成不透明白底，因此不采用。

## 错误方向的清理与原应用核验

此前尝试制作过独立的本地定制副本。**原版
`/Applications/OpenChamber.app` 从未被覆盖或改写。**
因此准确的收尾是清理试验副本及其文件，而不是用另一个包“还原”原版。

已停止并删除：

- `~/Applications/OpenChamber Sub2API.app`：定制测试应用。
- `~/.config/openchamber/sub2api/`：临时源码克隆、构建脚本及隔离测试配置。
- `~/Library/Application Support/OpenChamber Sub2API/`：隔离的 Chromium 测试资料。
- `~/.config/openchamber/quota/sub2api.json`：旧方案的专用配置。

2026-10-08 再次核验：上述路径均不存在，也没有定制测试应用进程。
原版 `app.asar` 的 SHA-256 与试验前记录完全一致：

```text
9b61ff1b441b69159a82f65f538ee104b1be2336f3db0d4a5f026ff6d8e53495
```

原版签名校验通过：

```bash
codesign --verify --deep --strict /Applications/OpenChamber.app
```

后续如果官方升级，`app.asar` 的哈希正常会改变；上面的值是本次
清理核验的基线，不应拿它判断升级后的官方版本是否正常。

## 验证结果

- 两 key 版本的 18 项解析、异常输入、部分缺失和跨日计算测试通过。
- 已在原版宿主中验证一块面板显示两行实时金额，每次刷新分别请求两个 key。
- 已验证某行缺失时保留该行旧值、另一行仍可更新；有效的零费用正常显示为 `$0.0000`。
- `.env` 权限检查要求 `0600`，且 `.gitignore` 明确排除该文件。
- 已检查实际渲染的背景透明度与金额字号，而不是只检查 CSS 源码。
- 未执行官方升级测试；已确认实现不修改安装包、签名或更新器。

## 相关页面

- [[harness-engineering]]：优先使用宿主提供的扩展接口，避免为小功能维护定制客户端。
