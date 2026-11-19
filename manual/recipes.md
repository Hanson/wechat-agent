# 接口实现配方（参数怎么填）

> 手写手册。回答一个问题：**每个参数的值从哪来、怎么转换、常量是什么、接口之间怎么衔接**。全部配方提炼自平台侧实现代码遍历（配方与线上服务逐条核对）。参数规格以 `openapi/` 产出为准；踩坑判型见 `pitfalls.md`。

## 0. 通用约定（先读）

- **请求形态**：所有协议接口都是 `POST {protoUrl}{path}` + JSON body，`guid` 恒为第一个字段（企微同构）。
- **响应信封三形态**（同一业务接口按协议不同）：

| 协议 | 信封 | 取值示例 |
|---|---|---|
| 企微 PC | `data.*` 包裹 | `data.file_id`、`data.msg_data.id` |
| 个微 | 顶层平铺 | `newMsgId`、`file_id`、`baseResponse.ret` |
| Fila 手机 | `error_code` + `data.*`，camelCase | `data.fileId`、`error_code:0`；bytes 字段输出 base64，uint64 输出字符串 |

- **conversation_id 形态**（发送与回调的会话定位）：

| 协议 | 群 | 单聊 | 转换规则 |
|---|---|---|---|
| 企微 PC | `R:`+roomId | `S:`+userid | 发送原样透传 |
| 个微 | `wxid_` 前缀拼接 | 同左 | 发送前 `TrimPrefix(conversationId, robot.wxid+"_")` 剥前缀 |
| Fila | `R:`+roomId（可带 `;asid` 后缀） | `S:`+userid | `filaConvType` 拆 conv：`R:`→conv_type 1（群）/`S:`→0，先去 `;asid` 再取 id |

- **CDN file_type 三套枚举，切勿混用**：

| 语义 | 企微 cos | Fila CDN | 个微下载 |
|---|---|---|---|
| 原图 | 1 | 1 | 1（Origin） |
| 图片/中图 | 2 | 2 | 2（Mid） |
| 缩略图 | 3 | 3 | 3（Thumb） |
| 视频 | 4 | **5** | 4 |
| 文件/语音 | 5 | **4** | 5 |

  - 企微 cos 枚举；Fila 网关枚举（**4/5 与企微正好相反**）；个微下载枚举（代码内 iota 0-5，无 7）。
  - 直调协议机口径另有 **file_type=7**（新版电脑版大文件通道，fileid 以 `7f0c` 开头必传 7）——这是生产判例结论，平台封装代码内不出现，见 `pitfalls.md`。
- **发送响应统一取 msg id 的位置**：企微/Fila 取 `data.msg_data.id`（Fila 为 `data.msgId`）；个微取顶层 `newMsgId`。

---

## 1. 发送族：怎么把一条消息发出去

### 1.1 文本

POST /msg/send_text

```json
{"guid": "8f3a...", "conversation_id": "S:1688851234", "content": "你好"}
```

| 协议 | 字段差异 | 实现 |
|---|---|---|
| 企微 PC | `conversation_id` 原样 `S:`/`R:` 前缀 | 平台实现 |
| 个微 | `to_username`（剥 `wxid_` 前缀后的对方 wxid） | 平台实现 |
| Fila | 不走 send_text，走 `/msg/send_message_scene`：body `{guid, conversation_id, text:{content}}`——text oneof 子对象由调用方手工构造（scene 不在网关自动包装清单，仅 send_text/send_room_at 被包装） | |

### 1.2 群 @（at_list 常量转换矩阵）

POST /msg/send_room_at

```json
{"guid": "8f3a...", "conversation_id": "R:2000123456", "content": "通知", "at_list": ["all"]}
```

| 协议 | `"all"` 转换为 | content 要求 | 实现 |
|---|---|---|---|
| 企微 PC | `"0"`，且 content 前强制加空格 | 协议端自动拼 @昵称 | 平台实现 |
| 个微 | `"notify@all"` | 同上 | 平台实现 |
| Fila | `["0"]` | **必须自带 `@昵称` 段**（App 不拼）：先查本地群成员数据拼 `@名1@名2`，`["0"]`→`@所有人`；查不到实时拉全量成员落库自愈（SetNX 30s 节流） | 平台实现 |

### 1.3 图片：上传→发送完整链路（最常用配方）

**第一步，上传换 CDN 凭证**（详见 §2.1）：

POST /cdn/c2c_upload

```json
{"guid": "8f3a...", "url": "https://bucket.cos.ap-guangzhou.myqcloud.com/a.png", "file_type": 2}
```

**第二步，上传响应字段 → 发送参数映射**（这是衔接关键）：

| 上传响应字段 | 企微 send_image 参数 | 个微 send_image 参数 | Fila send_image（image oneof 内） |
|---|---|---|---|
| `file_id` | `file_id` | `file_id` | `image.file_id` |
| `aes_key` | `aes_key` | `aes_key` | `image.aes_key` |
| `file_md5` | `md5` | `file_md5` | `image.md5` |
| `file_size` | `size` | `file_size`；另 `thumb_file_size` 同值再传一处 | `image.size` |
| `image_width` | `image_width` | `thumb_width` | `image.width` |
| `image_height` | `image_height` | `thumb_height` | `image.height` |
| — | `is_hd` 恒 `false` | — | — |

