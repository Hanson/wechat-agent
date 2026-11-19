# 三协议登录状态机

> 手写手册。统一状态机（规划归一化层）先行给出语义基准；各协议原始形态见下文分节。

## 统一状态枚举（归一化层口径）

| 状态 | 语义 | agent 动作 |
|---|---|---|
| waiting | 码已出，等用户扫 | 展示二维码，继续轮询 |
| scanned | 已扫码，等手机确认 | 提示用户在手机上点确认 |
| confirming | 确认中/服务端处理 | 继续轮询 |
| success | 登录成功 | 停止轮询，开始业务调用 |
| expired | 码过期/会话失效 | 重新取码 |
| verify:code | 需要二次验证码 | 提交验证码 |
| verify:face | 需要人脸验证 | 引导用户走人脸流程 |
| verify:lan | 需要局域网验证（个微） | 引导手机侧完成局域网验证 |

原始 payload（各协议自己的状态字段/类型值）会随 verify:* 一并透出。

## 企微 PC（wework_pc）

- 取码：`get_login_qrcode`（参数 guid；verify_login 可选，二次验证场景）。响应含 base64 图与 `qrcode_content`。
- 轮询：`check_login_qrcode`（参数 guid）。
- 验证码：`verify_login_qrcode`（参数 guid + code）。
- 已知形态：
  - 码过期 → 重取新码，旧码作废。
  - 扫码后如触发企业风控，可能进入设备会话级风控（建群类操作受限，收发不受影响）。
  - 顶号拉锯（另一端反复挤下线）与 cursor 死循环属协议侧无解形态，只能重新扫码。

## 个微（wechat）

- 取码/轮询接口与企微同名：`get_login_qrcode` / `check_login_qrcode`（参数均仅 guid）。
- verify 分支类型（统一归一化前的原始类型值）：
  - 验证码（后端 SecurityCode 必成功；若前端报「获取验证信息失败」，属插件通道 5s 超时，重试/刷新即可）
  - 人脸验证（分层 302；扫码确认 ≠ 在线，人脸不放行时 wait confirm 可挂满 4 分钟后 -2007 过期）
  - 局域网验证（type 11）
- 判型要点：
  - 「扫码后一直 confirming 最后 -2007」= 人脸验证未放行。
  - 在线判定不能只看扫码确认，以 `get_profile` 三态（在线/离线/不存在）复核。

## Fila 手机版（fila）

形态完全不同：**auth_code 出码 + 设备在线状态查询**，没有 check 轮询语义。

- 出码：取 auth_code（编码串，本地展示为二维码）。
- 手机版 App（企微手机端 + Fila 注入）扫码 → 设备建 WS 长连到网关 → 在线状态可查。
- 三种「看起来没成功」的形态（高频误报）：
  1. 码未被使用：App 未扫/扫了没确认，状态一直未在线。
  2. WS 通但账号从未登录：连接建立 ≠ 登录，查账号资料为空。
  3. 断连后 App 被杀不回连：clean logout 后零 reconnect，需重新走登录。
- auth_code 归属判别：码密文本地可解（AES+XOR 出 key/state/ts），state 是会话 UUID 不是 guid；定位设备以网关日志为准（平台侧排障职责）。

## 登录成功后的信号

- 回调侧：登录成功/掉线事件经回调推送（per-guid 回调优先于 app 级全局回调）。事件 notify_type 口径（payload 结构见回调文档）：

| 事件 | 企微 PC / Fila | 个微 |
|---|---|---|
| 登录成功 | `11003`（UserLogin） | `1003`（UserLogin） |
| 掉线 | `11004`（UserLogout） | `1004`（UserLogout） |
| 扫码状态变化 | `11002`（LoginQRCodeChange） | `1002`（LoginQRCodeChange） |

- Fila 侧说明：上表 11003/11004 为平台归一后口径；手机原始事件为 type=1（登录）/ type=2（掉线），平台转发时已归一，agent 只需按上表判型。
- 轮询侧：`get_profile` 返回账号资料即在线。

## 掉线与重连

- 企微 PC：mmtls 断连可能无显式掉线回调（盲区），需对账 ticker 兜底——agent 侧以 `get_profile` 定期探活为宜。
- 个微：扫码确认 ≠ 在线；fd 泄漏/断连后不主动重登会保持「幽灵在线」形态，发送报错再重登。
- Fila：账号层重登指纹 = 回调收到 type=1（登录）但无 type=2（掉线）前置；网络 NAT 漂移会引发断连循环，重连 11003（账号资料同步）后自动恢复。
