# 坑手册（首版）

> 手写手册。来源：线上工单与排障档案提炼。参数级细节以 `openapi/` 产出为准。

## 图片/文件下载 file_type 矩阵（个微 CDN）

| file_type | 含义 | 校验 |
|---|---|---|
| 1 | 按报文规格下载（保底） | 产物精确 == msg.size（is_hd true/false 通吃） |
| 2 | 更大版本规格 | 不报 incomplete，校验仅单向（可大于 file_size） |
| 3 | 缩略图 | — |
| 7 | 新版电脑版大文件通道（fileid 以 `7f0c` 开头必传 7，传 5 必 -5103017） | file_size=0 可行 |

- 1 与 2 是 CDN 独立规格对象，互为保底：**一档失败必试另一档**，单档失败 ≠ 凭证过期（7 天旧凭证仍可下）。
- 无 hdlength 的消息：CDN 无原图规格，type1 必 -5103017，只剩模糊缩略图，属固有形态非下载问题。
- 产物成功判据：`errcode:0` + 响应 size == file_size。**t2 规格「响应 md5 ≠ 落盘 md5」是转码脚本覆盖写回所致，以响应 size 为准勿误判**。

## 产物格式三形态（判别 = 文件头 hex）

| 形态 | 文件头 | 可否直接展示 |
|---|---|---|
| t1 发送方原始文件 | 明文头（如 89504E47 PNG） | 可 |
| t2 wxgf 私有容器 | `77 78 67 66`（"wxgf"，HEVC） | 浏览器打不开（客户「资源不完整」的真因），需转码 |
| t3 标准 JPEG | FFD8 | 可 |

## 凭证形态分流（企微互通文件）

- `file_id` 以 30 开头 → 走 c2c download 通道，传 file_id。
- url 型（7f0c hex 开头）+ auth_key（v1_ 前缀）→ 走企微互通下载通道（url + auth_key）。
- 图片：enc_idx/key/token 三件套；视频：decode_key。
- 互通群企微侧图/视频/文件裂图：XML 只有 tp* 凭证（无 aes_key），走企微互通下载通道，token 有效期 1.5-4h。

## 上传与转换

- **to_silk 禁用**：c2c_upload 传 `to_silk:true` 100% 打崩企微手机端（WS 1006 EOF = 进程崩溃指纹，自动重试 = crash loop）。音频直传原格式。
- 定时任务复用签名 URL：URL 有效期 1h，过期后整轮 403（按 10min 周期爆发的 403 指纹）。

## 发送族约束（Fila）

- 发送文本走 text oneof（scene 自动包装不覆盖全部场景，@/引用消息不动包装层）。
- 媒体 oneof 不构造（乱构造会打崩 App）；appinfo 必填。
- send_image 必嵌套 image 对象（平铺报 unknown field aes_key；宽高真名 width/height）。
- bytes 类字段（sph 等）：网关不自动编码（extras 除外），传 base64。

## 键名与大小写双坑

- 直调网关必须 snake_case 字段；响应键是 camelCase（两坑对称）。
- 响应嵌套键静默 no-op：字段名对不上不报错、行为不生效（roomInfos 型），逐字段 diff 文档值形态。
- Fila 错误信封双层：`error_code`（外层网关）+ `error_message`（内层协议）。

## 二维码三形态

| 协议 | 形态 | 渲染 |
|---|---|---|
| 企微 PC | base64 图 + qrcode_content 链接 | 直接展示或自渲染 |
| 个微 | base64 图 | 直接展示 |
| Fila | auth_code 编码串 | 本地编码成二维码图 |

- 码过期重取即可；Fila 的 auth_code 不能重复使用判断（以设备在线状态为准）。

## 回调与推送

- per-guid 回调优先于 app 级。
- 回调里通知 ID ≠ 业务消息 ID（企微工单消息 ID 是 notify id 非 msgs 自增 id）。
- 断连窗口内的推送会丢（设备 WS 稳定不重发）：admin 重启窗口丢失的 login event 靠启动对账补偿（平台侧行为），agent 侧用 get_profile 兜底。

## 消息类型速记（接收侧）

- 企微视频号：收 = msg_type 4 + ct 13；发必须走 send_link（不能发原始视频）。
- 企微图片回调缺失三形态：服务端 sync 不下发 = 源头丢（非丢失，无解）；文件名是起始时间戳非业务字段。
- 个微表情（msg_type 47）：cdnurl 走两步转存，或 encrypturl+aes_key 一步直达；错 aes_key 可能 errcode=0 但产物 0 字节（判成功必查 size）。
- word_chain 群接龙 = type 17；引用消息 message 恒 text oneof。

## 杂项高频坑

- 长轮询/大 base64 请求：用可承载大 body 的客户端（shell curl 截断坑），HTTP 客户端调大缓冲。
- 同 guid 慢请求连锁超时：大文件下载期间避免同设备并发发消息。
- file_id 原样透传：任何「重构/清洗」都会造成 local file=missing 家族错误。
