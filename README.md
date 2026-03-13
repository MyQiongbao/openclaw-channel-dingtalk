# DingTalk Channel for OpenClaw

钉钉企业内部机器人 Channel 插件，使用 Stream 模式（无需公网 IP）。

npm 包：[`@mjand66/openclaw-dingtalk`](https://www.npmjs.com/package/@mjand66/openclaw-dingtalk)

---

## 核心改进：AI 任务实时进度推送

> 本仓库 fork 自 [soimy/openclaw-channel-dingtalk](https://github.com/soimy/openclaw-channel-dingtalk)，最核心的改进是将钉钉机器人的回复模式从**任务完成后统一返回**升级为**实时推送执行进度**。

### 原有问题

上游版本中，当 AI 执行复杂任务（如调用多个工具、搜索、读写文件等）时，钉钉用户需要等待所有步骤全部完成后，才能收到一条最终回复。在耗时较长的任务（30 秒～几分钟）场景下，用户完全不知道 AI 在做什么，群聊中尤为明显。

### 本 fork 的方案

**订阅 OpenClaw 全局 Agent 事件总线**，在任务执行过程中实时向钉钉推送进度消息：

```
用户: 帮我分析最近一周的销售数据并生成报告

机器人: 思考中...
机器人: 🔧 正在执行：read_file
机器人: 🔧 正在执行：search_database
机器人: 🔧 正在执行：generate_chart
机器人: [最终完整报告]         ← 任务完成后统一回复
```

实现细节：
- 收到用户消息后立即发送"思考中..."提示
- 监听 `tool:start` 事件，每次工具调用开始时推送 `🔧 正在执行：<工具名>` 通知
- 每 30 秒发送一次心跳，防止长任务期间用户误以为机器人已挂起
- 任务结束后自动清理事件订阅

### 同步修复：并发会话 runId 串扰

多用户同时向机器人发消息时，上游代码存在竞态条件——多个并发 handler 会争抢同一个 `lifecycle.start` 事件的 `runId`，导致进度通知发送到错误的会话。本 fork 通过模块级 `claimedRunIds` Set 保证每个 Agent 运行的生命周期事件只被一个 handler 独占认领。

---

## 安装

```bash
openclaw plugins install @mjand66/openclaw-dingtalk
```

或从源码安装：

```bash
git clone https://github.com/MyQiongbao/openclaw-channel-dingtalk.git
openclaw plugins install ./openclaw-channel-dingtalk
```

## 配置

在钉钉开放平台创建企业内部应用机器人，获取 Client ID 和 Client Secret，填入 `openclaw.json`：

```json
{
  "channels": {
    "dingtalk": {
      "clientId": "YOUR_CLIENT_ID",
      "clientSecret": "YOUR_CLIENT_SECRET"
    }
  }
}
```

| 配置项 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `clientId` | string | — | 钉钉应用 Client ID（必填） |
| `clientSecret` | string | — | 钉钉应用 Client Secret（必填） |
| `cardMode` | boolean | `false` | 启用互动卡片模式（流式输出） |
| `thinkingMessage` | string | `"思考中..."` | 任务开始时的提示文本 |
| `logLevel` | string | `"info"` | 日志级别：`debug` / `info` / `warn` / `error` |

## 其他功能

- **Stream 模式** — WebSocket 长连接，无需公网 IP 或 Webhook
- **私聊 & 群聊** — 支持直接对话和群内 @机器人
- **多种消息类型** — 文本、图片、语音（自动识别）、视频、文件、钉钉文档/钉盘文件卡片
- **引用消息** — 支持大多数引用场景，`钉钉文档/钉盘文件卡片` 群聊引用暂不支持
- **Markdown 回复** — 自动将 Markdown 表格转为可读文本格式
- **互动卡片** — 支持流式更新

> 实时进度通知仅在 `cardMode: false`（默认）时生效。启用卡片模式后，AI 输出会直接流式更新到卡片中。

## 常见问题

**机器人无响应** → 检查 `clientId` / `clientSecret` 是否正确，确认开放平台已开启 Stream 模式，查看日志：`tail -f ~/.openclaw/logs/gateway.log`

**群聊 @机器人 无响应** → 确认应用已添加到该群并授权消息权限

## 许可证

MIT © [mjand66](https://github.com/MyQiongbao)