POST /msg/send_image

（企微形态示例）

```json
{"guid": "8f3a...", "conversation_id": "S:1688851234", "file_id": "3052520...", "aes_key": "v1_xxx", "md5": "abc123", "size": 33494, "image_width": 1080, "image_height": 1920, "is_hd": false}
```

- 个微形态：平铺 + `to_username`；**平铺才是正确形态，不要嵌套 image 对象**。
- Fila 形态：媒体走 **oneof 嵌套** `image:{file_id,size,width,height,aes_key,md5}`，宽高真名 `width/height`；**凭证缺一即被平台拦截**——缺凭证的 body 会打崩 Fila App。
- 上传必须走长超时通道（企微 `RequestUploadWithMapCtx`，普通 15s client 大文件必超时）。

### 1.4 语音（15 天 silk 规则）

- 凭证链（三协议同构）：voice 素材需要 silk 凭证；**创建时间超 15 天且已有下载地址** → 把 `.mp3` 后缀替换成 `.silk` 取新 URL 重新上传换新凭证再发（企微、Fila）。
- 上传 file_type：企微=5，Fila=**4**（枚举相反，见 §0）。
- Fila `voice_time` 用秒；上传响应 `duration` 是毫秒须 `/1000`。

POST /msg/send_voice

```json
{"guid": "8f3a...", "conversation_id": "S:1688851234", "file_id": "3052520...", "aes_key": "v1_xxx", "md5": "abc123", "size": 9216, "voice_time": 3}
```

- 个微同字段名；Fila oneof `voice:{file_id,size,voice_time,aes_key,md5}`。
- **to_silk 永不构造**：音频转码在平台侧本地完成，协议上传无此参数；直调时带 `to_silk:true` 会 100% 打崩企微手机端（`pitfalls.md`）。

### 1.5 视频

- 企微：上传 `file_type=4`，响应 `data.video_duration/video_width/video_height` 直填发送参数；`file_name` 常量 `"视频.mp4"`。
- 个微：thumb 凭证来自 `CdnUpload(url, file_type=4)` 响应；`mp4_identify` 禁传。
- Fila：oneof `video:{file_id,size,file_name,aes_key,md5,duration,width,height}`——width/height 实际填上传响应 `image_width/image_height`。
- **>30MB 大视频**（仅企微）：改走 `/cloud/cdn_big_upload`（`{guid,url}` 无 file_type）+ `/cloud/upload_video_preview`（`{guid,url}` → `data.image_url/preview_image_md5`）两步，再发 `/msg/send_big_video`；`video_duration` 取 COS 元数据而非上传响应。已知实现缺陷：`preview_image_width/height/size` 三字段读取位置错误恒 0，对接时宽高可自行探测补全。

### 1.6 文件

- 企微：上传 `file_type=5`；个微：`to_username` 剥前缀，`file_crc/file_key` 禁传；Fila：oneof `file:{file_id,size,file_name,aes_key,md5}`，file_name 空则 `<unix>_file`。
- 实现：三协议均为平台封装。

### 1.7 链接 / 名片 / 小程序 / GIF / 位置

| 消息 | 协议差异要点 | 实现 |
|---|---|---|
| 链接卡片 | 企微 `/msg/send_link` 字段 `description`；个微 URL 版 `/msg/send_link_card` 字段名是 **`desc`**；个微有图版 `/msg/send_link_card_cdn` 需 thumb 四件套（thumb 先 `CdnUpload(file_type=3)`） | 平台实现 |
| 名片 | 企微 `/msg/send_personal_card` 字段 `user_id`；个微 `/msg/send_share_card` 字段 **`share_username`** | 平台实现 |
| 小程序 | 企微 `/msg/send_weapp` 字段 `md5/size`；个微 `/msg/send_mini_app` 字段 **`file_md5/file_size`**；其余同名（`aes_key,appid,page_path,title,username,appicon,file_id,appname`），值来自收到小程序消息时落库的素材（回调 `cdn.aes_key`/`cdn.file_id`） | 平台实现 |
| GIF | 企微 `/msg/send_gif_url` `{url,md5,image_width,image_height}`；Fila oneof `emotion:{url,md5,width,height}`（字段名变短） | 平台实现 |
| 位置 | 仅 Fila `/msg/send_location`，oneof `location:{longitude,latitude,address,title}` | 平台实现 |
| 视频号 | 个微 `/msg/send_finder_video` 带 `object_id/object_nonce_id`（视频号详情接口取）；企微 `/msg/send_feed_video` | 平台实现 |

个微链接 CDN 版示例（thumb 凭证来自缩略图上传响应）：

POST /msg/send_link_card_cdn

```json
{"guid": "8f3a...", "to_username": "wxid_abc123", "title": "标题", "desc": "描述", "url": "https://example.com", "thumb_file_id": "3052520...", "thumb_file_md5": "abc123", "thumb_aes_key": "v1_xxx", "thumb_file_size": 1024}
```

