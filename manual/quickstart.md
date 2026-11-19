# 最小闭环示例（建实例 → 取码 → 配回调 → 发消息 → 对账）

> 手写手册。开放平台自有端点（/open/*）以上线实现为准；协议透传参数见 `openapi/`。

## 0. 前置

- 已获 `app_key` / `app_secret`（收款开通后发放，唯一人工步骤）。
- 本地起一个 HTTP 服务接收回调（webhook），或先用轮询模式跑通。

## 1. 创建实例

POST /open/CreateInstance —— 鉴权（app_key/app_secret）+ 协议通道（`channel`：1=企微 PC / 2=个微 / 3=Fila）+ 过期时间（`expired_at`，unix 秒，必填且须晚于当前时间）+ 可选备注。

```json
{"app_key": "ak", "app_secret": "sk", "channel": 1, "expired_at": 1800000000, "description": "agent-01"}
```

返回 `guid` + `proto_url` + `instance_id`。**Fila 通道在手机扫码绑定前 `guid` 为空**，后续登录/删除接口用 `instance_id` 定位。

配套：POST /open/SetProxy（socks5 代理，空串清除；仅企微 PC 与个微）、POST /open/DeleteInstance（删除后立即失效，回调配置同步清理）。

> 实例定位规则（登录/代理/删除接口通用）：`guid` 与 `instance_id` 二选一，guid 优先；非本 app 的实例一律报「实例不存在」。

## 2. 取登录二维码

POST /open/GetLoginQrcode —— 企微 PC / 个微返回 `qrcode_base64`（企微另有 `qrcode_content` 可自渲染链接）；Fila 返回 `auth_code`（手机端 Fila App 扫码接入）。

```json
{"app_key": "ak", "app_secret": "sk", "guid": "your_guid"}
```

Fila（guid 未生成，用创建时返回的 instance_id）：

```json
{"app_key": "ak", "app_secret": "sk", "instance_id": 123}
```

码有时效，过期重取即可（新码替换旧码）。协议侧原始接口（/login/get_login_qrcode）也可经 GuidRequest 透传调用，但自有端点已做归一化，推荐前者。

## 3. 轮询登录状态

POST /open/CheckLoginQrcode（定位参数同上）—— 返回归一状态 `status`（waiting/scanned/confirming/success/expired/verify:code/verify:face/verify:lan）+ `raw_status`/`raw_payload`（verify 分支处置所需原始载荷）+ `nickname`/`avatar`/`account`（企微 vid / 个微 wxid）；Fila 另有 `online`（设备 WS 连接状态，连接 ≠ 账号已登录）。

状态语义与各协议差异见 login.md 状态机。若返回 verify:* 分支，按 login.md 处理。企微 PC 的验证码提交仍走透传：

POST /login/verify_login_qrcode

```json
{"guid": "your_guid", "code": "123456"}
```

## 4. 配置回调（收消息）

app 级回调：POST /open/SaveCallback —— 保存前做连通性验证，空串清除。

```json
{"app_key": "ak", "app_secret": "sk", "callback_url": "https://your.host/callback"}
```

配好后用 POST /open/TestCallback 发一条带签名的测试推送验证连通（响应含 `target_url` / 对端响应原文）。

per-guid 回调（优先级高于 app 级）：直接透传协议接口。企微/个微：

POST /client/set_notify_url

```json
{"guid": "your_guid", "notify_url": "https://your.host/callback"}
```

Fila 手机版：透传 `/fila/set_callback`、查询 `/fila/get_callback`、删除 `/fila/remove_callback`（参数与响应见 openapi/fila/ 产出）。

回调 payload 结构、签名校验（X-Jh-Sign）与重试熔断策略见 [callback.md](callback.md)。

## 5. 发消息

企微 PC（会话 id 由登录后同步获得）：

POST /msg/send_text

```json
{"guid": "your_guid", "conversation_id": "target_conversation", "content": "hello from agent"}
```

个微：

POST /msg/send_text

```json
{"guid": "your_guid", "to_username": "wxid_target", "content": "hello from agent"}
```

Fila：`data` 结构见 openapi/fila（发送族接口 text oneof 约束见 pitfalls.md）。

## 6. 消息对账

轮询拉取历史消息确认闭环：

POST /sync/sync_msg

```json
{"guid": "your_guid", "limit": 100, "sync_key": ""}
```

（个微此接口仅 guid；首次 sync_key 传空，之后回传响应里的新 sync_key。）

## 7. 排障最小集

1. `user/get_profile` 探测设备在线与账号资料（字段仅 guid）：

POST /user/get_profile

```json
{"guid": "your_guid"}
```

2. 全链路不通时按 errors.md 判型：先看信封错误码，再按「三零判型」区分插件通道/设备问题。
3. 仍无法定位：提供 guid + 时间点 + path 给平台侧查协议机日志。
