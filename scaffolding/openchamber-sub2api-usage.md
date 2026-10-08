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

## 最终方案

使用原版 OpenChamber 的独立扩展，在右侧「工作状态 → Sub2API」显示
一把网关 key 的今日实际费用；第二把 key 由独立的
「Sub2API Relay」栏目显示。右侧扩展栏也可以打开相应面板。

文档中的两把 key 名称分别以 `example-key-a`、`example-key-b` 脱敏。
每个面板各自使用独立的扩展集成凭证，因此展示的费用互不混合。

不修改 `opencode.json`，不修改 OpenChamber 安装包、签名或升级流程。
扩展使用公开 SDK 的 `apiVersion: 1`，当前最低宿主版本为 `2.1.1`。

`example-key-a` / `example-key-b` 代表两把不同密钥的显示名称，不是分组名称。
统计范围分别是各自的 key，
不是账户总消费，也不是同分组下其他 key 的合计。

## 数据来源与口径

网关管理页面（域名已脱敏）：<https://gateway.example.com/keys>。

实际读取接口：

```http
GET https://gateway.example.com/v1/usage?timezone=Asia%2FShanghai&days=1
Authorization: Bearer <由 OpenChamber 在服务端附加>
```

只显示返回值中的 `usage.today.actual_cost`，即今日实际扣费，保留四位小数。
不使用原价字段 `cost`、累计字段 `usage.total` 或账户余额 `balance`。

- 按 `Asia/Shanghai` 的自然日统计。
- 页面可见时每 300 秒（5 分钟）刷新一次，也支持手动刷新。
- 请求失败不显示成 `$0.0000`，而是标记失败并保留当天最后一次成功结果。
- 上海时间跨日后清除前一天的旧值，避免把昨日费用标为今日用量。
- 金额来自实时查询，不把某次观察到的 `$0.2463` 或 `$1.6085` 写死。

### 刷新时机的取舍

**当前实现为可见时每 300 秒（5 分钟）轮询，只调整了间隔。以下事件驱动优化建议尚未实施。**

初版的 60 秒不是网关要求，而是为简单实现、兼顾同一 key 在其他工具中的消费而选的折中；现已按使用需求改为 300 秒。
它会在没有新增消费时仍然查询，也不能保证一轮对话结束后立即更新。

更合适的策略是事件优先、低频轮询兜底：

1. 首次打开面板、从后台回到前台时刷新。
2. 当前会话从忙碌转为空闲时，延迟约 3 秒刷新，并合并短时间内的重复触发。
   SDK 的 `onSessionLifecycle` 可提供此信号；`completed` 表示空闲，不等同于业务成功。
3. 面板可见时每 5 分钟兜底查询，覆盖其他工具消费、其他会话及遗漏的事件。
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

最终采用宿主已有的扩展能力：

- `contributes.panel`：提供扩展栏面板。
- `contributes.statusSection`：在工作状态面板添加 Sub2API 栏目。
- `contributes.integration.token`：由宿主管理 bearer API key。
- `host.request()`：通过宿主服务端向指定网关发请求。

扩展文件、登记信息及集成凭证均位于用户数据目录，官方正常升级不会覆盖它们。
如果未来 SDK 出现不兼容变更，只更新扩展，不重新打包客户端。

## 文件与配置位置

两个扩展目录：

```text
~/.config/openchamber/extensions-local/sub2api-usage/
├── package.json          # 面板、工作状态栏目、集成和构建命令
├── README.md             # 安装与维护说明
├── validation.json       # 验证及原应用完整性记录
├── panel/
│   ├── index.html        # 面板结构与样式
│   ├── main.ts           # SDK 连接、刷新、展示与错误状态
│   ├── main.js           # 打包后的独立 IIFE，宿主直接加载
│   ├── usage.ts          # 响应解析与上海自然日计算
│   └── usage.test.ts     # 用量口径与异常数据测试
└── bun.lock              # 开发依赖锁文件

~/.config/openchamber/extensions-local/sub2api-relay-usage/
├── package.json          # 独立的 relay 集成与工作状态栏目
└── panel/                # 独立面板源码、bundle 与测试
```

宿主登记与凭证：

| 文件 | 用途 |
| --- | --- |
| `~/.config/openchamber/extensions.json` | 本地文件夹扩展登记、授权及启用状态 |
| `~/.config/openchamber/guest-auth.json` | 官方集成机制保存的 API key 与集成设置，权限为 `0600` |

扩展 ID 分别为 `sub2api-usage` 和 `sub2api-relay-usage`。两个集成各自保存
一把 key，配置中的 `key-name` 只用于显示。relay key 由 API02 管理页面
复制后，经本机 OpenChamber 集成接口写入凭证库，不经过源码、命令行参数或本文。

以后轮换 API key，需要在「设置 → 集成 → Sub2API」更新；
扩展不会自动跟随 `opencode.json` 的 key 变化。

## 凭证与权限

数据流：

```text
扩展 iframe
  → SDK host.request()
  → 原版 OpenChamber 集成代理
  → 服务端附加 API key
  → https://gateway.example.com/v1/usage
  → 扩展只渲染今日实际费用
```

扩展只申请访问所声明外部服务的 `network` 权限。
没有申请聊天内容、文件系统、命令执行或模型调用权限。
API key 不写进扩展源码或页面，登录邮箱和密码也没有保存到扩展或本文。

## 安装、刷新与卸载

本机已通过原版宿主完成安装和授权：

1. 「设置 → 扩展」分别添加上述两个扩展文件夹。
2. 审核两项访问网关的权限并启用。
3. 「设置 → 集成 → Sub2API」和「设置 → 集成 → Sub2API Relay」分别连接对应 key。
4. 在各自集成设置中设置脱敏占位为 `example-key-a` 和 `example-key-b` 的显示名。
5. 刷新 OpenChamber 界面。如果栏目被隐藏，在工作状态面板的「选择栏目」中启用两个栏目。

样式文件通过本地文件夹安装直接加载。修改 `panel/index.html` 后刷新界面即可，
无需重编译或重新签名客户端。修改 TypeScript 后执行：

```bash
cd ~/.config/openchamber/extensions-local/sub2api-usage
bun install
bun run build
bun test
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

- 16 项用量解析、异常输入和跨日计算测试通过。
- 已在原版宿主中验证实时金额、手动刷新、自动刷新及失败后保留旧值。
- 已检查实际渲染的背景透明度与金额字号，而不是只检查 CSS 源码。
- 未执行官方升级测试；已确认实现不修改安装包、签名或更新器。

## 相关页面

- [[harness-engineering]]：优先使用宿主提供的扩展接口，避免为小功能维护定制客户端。