### 1.8 引用回复：参数从回调里怎么拿

被引用消息的原料全部来自**收到该消息时的回调与落库**，不是现查接口：

| 发送参数 | 从哪拿 | 说明 |
|---|---|---|
| `appinfo` | 回调字段 `appinfo` 原样保存 | **Fila 必填**——手机端靠它定位原消息，缺失直接报错 |
| `sender` / `sender_name` | 回调的发送者字段 | Fila sender 须 ParseUint 成数字，非数字报错 |
| `content_type` | 回调 `content_type` | Fila 恒填 **2** |
| `quote` | 被引用消息摘要前 100 字符 | `SafeTruncate(GetTypeContent(),100)` |
| `message` | 被引用消息对象化 | 企微：text→Content；image/video/file→`Cdn.FileId/AesKey/Md5/Size/AuthKey`（image 宽高网络探测 DownloadUrl，**video 宽高写死 720/1280**）；link→Title/Desc/Url/ImageUrl。Fila：**恒 text oneof** `{msg_type:"text",sendtime,conv_type,conv_id,text:{content:摘要}}`——非文本也不构造媒体 oneof（凭证不全会崩 App） |

POST /msg/send_quote_msg

（企微/Fila 平铺契约同形，下为 Fila 示例）

```json
{"guid": "8f3a...", "conversation_id": "R:2000123456", "quote": "原始消息摘要", "content": "回复内容", "appinfo": "<原消息 appinfo 原文>", "sender": "1688851234", "sender_name": "张三", "content_type": "2", "message": {"msg_type": "text", "text": {"content": "原始消息摘要"}}}
```

- 企微与 Fila 均有平台封装（conv_type/conv_id 由 §0 的 filaConvType 拆出）。
- 接收侧衔接：Fila 回调把 `extra_content/quote_content/quote_appinfo` 原料透传；企微用 `extra_content` 解码重建引用。
- 个微无 send_quote_msg，走 `/msg/send_refer_msg`：

POST /msg/send_refer_msg

```json
{"guid": "8f3a...", "to_username": "wxid_abc123", "content": "回复内容", "refer_msg": {"msg_id": "6300001234567890", "from_username": "wxid_def456", "from_nickname": "李四", "source": "聊天记录", "content": "原始消息内容", "msg_type": 1}}
```

  refer 内六项全部取自被引用消息本体。

### 1.9 撤回：三协议三种定位方式

POST /msg/revoke_msg

（企微形态）

```json
{"guid": "8f3a...", "conversation_id": "S:1688851234", "msgid": "6300001234567890"}
```

| 协议 | 定位方式 | 参数 | 实现 |
|---|---|---|---|
| 企微 PC | 消息 id | `{guid, conversation_id, msgid}`——msgid=发送响应/回调的 OriginId | 平台实现 |
| 个微 | 消息 id | `{guid, to_username, msg_id, client_msg_id:0}`（client_msg_id 写死 0） | 平台实现 |
| Fila | **按 appinfo 撤回** | 先按 `uin+origin_id` 查库取该消息 `Extra.Appinfo`（缺则报错），发 `{guid, appinfo, conv_id, conv_type}`；appinfo 之所以可查，是发送响应的 appinfo 被反转 base64 存库 | 平台实现 |

### 1.10 已读 / 转发 / 群发（速记）

- 已读：企微 `/msg/report_unread` body 只有 `{guid, conversation_id}`（unreadCount 只用于本地推送不传协议）；个微 `/msg/set_session_read` `{guid, to_username}`。
- 合并转发：仅企微 `/msg/send_forward_msg` `{guid, conversation_id, message_list}`；图片项 file_id 是 http 开头时须先 `c2c_upload(file_type=2)` 换 CDN 凭证回填。
- 群发：仅企微 `/msg/send_group_msg`——四个 `*_msg_list` 为空必须显式 `[]`，nil 会报错。

---

## 2. 上传族：媒体先上 CDN 才能发

### 2.1 主通道对照

| 协议 | 接口 | 请求字段 | 响应取值 | 实现 |
|---|---|---|---|---|
| 企微 PC | `/cdn/c2c_upload` | `{guid, url, file_type}`（企微枚举） | `data.file_id/aes_key/file_md5/file_size/image_width/image_height/video_duration` | 平台实现 |
| 个微 | `/cloud/cdn_upload` | `{guid, url, file_type}`（企微枚举口径） | **顶层平铺** `file_id/aes_key/file_md5/file_size[/image_width/image_height/thumb_*]` | 平台实现 |
| Fila | `/cdn/c2c_upload`（网关同名） | **`file_url`**（不是 url）+ `file_type`（Fila 枚举） | `data` camelCase `fileId/aesKey/fileSize/md5/width/height/duration(毫秒)` | 平台实现 |

