# DingTalk Channel for OpenClaw

钉钉企业内部机器人 Channel 插件，使用 Stream 模式（无需公网 IP）。

本仓库 (`MyQiongbao/openclaw-channel-dingtalk`) 是 soimy/openclaw-channel-dingtalk 的 fork，在上游基础上修复了并发会话 runId 串扰问题，并新增了实时过程通知功能。

npm 包：[`@mjand66/openclaw-dingtalk`](https://www.npmjs.com/package/@mjand66/openclaw-dingtalk)

## 功能特性

- ✅ **Stream 模式** — WebSocket 长连接，无需公网 IP 或 Webhook
- ✅ **私聊支持** — 直接与机器人对话
- ✅ **群聊支持** — 在群里 @机器人
- ✅ **多种消息类型** — 文本、图片、语音（自带识别）、视频、文件、钉钉文档/钉盘文件卡片
- ✅ **引用消息支持** — 支持恢复大多数引用场景（文字/图片/图文/文件/视频/语音/AI 卡片），优先走确定性索引；`钉钉文档/钉盘文件卡片` 目前仅支持单聊引用，群聊引用暂不支持
- ✅ **Markdown 回复** — 支持富文本格式回复
- ✅ **Markdown 表格兼容** — 自动把 Markdown 表格转换为钉钉更稳定的可读文本
- ✅ **互动卡片** — 支持流式更新，适用于 AI 实时输出
- ✅ **实时过程通知** — AI 执行工具调用时实时发送进度提示，避免群聊长时间无响应
- ✅ **完整 AI 对话** — 接入 Clawdbot 消息处理管道

## 安装

### 方法 A：通过 npm 包安装（推荐）

```bash
openclaw plugins install @mjand66/openclaw-dingtalk
```

### 方法 B：通过本地源码安装

```bash
git clone https://github.com/MyQiongbao/openclaw-channel-dingtalk.git
openclaw plugins install ./openclaw-channel-dingtalk
```

## 钉钉机器人配置

在钉钉开放平台创建企业内部应用机器人，获取以下凭据：

- **Client ID**（App Key）
- **Client Secret**（App Secret）

将其填入 `openclaw.json` 或通过 `openclaw config` 命令配置：

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

## 配置项

| 配置项 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `clientId` | string | — | 钉钉应用 Client ID（必填） |
| `clientSecret` | string | — | 钉钉应用 Client Secret（必填） |
| `cardMode` | boolean | `false` | 启用互动卡片模式（流式输出） |
| `cardTemplate` | string | 内置模板 | 自定义卡片模板 ID |
| `thinkingMessage` | string | `"思考中..."` | 开始处理时发送的提示文本 |
| `maxRetries` | number | `3` | 消息发送失败时的最大重试次数 |
| `retryDelay` | number | `1000` | 重试间隔（毫秒） |
| `logLevel` | string | `"info"` | 日志级别：`debug` / `info` / `warn` / `error` |

## 消息类型支持

| 消息类型 | 私聊 | 群聊 | 说明 |
|---|---|---|---|
| 文本 | ✅ | ✅ | 普通文字消息 |
| 图片 | ✅ | ✅ | 发送图片文件 |
| 语音 | ✅ | ✅ | 自动语音识别转文字 |
| 视频 | ✅ | ✅ | 发送视频文件 |
| 文件 | ✅ | ✅ | 发送附件文件 |
| 钉钉文档/钉盘文件卡片 | ✅ | ⚠️ | 群聊引用暂不支持 |
| 引用消息 | ✅ | ✅ | 支持大多数引用场景 |

## 实时过程通知

本 fork 新增了 **实时过程通知** 功能。当 AI 正在执行工具调用时，会实时向钉钉发送当前进度，避免群聊中长时间等待无响应。

工作原理：
- 订阅 OpenClaw 全局 `onAgentEvent` 事件总线
- 收到 `tool:start` 事件时，立即推送 `"正在执行：<工具名称>"` 通知
- 每 30 秒发送一次心跳消息，避免超时无响应
- 工具调用结束后自动取消订阅，清理资源

同时修复了并发会话 runId 串扰问题：使用模块级 `claimedRunIds` Set 确保每个 `lifecycle.start` 事件只被一个 inbound handler 认领，防止多会话并发时过程通知错乱。

## 进程级（memory-only）运行态说明

以下命名空间/状态刻意保持为**仅进程内内存态**，不会进行磁盘持久化：

- `dedup.processed-message`（消息去重窗口）
- `session.lock`（同 session 串行锁）
- `channel.inflight`（gateway in-flight 防重锁）

这样设计是为了保证并发控制语义简单且可预期，避免跨进程/重启后引入锁状态不一致问题。

## 学习命令

插件支持以下内置学习指令（通过 `/learn` 前缀触发）：

- `/learn help` — 查看所有可用命令
- `/learn reset` — 清空当前会话上下文
- `/learn history` — 查看历史消息记录

## 常见问题

### 机器人无响应

1. 检查 `clientId` / `clientSecret` 是否正确
2. 确认钉钉开放平台已开启 Stream 模式
3. 查看网关日志：`tail -f ~/.openclaw/logs/gateway.log`

### 群聊中 @机器人 无响应

确认以下设置：
- 钉钉应用已添加到对应群组
- 机器人已被授权该群的消息权限
- 检查日志中是否有 `atUsers` 解析相关错误

### 实时进度通知不显示

本功能仅在非卡片模式（`cardMode: false`）下通过独立文本消息发送进度。若启用了卡片模式，AI 输出会直接流式更新到卡片内容中。

## 开发

```bash
git clone https://github.com/MyQiongbao/openclaw-channel-dingtalk.git
cd openclaw-channel-dingtalk
pnpm install

# 格式检查
pnpm format:check

# 类型检查
pnpm type-check

# 运行测试
pnpm test

# 代码风格修复
pnpm lint:fix
```

## 许可证

MIT © [mjand66](https://github.com/MyQiongbao)
