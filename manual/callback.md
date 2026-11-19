# 回调对接（配置 / payload / 签名 / 重试）

> 手写手册。平台向你的服务推送消息与设备事件；本文覆盖配置方式、payload 结构、签名校验与重试语义。

## 1. 配置回调地址

两级配置，**per-guid 优先于 app 级**：某 guid 配了独立回调时，该 guid 的推送只发 per-guid 地址；未配置的 guid 走 app 级地址。

| 级别 | 配置方式 | 生效范围 |
|---|---|---|
| app 级 | POST `/open/SaveCallback`（app_key/app_secret 鉴权） | 该 app 名下全部实例 |
| per-guid | 透传协议接口：企微/个微 `/client/set_notify_url`；Fila `/fila/set_callback` | 单个 guid |

### SaveCallback（app 级）

```json
{"app_key": "ak", "app_secret": "sk", "callback_url": "https://your.host/callback"}
```

- 保存前做连通性验证（格式必须为 http/https，上限 1024 字符，不可达拒绝）。
- `callback_url` 传空串 = 清除（此时无 per-guid 配置的 guid 不再推送）。
- 保存成功后**立即生效**（推送侧内存缓存同步，无需等待）。

### 测试推送：TestCallback

```json
{"app_key": "ak", "app_secret": "sk"}
```

可选带 `"guid": "your_guid"` 测 per-guid 地址（该 guid 归属本 app 才可测，否则报「实例不存在」）。响应：

```json
{"success": true, "message": "推送成功", "response": "对端响应原文（截断 512）", "target_url": "https://your.host/callback"}
```

`target_url` 即实际测试的地址（per-guid 优先解析后的结果），可用来确认优先级是否符合预期。

## 2. 推送协议

平台向你配置的地址发起：

```
POST <callback_url>
Content-Type: application/json
AppKey: <app_key>
AppSecret: <app_secret>
X-Jh-Timestamp: <unix 秒>
X-Jh-Sign: <签名，见下节>
```

**响应判定：HTTP 2xx 即为成功**，其他状态码或连接失败按重试策略处理。对端无需返回特定 body。

## 3. 签名校验

```
sign = hex( HMAC-SHA256( app_secret, timestamp + "." + body ) )
```

- `timestamp` = `X-Jh-Timestamp` 的数字字面值（unix 秒）；`body` = 原始请求体字节（勿二次序列化）。
- 校验示例（Python）：

```python
import hmac, hashlib

expected = hmac.new(
    app_secret.encode(),
    f"{ts}.{body}".encode(),   # ts = X-Jh-Timestamp 字面值
    hashlib.sha256,
).hexdigest()
hmac.compare_digest(expected, x_jh_sign)  # True = 合法
```

- 建议同时校验时间窗（如 |now - timestamp| ≤ 300s）防重放。
- `AppKey` / `AppSecret` header 为存量兼容保留；新对接方以 `X-Jh-Sign` 校验为准。

## 4. 重试与熔断

单条推送失败（非 2xx 或网络错误）后按固定退避重试：

| 尝试 | 距上次失败 |
|---|---|
| 首次 | — |
| 重试 1 | 1s |
| 重试 2 | 5s |
| 重试 3 | 30s |

4 次尝试全部失败后放弃本条，并计入熔断计数：**同一 URL 连续失败 10 条 → 熔断 5 分钟**（期间静默跳过推送），冷却后半开放行试探，成功则恢复。

含义：回调地址长期不可用时消息**不保证送达**（重试窗口有限、熔断期丢弃），agent 侧应在收到 `11007`（SessionTimeout）等掉线事件或对账缺失时主动经 `/sync/sync_msg` 拉取补齐。

## 5. payload 结构

顶层三协议同构（Fila 事件已由平台归一）：

```json
{
  "guid": "your_guid",
  "notify_type": 11010,
  "data": { }
}
```

`data` 内容随 `notify_type` 变化，各事件字段以 `openapi/` 协议文档为准。常用事件：

| 事件 | notify_type（企微 PC / Fila） | notify_type（个微） |
|---|---|---|
| Ready | 11001 | 1001 |
| 扫码状态变化 | 11002 | 1002 |
| 登录成功 | 11003 | 1003 |
| 掉线 | 11004 | 1004 |
| 新消息 | 11010 | 1010 |

完整事件语义与登录状态机见 [login.md](login.md)。

TestCallback 的测试推送固定形态（可用于联调签名校验）：

```json
{"type": "test", "notify_type": 11001, "guid": "your_guid", "msg": "这是一条测试回调消息", "timestamp": 1732012800}
```

## 6. 注意事项

- 回调地址必须公网可达；私网地址（127.0.0.1 / 10.x / 172.16-31.x / 192.168.x）保存时会告警，仅适合自测。
- 强烈建议 HTTPS：`AppSecret` 与签名明文随每条推送传输，HTTP 明文下可被截获伪造。
- 签名密钥即 `app_secret`；怀疑泄漏时联系平台侧轮换（换 secret 后旧签名立即失效）。
- 处理要幂等：极端情况下重试可能造成对端已收 2xx 但平台未及时读到响应的重发。