- `url`/`file_url` = 文件可下载地址（对象存储/公网直链均可），协议机侧自行拉取。
- **三协议都必须走长超时通道**（企微 RequestUploadWithMapCtx / Fila 125s 大文件 client），普通 15s client 大文件必超时。
- Fila 防"假成功"：上传产物 `< 1024` 字节判定失败拦截（设备侧下载素材失败拿到错误页当媒体传回的形态）。

### 2.2 大文件 / 朋友圈通道 `/cloud/cdn_big_upload`

| 协议 | 请求 | 触发 | 响应 | 实现 |
|---|---|---|---|---|
| 企微 | `{guid, url}`（无 file_type） | 视频 >30MB | `data.file_id/file_size` | 平台实现 |
| Fila | `{guid, file_url, scene}`，scene 枚举 3/7/8 | 大文件；**朋友圈实测 scene=8** | `fileId/fileSize`（`*1*` 前缀 FTN 凭证），**无 md5/宽高/时长**，需从对象存储元数据补 | 平台实现 |

- 朋友圈 media_list 的 fileid **必须用本通道产物**：c2c_upload 的 hex fileid 手机端必裂。
- 接口名注意：wire 路径就叫 `cdn_big_upload`（不是 cdn_upload_big）。

### 2.3 辅助上传

| 接口 | 协议 | 字段 | 说明 |
|---|---|---|---|
| `/cloud/cdn_upload_sns_image` / `cdn_upload_sns_video` | 个微 | `{guid, url}` | 朋友圈专用，无 file_type |
| `/cloud/add_image` | 企微 | `{guid, url}` | — |
| `/cloud/upload_video_preview` | 企微 | `{guid, url}` | 大视频预览图 |
| `/cloud/update_cdn_rule` | 企微 | `{guid}` | CDN 规则刷新（6h 定时全量刷，Fila 实例排除——网关无此命令） |
| `/cloud/update_cdn_dns` | 个微 | `{guid}` | — |

---

## 3. 下载族：收到消息取媒体

### 3.1 企微 PC 分流（编排层决策）

`CloudDownload`按凭证形态分流：

- file_id 以 `"30"` 开头 → `/cdn/c2c_download`（传 file_id + file_type + 凭证）。
- URL 型 file_id（`https://imunion...` 直链）→ `/cloud/cdn_wx_download`（`{guid, url, auth_key, aes_key, file_name}`，**该接口不收 file_type**）。
- file_type 常量：image=2、video=4、file/voice(.silk)=5；**图片 2 失败自动降级重试 1**（原图档）。

### 3.2 个微

| 接口 | 字段 | 要点 |
|---|---|---|
| `/cloud/cdn_download` | `{guid, file_id, file_type, aes_key, file_name, file_size}` | file_type 用个微枚举（图=2 缩略=3 视频=4 文件=5）；file_size 必传（==msg.size）；`file_id=="" && ww_url!=""`（互通群企微凭证）→ 改走 wwfile 通道 |
| `/cloud/cdn_download_wwfile` | `{guid, url, auth_key, aes_key, file_name}`（无 file_size） | 凭证来自消息 XML tp* 字段；文档名 download_ww_file 未实现，可用名就是本名 |
| `/cloud/download_voice` | `{guid, msg_id, length, room_username, file_name:"voice.mp3", to_mp3:true}` | msg_id=消息 OriginMsgId；length=voicemsg XML `length`（字节）；响应顶层 `url`（已转 mp3） |
| `/cloud/download_wximage` | `{guid, url, token, enc_idx, key, file_name}` | 朋友圈/加密图四元组原样透传 |

- 下载产物 md5 有 30 天缓存与 pending 互斥；>200MB 跳过。
- 产物判成功：`errcode:0` + 响应 size == file_size；t2 规格（wxgf 容器）落盘 md5 与响应 md5 不一致是转码覆盖所致，**以响应 size 为准**（`pitfalls.md`）。

### 3.3 Fila（file_id 三前缀分流 + 重试矩阵）

file_id 分流：`http` 前缀→`/cloud/cdn_wx_download`（Fila 版多 `md5/file_size`）；`*1*` 前缀（FTN）→`/cdn/big_download`（仅 `{guid, file_id, file_name}` 三字段）；其余→`/cdn/c2c_download`。

`c2c_download` 专用规则：

| 规则 | 内容 | 实现 |
|---|---|---|
| file_type 换算 | 入参企微语义，4↔5 互换后下发 | 平台实现 |
| 图片换档保底 | 1↔2 互换重试（档位与 size/md5 必须配套，不匹配报 incomplete） | 平台实现 |
| error_code=4 | CDN 文件不完整：间隔 3s 重试至多 2 次 | 平台实现 |
| error_code=10 | 渠道未配存储：推平台 COS 兜底 + 踢线重连 + 重试一次 | 平台实现 |
| URL 型凭证走 c2c | 必败 error_code=4——URL+auth_key(`v1_` 前缀) 必须走 wx_download | 平台实现 |

---

## 4. 语音转文字（三协议三条链路）

