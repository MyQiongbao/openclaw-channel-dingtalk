# DingTalk Stream 上游吞吐排查报告（2026-03-02）

## 排查范围

本文档记录了在受控条件下，对 DingTalk Stream 上游回调投递吞吐进行的排查结果。重点是确认消息缺失发生在本地回调处理之前还是之后。

## 排查目标

- 判断消息丢失发生在本地 SDK 回调接收前还是接收后。
- 排除本地插件处理、ACK 返回、连接异常等因素。
- 对比 SDK keepalive 开关对接收率的影响。

## 环境与控制条件

- Channel 插件：`@soimy/dingtalk`（本地工作区）
- Stream SDK：本地 fork 版本，已加入 `DWClient` 调试 phase 与计数器
- SDK-only 单消费者约束（用于排除竞争消费）：
  - 执行 `openclaw gateway stop`
  - 仅运行 `scripts/dingtalk-stream-monitor.mjs`
- 监控参数：
  - `--sdk-debug`
  - A 组：`--no-sdk-keepalive`
  - B 组：`--sdk-keepalive`

## 观测点说明

SDK 调试 phase：

- `inbound-received`：回调帧到达 SDK 入站解析点
- `callback-dispatch`：回调已分发到监听器
- `socket-send`：ACK 已发送到钉钉服务端
- `socket-open`：连接建立成功

监控统计字段：

- `callbackReceived`, `callbackAcked`
- `socketClose`, `socketError`
- `probeOk`, `probeFail`
- `sdkTopPhases`

## 实测结果

### A 组（关闭 SDK keepalive）

- App 发送：20 条（`trace-SDK-*`）
- SDK 实际入站：
  - `inbound-received(real)=7`
  - `callback-dispatch(real)=7`
  - `socket-send(real)=7`
  - `trace payload matched=7`
- 连接与传输健康：
  - `socketClose=0`
  - `socketError=0`
  - `probeFail=0`

结论（A 组）：

- 到达 SDK 的消息都完成了分发与 ACK。
- 缺失消息并非在本地回调处理内部丢失。

### B 组（开启 SDK keepalive）

- App 发送：20 条（`trace-SDK-kal-*`）
- SDK 实际入站：
  - `inbound=10`
  - `dispatch=10`
  - `ack=10`
  - `trace=10`
- 收到的 trace ID：
  - `trace-SDK-kal-001`
  - `trace-SDK-kal-006`
  - `trace-SDK-kal-007`
  - `trace-SDK-kal-008`
  - `trace-SDK-kal-010`
  - `trace-SDK-kal-011`
  - `trace-SDK-kal-013`
  - `trace-SDK-kal-014`
  - `trace-SDK-kal-016`
  - `trace-SDK-kal-017`
- 连接与传输健康：
  - `socketClose=0`
  - `socketError=0`
  - `probeFail=0`

结论（B 组）：

- keepalive 对接收率有改善（7 -> 10），但仍未达到 20/20。
- 本地 SDK 对已收到消息的处理与 ACK 仍然是完整可用的。

## 核心结论

高置信结论：

- 本次问题主要发生在 **本地 SDK `inbound-received` 之前**。
- 已收到回调在本地链路中未见丢失（分发与 ACK 全量成功）。
- 采样窗口内本地网络/连接无异常迹象（`socketClose=0`, `socketError=0`, `probeFail=0`）。

工作假设：

- 主要瓶颈位于 DingTalk 服务端 Stream 推送链路（上游投递/路由/应用侧推送选择）。

## 对钉钉侧升级排查建议（可直接贴工单）

```text
在单消费者（仅 monitor 进程）条件下进行了 SDK 级别打点排查：
- 每轮 App 发送 20 条消息。
- 关闭 SDK keepalive：SDK 入站回调仅 7/20。
- 开启 SDK keepalive：SDK 入站回调仅 10/20。
- 对于本地已收到的回调，分发与 ACK 均 100% 成功。
- 本地传输稳定：socketClose=0、socketError=0、probeFail=0。
结论：缺失消息未到达本地 SDK 回调入口，问题更可能位于服务端 Stream 推送路径，而非本地插件处理逻辑。
```

建议随工单附带：

- 原始监控日志（`/tmp/sdk-*.log`）
- 缺失 trace ID 列表
- 样例 `connectionId` / `messageId` 片段
