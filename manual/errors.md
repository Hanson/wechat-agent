# 错误码判型

> 手写手册。核心原则：**先看信封，再看协议语义，最后对时间线**。

## 响应信封的两套键（高频坑）

不同协议侧的错误信封键名不同：

| 形态 | 键 | 出处 |
|---|---|---|
| 企微/Fila 网关 | `err_code` / `err_msg` | 多数 |
| 个微部分接口 | `error_code` / `error_message` 或 `errcode` | 另一套 |

解析前先打原始响应。「解析不上、假象 CURL_FAIL」多数是键名取错——先看原始响应再查设备在线。

## 平台侧（GuidRequest 层）错误

| 错误 | 含义 | 处理 |
|---|---|---|
| app key 不存在或 secret 错误 | 鉴权失败 | 检查 app_key/app_secret |
| guid 不能为空 | data.guid 缺失 | 补 guid |
| guid 不存在 | 实例不存在或不属于该 app | 核对 guid 与 app 归属 |
| 坐席已过期 | 实例 expired_at 已过 | 全端点统一拒绝，需续期 |
| 应用对 guid 权限不足 | Full 模式 scopes 未含该 path | 走授权配置 |
| 路径不允许 | Saas/Sph 模式黑名单或白名单外 | 见 auth.md 权限模式 |
| 限流/超频 | 频控触发 | 按退避重试，勿硬打 |

## 协议侧高频错误码判型

### 通用

- **超时（timeout）**：透传默认超时窗口有限。同一 guid 上的慢请求会连锁拖垮后续请求（连接复用），先查该设备是否有大文件在传。
- **-1 / 空文案**：多为协议侧兜底失败。判型靠「只读探针」：用 get_profile 等只读接口逼出详细文案。

### 个微（wechat）

- **-2007**：验证会话过期（人脸/确认挂满 4 分钟不放行）。重新扫码。
- **-5103017**：CDN 取数层失败。两种成因：① file_size/aes_key 错配（解密前取数就失败）；② CDN 对象已删/全规格失效（两设备交叉均败 + 反向旧 fileid 成功 → 对象问题，重试无义，降级或重发消息）。
- **"guid not exists"**：信封是另一套键名（见上）。先查设备在线再查参数。
- **"plaintext size differs"（-1）**：file_size 与对象规格错配。
- **"ciphertext or padding invalid"**：aes_key 错（新版明确报错；旧版静默 0 字节——判成功必查 size/md5）。
- **"local file=missing"**：file_id 被调用方重构损坏（内嵌 `<seq>_<aes>_<md5>` 串变异）。调用方必须原样透传 msg 里的 file_id。

### Fila（fila）

- **error_code=4（SEND_FAILED）**：常因 MmContext 登录态丢。用只读探针逼详细文案。
- **code=10**：渠道未配对象存储（收图场景）。需平台侧配置后踢线重连生效。
- **command not supported**：guid 不在该网关（掉线/连到别的网关）或路径不存在（Fila 通道未知路径）。
- **11003**：账号资料同步（重连后自动触发），期间短暂不可用属预期。
- **-1003**：朋友圈账号级拒绝（非协议缺陷）。
- **-1007**：post_sns 带 store_type=1（本地）配 CDN fileid 必败；store_type 按 SDK demo 用 0。
- **error_code=-1 空文案**：两种成因，判型一锤定音法=网关日志 entering 行有无（平台侧排障）。

### 企微（wework_pc）

- **-102 / -1800016**：企微风控锁定。
- **-18000093**：建群设备会话级风控（限 guid 虚拟设备；手机端建群正常）。
- **cloud 相关接口已关闭**：企微 PC 实例禁用 /cloud 前缀（/cloud/add_image 除外），走私有化 CDN 接口。
- **"设备不存在"四判型**：打错协议机 / 抄错 guid / 外部 remove 摘除 / 实例自杀（stop 后 update 必败且 restore 不执行）。

## 三零判型（登录/桥接全零响应）

「ntchat 无响应 + admin 无日志 + bridge 无记录」三零 = 客户端/插件问题，勿查服务端。变种：

- 插件通道 6s 必死（manual login 带 send by bridge 标记）→ 清 bridge_id、退出插件。
- 装/更新插件后未刷新页面 → 孤儿注入断链，postMessage 5s 超时，刷新页面即愈。

## 排障顺序（速查）

1. 原始响应打出来，确认信封键名。
2. `get_profile` 探设备在线（只读探针兼逼错文案）。
3. 对时间线：同 guid 前后请求、协议机耗时窗口（发送类超时以网关 req 行数为准）。
4. 交给平台侧：guid + 精确时间点 + path。