| 步骤 | 企微 PC | 个微 | Fila |
|---|---|---|---|
| 1 申请 | `/msg/apply_voice_id` `{guid, conversation_id, msgid}` → `data.voiceid` | `/msg/new_trans_voice_id` `{guid}` → 顶层 `voice_id` | `/msg/apply_voice_to_text` `{guid, msgid}`（msgid 必须 ParseUint 数字）→ `data.voiceid` |
| 2 上传 | — | `/msg/upload_voice_trans` `{guid, voice_id, msg_id, length}`（length=Cdn.Size 字节） | — |
| 3 查询 | 固定 sleep 1s 后 `/msg/query_voice_text` `{guid, conversation_id, msgid, voiceid, seqid}`（seqid=回调 `data.seq`） | `/msg/get_voice_trans` `{guid, voice_id}`，轮询 1s×≤10 次 | `/msg/query_voice_to_text` `{guid, voiceid, seqid, msgid}` |
| 4 终止 | `data.text` 直接完整 | `transRes.result` | **流式**：`data.text` 分段拼接、`data.seqid` 游标回传（首次 0）、`data.isend=true` 结束；轮询 1s×≤15 次，耗尽返回已转写部分 |

- msgid 来源：企微=回调 `data.id`；个微=消息 OriginMsgId；Fila=DB origin_id 数字串。
- Fila 链路须用 35s client（普通 15s 提前断连）。实现为平台封装。
- 个微另有 `/msg/check_voice_trans`（已定义，主链路未用）。

---

## 5. 群聊域

### 5.1 建群与成员增删

| 操作 | 企微 PC | 个微 | 要点 |
|---|---|---|---|
| 建群 | `/room/create_outer_room` `{guid, user_list}`（内部成员填 Uin、外部填 UserId） | `/room/create_chatroom` `{guid, username_list}`（wxid 数组） | 响应键不同：企微 `data.roomid` / 个微 `chatRoomName.string`；建群后每步 sleep 1s 再改名/公告/管理员 |
| 建空群 | `/room/create_empty_outer_room` `{guid}` | — | 保留能力 |
| 拉人 | `/room/invite_room_member` `{guid, room_id, user_list}` | **按群人数分流**：≤40 `/room/add_chatroom_member`；>40 `/room/invite_chatroom_member`（发邀请链接）；两接口字段完全同名 `{guid, room_username, username_list}` | 阈值判定在 handler 层 |
| 踢人 | `/room/remove_room_member` `{guid, room_id, user_list}` | 无对应接口 | — |
| 群内加好友 | `/room/add_room_contact` `{guid, room_id, user_id, corp_id, verify}` | `/room/add_chatroom_member_friend` `{guid, room_username, username, verify_content}`（无 corp_id） | — |
| 接受邀请 | `/room/accept_invite_url` `{guid, invite_url}` | — | invite_url=回调事件 `data.url` |

### 5.2 群列表三形态分页（大坑区）

| 协议 | 接口 | 请求 | 翻页游标 | 终止条件 |
|---|---|---|---|---|
| 企微 PC | `/room/get_room_list` | `{guid, limit:100, start_index}` | 响应 `data.next_start` 回传 | `next_start == -1` |
| Fila | `/session/fetch_room_list` | 仅 `{guid}`（无分页，一次全量） | — | — |
| 个微 | `/room/get_chatrooms`（Deprecated） | `{guid, page_num, page_size:100}`（页码制） | `total_page` | `total_page==0` |

- **企微解析必须兼容 Fila 形态**：Fila 网关同路径分页键是 `data.result.nextStart`（嵌套+camelCase），旧解析恒读 `data.next_start` 得 0≠-1 → 死循环轰炸网关。

### 5.3 群详情 / 成员

POST /room/batch_get_member_detail

（Fila 形态——camelCase 且不带 user_list = 全量拉取）

```json
{"guid": "8f3a...", "roomId": "2000123456"}
```

- 企微形态 `{guid, room_id, user_list}`，响应 `data.persons[].info`。
- **Fila 必须不带 user_list**：protojson 严格模式遇未知字段报错；响应 `data.members` 有数组/JSON 字符串内嵌两种形态，成员 id 取 `member.remoteId`。
- `batch_get_room_detail` 响应键双读：企微 `data.roominfos`（全小写）/ Fila `data.roomInfos`（camelCase），解析先 camel 再 fallback 小写；Fila bytes 字段（群名/昵称）是 base64 须解码，企微明文不可盲解。
- Fila 成员接口读 App 本地库有竞态（11013 可能早于本地库更新 14s+），主源用 `/room/sync_room_info`：Fila 形态 `{"guid","roomId","version":0,"memberCount":0}` 字面量（camelCase，0 触发全量）；企微形态 `{guid, room_id, version}`（响应 `data.roominfos` 数组，version 相等即无更新）。
- 群二维码 `/room/get_room_qrcode`：`get_type=0`→`data.image_url`；`get_type=1`→`data.room_qrcode`（base64 图，需本地解码出链接）；**内部群仅支持 0**。

### 5.4 管理设置

| 操作 | 企微（键名） | 个微（键名） |
|---|---|---|
| 改名 | `/room/modify_room_name`（`room_name`） | `/room/modify_chatroom_name`（`name`） |
| 公告 | `/room/modify_room_notice`（`notice`） | `/room/set_chatroom_announcement`（`announcement`） |
| 备注 | `/room/modify_room_remark`（`remark`） | — |
| 管理员 | `/room/room_add_admin` `{guid, room_id, user_list}` | — |

### 5.5 置顶（不能走 map 序列化）

`/room/set_room_top_msg` 必须手工拼 JSON 后用 SetRaw 塞 **message 原始 JSON 对象**（不是转义字符串）：`{"guid","room_id","appinfo"}` + message 整体；appinfo=原消息 `gjson(RawMsg,"appinfo")`，message=整个 RawMsg（先删所有反斜杠）。取消置顶 `/room/remove_room_top_msg` 用置顶响应/DB 的 `top_id`。

---

## 6. 联系人与标签

### 6.1 加好友两条路（先搜索拿凭证，再发申请）

**企微 PC 加个微好友**（企微没有 add_friend，走专用接口）：
1. `/contact/search_contact` `{guid, keyword, type}`——type 常量 **1=手机号 2=邮箱 3=wxid**；成功判 `data.search_status=="SEARCH_OK"`，结果 `data.datalist[]`。
2. `/contact/add_search_wx_contact`：`user_id`=datalist 的 `contact_info_wx.uin`、`wx_ticket`=datalist 的 `ticket`、`openid`=datalist 的 `openid`、`verify`=验证语。

**个微加好友**：
1. `/contact/search_contact` `{guid, username, from_scene:0, search_scene:1}`（后两值恒定）→ 取 `userName.string`/`antispamTicket`。
2. `/contact/add_friend`：

POST /contact/add_friend

```json
{"guid": "8f3a...", "username": "wxid_abc123", "ticket": "搜索返回的 antispamTicket", "scene": 15, "verify_content": "我是..."}
```

  scene=**常量 15**（手机号搜索添加）。`ticket==""` → 已是好友直接跳过；`ret==-200000011` → 账号频控次日重试。

**通过好友申请**：

POST /contact/verify_friend

```json
{"guid": "8f3a...", "username": "wxid_def456", "ticket": "回调XML -ticket", "scene": 2, "opcode": 3}
```

- 个微：opcode 常量 **1=CHECK 2=ADD 3=AGREE**，通过=3；username/ticket/scene 全部来自好友申请回调 XML 的 `-fromusername/-ticket/-scene`。
- 企微 PC/Fila：`/contact/sync_apply_contact` 游标拉申请列表（企微 `{guid, seq, limit:100}`，seq 首期传 "0"（平台侧持久化游标），循环至 `data.last_seq=="0"`）→ `/contact/agree_contact` 企微 `{guid, user_id, corp_id}`（两值直接取列表条目）/ Fila `{"guid","userId"}`。通过条件：`create_time==update_time`（未处理过）**且** 7 天窗口内。

### 6.2 联系人同步三式

| 协议 | 入口 | 形态 | 游标 |
|---|---|---|---|
| 企微 PC | `/contact/sync_contact` `{guid, seq, limit:100}` | 增量：条目 `type` 2057=新增 / ContactTypeDelByUser=删除 | 平台侧持久化游标，循环至 `last_seq=="0"` |
| 个微 | `/contact/init_contact` `{guid, contact_seq, room_seq}` | 增量：响应 `contactUsernameList` 逐批走 brief_info；`continueFlag=true` 用 `currentWxcontactSeq` 推进 | **断点续传**：任何一轮失败必须先落 seq 再退出，否则从 0 重拉 |
| Fila | `/contact/get_cached_contact_list` 仅 `{guid}` | 全量覆盖（读 App 本地缓存；原 `/contact/get_contact_list` 走微信服务器恒 error_code=4 已弃用） | 无 |

- 个微 brief_info：`/contact/batch_get_contact_brief_info` `{guid, username_list}` **批次上限 15**；协议端对"坏用户"整批静默返回空 body → 整批失败后降级单人重试。
- Fila 缓存接口字段映射：`remoteId`=user_id（uint64 字符串直读禁 Uint()）；`extras.realRemark`=备注 / `extras.remarks`=描述（**两字段反接是历史事故点**）；labelId/corpOrVid/groupId 均 >2^53 须字符串形态。

### 6.3 标签

- 企微 `/label/sync_label_list` `{guid, seq, sync_type}`——sync_type **1=企业标签 2=个人标签**；分页 `data.has_next`+`data.seq`。Fila 同路径仅 `{"guid"}` 一次全量，字段 camelCase，组标签（dataType=1）本身也是一条标签须一并落库。
- 企微打标签 `/label/contact_add_label` `{guid, label_id, label_groupid, business_type:1, user_list, corp_or_vid}`（corp_or_vid=机器人所属企业 id）。
- 个微建标签 `/label/add_contact_label` `{guid, label_name}` → 取 `labelPairList[0]` 的 labelId（空=失败）。
- 个微整体替换：

POST /label/modify_contact_label_list

```json
{"guid": "8f3a...", "username_list": ["wxid_abc123"], "label_id_list": [10001, 10002]}
```

  语义=**全量覆盖**：label_id_list = 现有标签 ∪ 目标新标签（库中无该名先 add 建标签拿 id）；替换后本地关联表重建。

### 6.4 资料修改

- 企微 PC `/contact/update_contact`：整个扩展信息 JSON 直传（注入 guid+user_id；nil 列表补 `[]`）。
- Fila 同路径**双形态**：仅收 `user_id` + `desc/remark/remark_url/company_remark`，**仅非空才下发**（空串=清空备注）；标签三态 `label_id_list`：nil=不下发 / 空数组=显式清空 / 非空=全量设置。
- 个微改备注 `/contact/modify_remark` `{guid, username, remark}`。

---

## 7. 实例生命周期

### 7.1 创建顺序（铁律）

**create → set_notify_url → 落库**，顺序不能换：先 create 拿 guid，立刻配回调，再写 DB。

| create_client 字段 | 填法 |
|---|---|
| client_type | 4000=Fila 手机版（**Fila 不调 create_client**，只建 DB 记录：state=uuid + proto_url，手机扫码登录后回调回填 guid）；其余=企微/个微 |
| channel | 常量 1（唯一例外：个微 SPH 模式传 2） |
| expired_at | 用户到期时间，秒级时间戳 |
| name | 可选，展示名 |

POST /client/set_notify_url

```json
{"guid": "8f3a...", "notify_url": "https://your.callback/notify"}
```

- 企微侧仅在未配 MQTT 通道时才走 HTTP 回调（配了 MQTT 则跳过）；个微无条件调。notify_url 平台默认拼法：企微与个微的默认回调路径片段不同。
- 企微实例禁 `/cloud/*`（add_image 除外）；Saas 档禁 `/session /arch /finder /private /open` 前缀——路径权限矩阵见 `auth.md`。

### 7.2 恢复 / 删除

- **restore 必带 force**（wire 名 `force_online`），不带则网关静默 no-op（两案实证）；restore 后跟一发 `/login/keep_login` 保活：

POST /client/restore_client

```json
{"guid": "8f3a...", "force_online": true}
```

- 删除链：`/client/stop_client`（`{guid}`，失败不阻断）→ `/client/remove_client`（`{guid}`）→ DB 软删。**Fila 删除不走这条链**：改调 Fila 网关 `/mobile/block_device` `{device_id: <guid>, state: <创建时的 uuid>}`（注意字段名是 device_id）——断 WS+移除绑定+拉黑旧授权码；Fila 网关会拒收 `/client/*` 全族。

### 7.3 代理与桥接

- `/client/set_proxy` `{guid, proxy, is_long_proxy}`——proxy 传空串=清除代理；连接报错 "Could not connect to proxy" 平台自动异步清。
- `/client/set_bridge` `{guid, bridge}`——bridge=WS 桥接地址；传空串=解绑。

---

## 8. 登录状态机（速查，全量见 login.md）

- 企微/个微：`/login/get_login_qrcode`（`{guid}`）→ 企微取 `data.qrcode`（base64 图）+`data.qrcode_content`；个微取 `qrcode.buffer`。
- `/login/check_login_qrcode`（`{guid}`）轮询：status **0=waiting 1=scanned 2=success 5=expired**，其他数值=confirming；协议错误 -12007 归一为 5。
- 个微 verify 分支**优先于数字状态**（此时 status 常为中间值 10）：`extNotifyData.type==11`→局域网验证；`authUrl` 含 `ticket=`→验证码；非空 authUrl→人脸。企微验证码走 `/login/verify_login_qrcode` `{guid, code}`；个微走 `/login/verify_login_url` `{guid, ticket, pin}`。
- Fila：不走二维码，走 `/mobile/auth_code` `{"state": <创建实例时的 uuid>}` 出授权码 → 手机输入 → `/mobile/devices`（body `{}`）按 `device_id==guid` 匹配 `online`；码过期重取即可，auth_code 不可重复用于判断（以设备在线为准）。

---

## 9. 消息同步 / 朋友圈 / 视频号 / 组织架构

### 9.1 消息同步（企微）

POST /sync/sync_msg

```json
{"guid": "8f3a...", "sync_key": "", "limit": 100}
```

- 首轮 `sync_key` 传空串；翻页回传响应 `data.max_sync_key`；终止 `data.continue_flag == 0`；消费 `data.msg_list[]` 跳过已读回执类型。
- `/sync/push_sync_msg` `{guid}`：主动触发推送；**Fila 跳过**（消息由手机端主动推）。

### 9.2 朋友圈发帖

- 企微/Fila `/sns/post_sns` `{guid, content, link_info, media_list, user_list, finder_info}`——**nil/空键必须 delete 掉**。
- **store_type 恒 0**（=CDN；SDK 硬编码，传 1 配 CDN fileid 必报 -1007）；Fila media_list 每项 `{fileid, type, store_type:0}`（type 与 BigCdnType 一致：1图/2视频/3缩略图），fileid **必须来自 `/cloud/cdn_big_upload`**（§2.2）；user_list 须转 `[]uint64`，空=全部可见。
- 个微 `/sns/sns_post` `{guid, object_desc, with_user_list, ...}`——nil 切片必须转空数组，不能缺键。
- 评论双协议字段名不同：企微 `{guid, comment, sid, refCommentId}` / 个微 `{guid, object_id, content, reply_comment_id}`。

### 9.3 视频号（个微，协议路由坑）

`/finder/finder_detail` `{guid, object_id, object_nonce_id, last_buffer}`——**仅 iPad 协议**；Mac 协议设备必须改走 `/pcfinder/get_detail_by_object`（无 last_buffer），否则 `ret=-2`。响应双命名兼容：iPad=camelCase / Mac=snake_case。

### 9.4 组织架构（arch）

POST /arch/sync_arch

```json
{"guid": "8f3a...", "cli_version": "", "limit": 100}
```

- 企微：cli_version 首轮传空串（平台侧持久化游标），循环内回传上一响应 `data.svr_version`；节点数据用 `/arch/get_arch_node_data` 把上一响应 `data.node_list` **原样 SetRaw 内嵌**（不二次构造）。
- Fila 差异：键名全 lowerCamelCase（`svrVersion/nodeList/patchList`）；bytes 字段输出 base64 须解码；**全量形态**响应 `data.fullData` 直接内含 patchList（跳过二次调用）；isDelete 节点跳过。
- 该接口在平台侧有死循环风险档案（for 无轮数上限，终止仅靠 node_list 为空）——对接方务必设轮数上限。

---

## 10. 跨接口坑速查表

| 坑 | 接口 | 一句话 |
|---|---|---|
| 字段名三态 | 全族 | 企微 `description`/个微 `desc`；企微 `user_id`/个微 `share_username`；企微 `md5,size`/个微 `file_md5,file_size` |
| Fila 媒体发送必嵌套 oneof | send_image/voice/video/file | 平铺会被网关拒（unknown field）；企微/个微平铺，Fila 嵌套——三协议不能共用一套 body |
| 凭证缺失仍发送 | Fila 发送族 | 缺 aes_key/md5 的 body 会打崩手机 App，平台侧在入口拦截 |
| `desc` 消息 ≠ `description` | send_link_card | 个微字段名就是 desc |
| at_list "all" 三态 | send_room_at | 企微/Fila→"0"；个微→"notify@all"；Fila content 还须自带 @昵称 |
| 上传响应顶层 vs data | c2c_upload 族 | 个微顶层平铺、企微 `data.*`、Fila camelCase——解析不能复用 |
| file_type 4/5 相反 | 上传/下载族 | 企微 4=视频 5=文件；Fila 反之（§0 大表） |
| 图片 1↔2 换档保底 | Fila c2c_download | 一档失败必试另一档，单档失败≠凭证过期 |
| URL 型凭证 | Fila 下载 | `http` 开头/`v1_` auth_key 走 wx_download，走 c2c 必败 error_code=4 |
| 朋友圈 fileid 来源 | post_sns/cdn_big_upload | 必须用 big_upload 的 FTN 凭证（`*1*` 前缀），c2c 的 hex fileid 必裂；store_type 恒 0 |
| >30MB 视频 | 企微发送族 | 改走 big_upload + upload_video_preview 双通道 |
| 群分页终止值 | get_room_list | 企微 `next_start==-1`；Fila 键在 `data.result.nextStart`（解析错=死循环） |
| batch_get_member_detail | Fila | camelCase `roomId` 且**不带 user_list**；protojson 严格模式未知字段报错 |
| roominfos 键双读 | batch_get_room_detail | 企微全小写/Fila camelCase，只读一边=Fila 静默 no-op |
| 拉人 40 阈值 | 个微群 | ≤40 add / >40 invite，字段同名语义不同 |
| brief_info 批次 15 | 个微联系人 | 超批静默失败；坏用户整批空回须降级单人重试 |
| label 三态 | Fila update_contact | nil=不动 / 空=清空 / 非空=全量设置；空串备注=清空备注 |
| labelId 超 JS 安全整数 | Fila 联系人 | >2^53 一律字符串形态直读，禁 Uint() |
| restore 不带 force | restore_client | 网关静默 no-op，必须 `force_online:true` |
| create 后顺序 | 实例族 | create → set_notify_url → 落库；Fila 不 create（state+auth_code 流） |
| Fila 拒收 /client/* | 实例族 | Fila 走 /mobile/*（block_device 字段名 device_id） |
| Mac 视频号路由 | finder_detail | Mac 设备调 /finder/* 必 ret=-2，改 /pcfinder/* |
| arch 无轮数上限 | sync_arch | 对接方自设上限；Fila 全量形态直接用 fullData |
| 引用/撤回原料在回调 | send_quote_msg/revoke_msg | appinfo/sender/content_type/seqid 全部来自收到消息时的回调字段，不是现查 |
| 语音 15 天 silk | send_voice | 旧素材后缀换 .silk 重新上传换凭证；Fila duration 毫秒/1000 |
| to_silk 永不传 | c2c_upload | 协议无此参数；直调传 true 必崩企微手机端 |
